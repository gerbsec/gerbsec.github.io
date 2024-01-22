---
layout: post
category: writeup
---

## Machine Info

- Name: FriendZone
- Description: FriendZone is an easy difficulty Linux box which needs fair amount enumeration. By doing a zone transfer vhosts are discovered. There are open shares on samba which provides credentials for an admin panel. From there, an LFI is found which is leveraged to get RCE. A cron is found running which uses a writable module, making it vulnerable to hijacking.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 3.0.3
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:                                                                 
|   2048 a9:68:24:bc:97:1f:1e:54:a5:80:45:e7:4c:d9:aa:a0 (RSA)
|   256 e5:44:01:46:ee:7a:bb:7c:e9:1a:cb:14:99:9e:2b:8e (ECDSA)
|_  256 00:4e:1a:4f:33:e8:a0:de:86:a6:e4:2a:5f:84:61:2b (ED25519)
53/tcp  open  domain      ISC BIND 9.11.3-1ubuntu1.2 (Ubuntu Linux)
| dns-nsid:         
|_  bind.version: 9.11.3-1ubuntu1.2-Ubuntu
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-methods:      
|_  Supported Methods: OPTIONS HEAD GET POST
|_http-title: Friend Zone Escape software 
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
443/tcp open  ssl/http    Apache httpd 2.4.29
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=friendzone.red/organizationName=CODERED/stateOrProvinceName=CODERED/countryName=JO
| Issuer: commonName=friendzone.red/organizationName=CODERED/stateOrProvinceName=CODERED/countryName=JO
| Public Key type: rsa 
| Public Key bits: 2048              
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2018-10-05T21:02:30                                                                                                                        
| Not valid after:  2018-11-04T21:02:30                     
| MD5:   c144:1868:5e8b:468d:fc7d:888b:1123:781c                    
|_SHA-1: 88d2:e8ee:1c2c:dbd3:ea55:2e5e:cdd4:e94c:4c8b:9233
|_http-title: 404 Not Found
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET POST
| tls-alpn: 
|_  http/1.1
445/tcp open  EDtb
                         Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Hosts: FRIENDZONE, 127.0.1.1; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

### Port 21 

This is FTP I try logging in with anonymous login but it does not work.

### Port 445

This is Samba, I try to enumerate with smbclient and I can see a few shares. I keep this in mind as I also know we have a webpage on 80 and 443

```bash
HTB smbclient -L //10.10.10.123/
Password for [WORKGROUP\kali]:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        Files           Disk      FriendZone Samba Server Files /etc/Files
        general         Disk      FriendZone Samba Server Files
        Development     Disk      FriendZone Samba Server Files
        IPC$            IPC       IPC Service (FriendZone server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            FRIENDZONE
```

Here we can see Files, general, Development are available on the smb server. Attempting to access Files doesn't let me access it, but accessing Development and general works:

```bash
HTB smbclient  //10.10.10.123/general
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jan 16 15:10:51 2019
  ..                                  D        0  Tue Sep 13 10:56:24 2022
  creds.txt                           N       57  Tue Oct  9 19:52:42 2018

                3545824 blocks of size 1024. 1650272 blocks available
smb: \> get creds.txt
getting file \creds.txt of size 57 as creds.txt (0.3 KiloBytes/sec) (average 0.3 KiloBytes/sec)
smb: \> exit
➜  HTB smbclient  //10.10.10.123/Development
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jan 16 15:03:49 2019
  ..                                  D        0  Tue Sep 13 10:56:24 2022

                3545824 blocks of size 1024. 1650272 blocks available
smb: \> exit
```

Reading contents of creds.txt:
```bash
HTB cat creds.txt 
creds for the admin THING:

admin:WORKWORKHhallelujah@#
```

```bash
dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u  http://10.10.10.123

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/10.10.10.123/_24-01-22_14-34-47.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-22_14-34-47.log

Target: http://10.10.10.123/

[14:34:47] Starting: 
[14:34:48] 200 -  324B  - /index.html
[14:34:50] 403 -  293B  - /icons/                                          
[14:34:57] 200 -  747B  - /wordpress/                                      
[14:34:57] 301 -  316B  - /wordpress  ->  http://10.10.10.123/wordpress/
```

