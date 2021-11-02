# Writer

## Remote Discovery

> Note: the box IP has been added to the `/etc/hosts` file as `writer.htb`

### Nmap Scan

An `nmap -sC -sV writer.htb` yields:
```
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
...
80/tcp  open  http        Apache httpd 2.4.41 ((Ubuntu))
...
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
```
