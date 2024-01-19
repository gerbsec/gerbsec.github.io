---
layout: post
category: writeup
---

## Machine Info

- Name: Lame
- Description: Lame is a beginner level machine, requiring only one exploit to obtain root access. It was the first machine published on Hack The Box and was often the first machine for new users prior to its retirement. 
- Difficulty: Easy

## Initial Access && Privilege Escalation

nmap:

```bash
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.9
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  P           Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2024-01-18T12:49:44-05:00
|_clock-skew: mean: 2h30m12s, deviation: 3h32m10s, median: 10s
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
```

Looking for exploits on the different services I find that 2.3.4 ftp is vulnerable to remote code execution vulnerablity.

Attempting to exploit it led nowhere as it seemed like some sort of patch was applied:

![[assets/images/2024-01-18-lame-writerup-image-1.png]]
![[assets/images/2024-01-18-lame-writerup-image-2.png]]

Looking at vulnerablities for samba:
![[assets/images/2024-01-18-lame-writerup-image-3.png]]
I find this exploit on Metasploit.

Running it lends us root:
![[assets/images/2024-01-18-lame-writerup-image-4.png]]
## Beyond Root

In this section I like to look at why certain things might have not worked plus things I could do to harden these machines. This should be a simple update of samba however, I wanted to see why I wasn't able to exploit VSFTPD. 

Looking at netstat on the machine I see a ton of ports open, one of which was 6200:
![[assets/images/2024-01-18-lame-writerup-image-5.png]]

this means that comparing it to my nmap scan, the firewall is running and blocking almost everything. From there what we can do is attempt to connect locally to confirm this:
![[assets/images/2024-01-18-lame-writerup-image-6.png]]
violla, it was just firewall, its always firewall...


best,
gerbsec