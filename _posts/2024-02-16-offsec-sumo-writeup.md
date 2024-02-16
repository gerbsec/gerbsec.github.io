---
layout: post
category: writeup
---

## Machine Info

- Name: Shakabrah
- Description: Surf's up!
- Difficulty: Easy

## Initial Access

Nmap:
```bash
# Nmap 7.94SVN scan initiated Fri Feb 16 14:56:22 2024 as: nmap -sC -sV -v -p- -o nmap --min-rate 1000 192.168.204.86
Nmap scan report for 192.168.204.86
Host is up (0.046s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 33:b9:6d:35:0b:c5:c4:5a:86:e0:26:10:95:48:77:82 (RSA)
|   256 a8:0f:a7:73:83:02:c1:97:8c:25:ba:fe:a5:11:5f:74 (ECDSA)
|_  256 fc:e9:9f:fe:f9:e0:4d:2d:76:ee:ca:da:af:c3:39:9e (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Feb 16 14:58:34 2024 -- 1 IP address (1 host up) scanned in 132.27 seconds
```

Visiting the webpage:

![](assets/images/2024-02-16-offsec-sumo-writeup-image-1.png)

I instantly lean towards command injection:
![](assets/images/2024-02-16-offsec-sumo-writeup-image-2.png)

From here I attempt to send myself a shell over 443 (my usual revshell port) but that doesn't work because of firewall, so instead I send myself a shell over 80:

![](assets/images/2024-02-16-offsec-sumo-writeup-image-3.png)


![](assets/images/2024-02-16-offsec-sumo-writeup-image-4.png)

We got a shell!
## Privilege Escalation

I check suid:

![](assets/images/2024-02-16-offsec-sumo-writeup-image-5.png)

I see vim:

![](assets/images/2024-02-16-offsec-sumo-writeup-image-6.png)

![](assets/images/2024-02-16-offsec-sumo-writeup-image-7.png)

best,
gerbsec