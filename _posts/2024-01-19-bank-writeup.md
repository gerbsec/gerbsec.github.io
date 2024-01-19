---
layout: post
category: writeup
---

## Machine Info

- Name: Bank
- Description:  Bank is a relatively simple machine, however proper web enumeration is key to finding the necessary data for entry. There also exists an unintended entry method, which many users find before the correct data is located.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
ORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 08:ee:d0:30:d5:45:e4:59:db:4d:54:a8:dc:5c:ef:15 (DSA)
|   2048 b8:e0:15:48:2d:0d:f0:f1:73:33:b7:81:64:08:4a:91 (RSA)
|   256 a0:4c:94:d1:7b:6e:a8:fd:07:fe:11:eb:88:d5:16:65 (ECDSA)
|_  256 2d:79:44:30:c8:bb:5e:8f:07:cf:5b:72:ef:a1:6d:67 (ED25519)
53/tcp open  domain  ISC BIND 9.9.5-3ubuntu0.14 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.9.5-3ubuntu0.14-Ubuntu
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.7 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Enumerating DNS:
```bash
$ HTB dig axfr bank.htb @10.10.10.29   

; <<>> DiG 9.19.17-1-Debian <<>> axfr bank.htb @10.10.10.29
;; global options: +cmd
bank.htb.               604800  IN      SOA     bank.htb. chris.bank.htb. 2 604800 86400 2419200 604800
bank.htb.               604800  IN      NS      ns.bank.htb.
bank.htb.               604800  IN      A       10.10.10.29
ns.bank.htb.            604800  IN      A       10.10.10.29
www.bank.htb.           604800  IN      CNAME   bank.htb.
bank.htb.               604800  IN      SOA     bank.htb. chris.bank.htb. 2 604800 86400 2419200 604800
;; Query time: 44 msec
;; SERVER: 10.10.10.29#53(10.10.10.29) (TCP)
;; WHEN: Fri Jan 19 08:15:12 EST 2024
;; XFR size: 6 records (messages 1, bytes 171)

```

So we see that bank.htb is a valid domain, I'll go ahead and add it to my `/etc/hosts` file.

Visiting the webpage IP:
![[assets/images/Pasted image 20240119081909.png]]
Visiting the webpage via domain name:
![[assets/images/Pasted image 20240119081922.png]]

So from here I can try some basic SQLi but it gets me nowhere, so I'll start a dirsearch and I found balance-transfer:

![[assets/images/Pasted image 20240119082012.png]]

Looking into one of the files I can see that they are encrypted, so my thought process is to find a file that may be different. To do this quickly I'll wget everything into a directory and start looking for files that are smaller than the smallest one I can see right now which is about 581.

```bash
balance-transfer find . -size -581c -exec ls -l {} \;
-rw-r--r-- 1 kali kali 257 Jun 15  2017 ./68576f20e9732f1b2edc4df5b8533230.acc
balance-transfer cat ./68576f20e9732f1b2edc4df5b8533230.acc
--ERR ENCRYPT FAILED
+=================+
| HTB Bank Report |
+=================+

===UserAccount===
Full Name: Christos Christopoulos
Email: chris@bank.htb
Password: !##HTBB4nkP4ssw0rd!##
CreditCards: 5
Transactions: 39
Balance: 8842803 .
===UserAccount===
```

This gives us some creds, I can try to SSH in:
```bash
balance-transfer ssh chris@bank.htb
The authenticity of host 'bank.htb (10.10.10.29)' can't be established.
ED25519 key fingerprint is SHA256:7S4JgORJLloHIy/gCCkxvRpbrpWXAlMs8QK2jFtpn/w.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'bank.htb' (ED25519) to the list of known hosts.
chris@bank.htb's password: 
Permission denied, please try again.
```

No luck!

So I'll login to the application and I find a support page, looks like a file upload, lets see if its a vulnerability:

![[assets/images/Pasted image 20240119083033.png]]

![[assets/images/Pasted image 20240119083044.png]]

I attempt different combinations of file bypasses but nothing works. Next I try to look at the source code of the page:

![[assets/images/Pasted image 20240119083136.png]]

So I'll upload with htb extension, and from my dirsearch earlier I know that we have uploads directory:

![[assets/images/Pasted image 20240119083239.png]]

![[assets/images/Pasted image 20240119083245.png]]
## Privilege Escalation

From here I'll do my basic enumeration and I find this weird emergency suid binary:
![[assets/images/Pasted image 20240119083330.png]]

Running it gives me root...

![[assets/images/Pasted image 20240119083352.png]]
## Beyond Root

There are a few things to fix, I need to remove the unencrypted creds. The upload page was secure, and coulda remained secure but obscurity if there wasn't a hint in the source code. What we can do is patch the htb extention and remove the hint from source. Next we should get rid of this emergency binary...

### Webpage

Checking php5.conf I can see that it has a files match on htb extension to execute php:

![[assets/images/Pasted image 20240119083758.png]]

I just have to go ahead and remove those 3 lines.

Next I'll delete the emergency binary:
```bash
bash-4.3# rm /var/htb/bin/emergency
```

best,
gerbsec