This leads me to a wordpress instance which is a deadend as its fake...
I look at the main page and it mentions this hostname: friendzoneportal.red, so I'll go ahead and dirsearch that after I add it to my `/etc/hosts` file.
### Port 53 DNS

I wanted to check if I can zone transfer to quickly find some subdomains on the domain:

```bash
$ dig axfr friendzoneportal.red @10.10.10.123

; <<>> DiG 9.19.17-1-Debian <<>> axfr friendzoneportal.red @10.10.10.123
;; global options: +cmd
friendzoneportal.red.   604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
friendzoneportal.red.   604800  IN      AAAA    ::1
friendzoneportal.red.   604800  IN      NS      localhost.
friendzoneportal.red.   604800  IN      A       127.0.0.1
admin.friendzoneportal.red. 604800 IN   A       127.0.0.1
files.friendzoneportal.red. 604800 IN   A       127.0.0.1
imports.friendzoneportal.red. 604800 IN A       127.0.0.1
vpn.friendzoneportal.red. 604800 IN     A       127.0.0.1
friendzoneportal.red.   604800  IN      SOA     localhost. root.localhost. 2 604800 86400 2419200 604800
;; Query time: 91 msec
;; SERVER: 10.10.10.123#53(10.10.10.123) (TCP)
;; WHEN: Mon Jan 22 14:43:22 EST 2024
;; XFR size: 9 records (messages 1, bytes 309)

```

Adding this output to my `/etc/hosts` file and I find a login page on the admin.friendzoneportal.red domain:

![](assets/images/2024-01-21-HTB-friendzone-writeup-image-1.png)

Putting in any credentials yeilds the same output of:

```
Admin page is not developed yet !!! check for another one
```

this is annoying, time to enumerate some more.

Looking at the certificate I can see that there is another domain:
![](assets/images/2024-01-21-HTB-friendzone-writeup-image-2.png)

friendzone.red

let me do the same process to this domain

![](assets/images/2024-01-21-HTB-friendzone-writeup-image-3.png)

More results, let's add em and enumerate:
![](assets/images/2024-01-21-HTB-friendzone-writeup-image-4.png)

I login to the application:

![](assets/images/2024-01-21-HTB-friendzone-writeup-image-5.png)

![](assets/images/2024-01-21-HTB-friendzone-writeup-image-6.png)

I can add an image name at the end of the image_id param:

![](assets/images/2024-01-21-HTB-friendzone-writeup-image-7.png)

I upload a rev in Development drive, and iirc the smb shares are all in /etc as stated when I listed them. 

Accessing that page gives me a reverse shell.
## Privilege Escalation

Checking mysql_data.conf file leads to some creds:

```bash
www-data@FriendZone:/var/www$ cat mysql_data.conf 
for development process this is the mysql creds for user friend

db_user=friend

db_pass=Agpyu12!0.213$

db_name=FZ
```

I can su into the user friend:
```bash
www-data@FriendZone:/home$ su friend
Password: 
friend@FriendZone:/home$
```

Upon some more enum I see this in pspy64:

```bash
2024/01/22 22:24:01 CMD: UID=0     PID=3183   | /usr/bin/python /opt/server_admin/reporter.py 
2024/01/22 22:24:01 CMD: UID=0     PID=3182   | /bin/sh -c /opt/server_admin/reporter.py 
2024/01/22 22:24:01 CMD: UID=0     PID=3181   | /usr/sbin/CRON -f
```

python script in opt runs every minute with python.

it imports os, so we can check if we can edit os, sure enough we can:

![](assets/images/2024-01-21-HTB-friendzone-writeup-image-8.png)

![](assets/images/2024-01-21-HTB-friendzone-writeup-image-9.png)

I added this line at the end that essentially adds the suid bit to /bin/bash

```bash
friend@FriendZone:/tmp$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1113504 Apr  4  2018 /bin/bash
friend@FriendZone:/tmp$ bash -p
bash-4.4# cat /root/root.txt 
179ebb93129e7dccc6dd29580992c57a
bash-4.4#
```

best,
gerbsec