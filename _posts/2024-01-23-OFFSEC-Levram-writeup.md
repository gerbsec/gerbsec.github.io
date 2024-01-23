---
layout: post
category: writeup
---

## Machine Info

- Name: Levram
- Description:  Part 3 of Mid Year CTF machines
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 b9:bc:8f:01:3f:85:5d:f9:5c:d9:fb:b6:15:a0:1e:74 (ECDSA)
|_  256 53:d9:7f:3d:22:8a:fd:57:98:fe:6b:1a:4c:ac:79:67 (ED25519)
8000/tcp open  http-alt WSGIServer/0.2 CPython/3.10.6
| http-methods: 
|_  Supported Methods: GET OPTIONS
|_http-server-header: WSGIServer/0.2 CPython/3.10.6
|_http-cors: GET POST PUT DELETE OPTIONS PATCH
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 404 Not Found
|     Date: Tue, 23 Jan 2024 13:47:41 GMT
|     Server: WSGIServer/0.2 CPython/3.10.6
|     Content-Type: text/html
|     Content-Length: 9979
|     Vary: Origin
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta http-equiv="content-type" content="text/html; charset=utf-8">
|     <title>Page not found at /nice ports,/Trinity.txt.bak</title>
|     <meta name="robots" content="NONE,NOARCHIVE">
|     <style type="text/css">
|     html * { padding:0; margin:0; }
|     body * { padding:10px 20px; }
|     body * * { padding:0; }
|     body { font:small sans-serif; background:#eee; color:#000; }
|     body>div { border-bottom:1px solid #ddd; }
|     font-weight:normal; margin-bottom:.4em; }
|     span { font-size:60%; color:#666; font-weight:normal; }
|     table { border:none; border-collapse: collapse; width:100%; }
```

Two ports, nice and easy!

Main page:

![](assets/images/2024-01-23-OFFSEC-Levram-writeup-image-1.png)

Default creds? `admin:admin`

![](assets/images/2024-01-23-OFFSEC-Levram-writeup-image-2.png)

nice! looks like there is an rce on version 0.9.7!

in order for it work we needed to create a project, just visiting the project page and clicking create with a random name should be enough:
![](assets/images/2024-01-23-OFFSEC-Levram-writeup-image-3.png)

Then we get our shell!
## Privilege Escalation

I was bored and ran linpeas right away:

![](assets/images/2024-01-23-OFFSEC-Levram-writeup-image-4.png)

so that was easy capabilities, work somewhat like suid, well they literally are since they ahve the suid cap set.

![](assets/images/2024-01-23-OFFSEC-Levram-writeup-image-5.png)

root!

best,
gerbsec