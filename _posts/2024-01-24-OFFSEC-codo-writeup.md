---
layout: post
category: writeup
---

## Machine Info

- Name: Codo
- Description: Part 2 of Mid Year CTF machines
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 62:36:1a:5c:d3:e3:7b:e1:70:f8:a3:b3:1c:4c:24:38 (RSA)
|   256 ee:25:fc:23:66:05:c0:c1:ec:47:c6:bb:00:c7:4f:53 (ECDSA)
|_  256 83:5c:51:ac:32:e5:3a:21:7c:f6:c2:cd:93:68:58:d8 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: All topics | CODOLOGIC
```

Codologic instance, upon viewing the page I attempt to login, I use the creds `admin:admin` and sure enough that works!

I go to the admin page and upload a php file in the forum logo:
![](assets/images/2024-01-24-OFFSEC-codo-writeup-image-1.png)

then I access it in the attatchments and get a shell!:
![](assets/images/2024-01-24-OFFSEC-codo-writeup-image-2.png)

## Privilege Escalation

For root it was quite simple, I find creds in the config file and use them to su into root:

![](assets/images/2024-01-24-OFFSEC-codo-writeup-image-3.png)

best,
gerbsec