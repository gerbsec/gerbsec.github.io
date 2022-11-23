---
layout: post
category: writeup
---


![](assets/images/sunshinectf22.png)


- sunshine CTF was fun! we were in person and unfortunately had no internet what so ever. 
- that caused me to not put as much effort as i normally would have.
- nevertheless i got some challeneges done and here are their writeups!

### network pong

this challenge was interesting. since i had just refreshed my knowledge on the topic it felt easy. it's a classic command injection vulnerability except
they are using command expansion. 

`
{ping,-c,1,[INJECT ME]}
`

i figured that out by entering a semicolon


![](assets/images/expansion.png)

then i entered a whoami command

`0.0.0.0$(whoami)`

![](assets/images/whoami.png)

that was successful, therefore I ran a read file

`0.0.0.0$(cat /etc/passwd)`

![](assets/images/space.png)

this was bad as it could have been either of two things:
- length based check
- it doesn't like spaces

i wanted to be optimistic about this so i went ahead and assume it's a space issue:

`0.0.0.0$(cat${IFS}/etc/passwd)`

![](assets/images/cat.png)

this looks better, the previous error is gone and it is complaining about cat, let me try less

`0.0.0.0$(less${IFS}/etc/passwd)`

no error, so now lets try to read the flag

`0.0.0.0$(less${IFS}flag.txt)`

![](assets/images/flag.png)


success!


### plumber game

this was a reversing challenge that was pretty easy generally.

it starts off with a upx packed binary that needed to be unpacked:

```
gerbsec@illusion:~$ strings plumber_game | grep upx
$Info: This file is packed with the UPX executable packer http://upx.sf.net 
```

so i unpacked it with upx

```
gerbsec@illusion:~/chall/upx-4.0.1-amd64_linux$ ./upx -d ../plumber_game 
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2022
UPX 4.0.1       Markus Oberhumer, Laszlo Molnar & John Reiser   Nov 16th 2022

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
   2144897 <-    689396   32.14%   linux/amd64   plumber_game

Unpacked 1 file.
```

looking at strings, it looks like go, so i sent it into gdb

i run `disass main.main` and i see an interesting `runtime.memequal` soon after a `ScanIn` i suspect this is the string being compared to with the password.

```
   0x000000000048edb1 <+257>:   mov    QWORD PTR [rsp+0x10],rax
   0x000000000048edb6 <+262>:   mov    QWORD PTR [rsp],rcx
   0x000000000048edba <+266>:   lea    rax,[rip+0x33050]        # 0x4c1e11
   0x000000000048edc1 <+273>:   mov    QWORD PTR [rsp+0x8],rax
   0x000000000048edc6 <+278>:   call   0x44e150 <runtime.memequal>
   0x000000000048edcb <+283>:   movzx  eax,BYTE PTR [rsp+0x18]
   0x000000000048edd0 <+288>:   test   al,al
   ```


i add a bp at `*main.main+278`, i run and enter 16 chrs for the password since in a previous check it says it needs 16.

looking at the output I see what could possibly be a string:

```
0xc420047d40 —▸ 0x4c1e11 (string.*+6241) ◂— 0x36325f3172347440 ('@t4r1_26')
```

i inspect that address

```
pwndbg> x/s 0x4c1e11
0x4c1e11:       "@t4r1_2600_l0v3rGC worker (idle)Imperial_AramaicMSpanList_InsertMSpanList_RemoveMeroitic_CursiveOther_AlphabeticSIGNONE: no trapZanabazar_Square\nruntime stack:\nbad frame layoutbad special kindbad symb"...
```

i see the password `@t4r1_2600_l0v3r` and use it when running the binary:

```
gerbsec@illusion:~/chall$ ./plumber_game 
Please enter your password below:
@t4r1_2600_l0v3r
Password accepted! Dispensing flag...
sun{go_to_the_other_castle}@t4r1_2600_l0v3r
```

win!


beyond these two challenges, the rest of the ones i solved felt very lame and not writeup worthy, i also now know that i need to practice my pwn skills and get better at math! thanks for hosting my friends @ucf!


best, gerbsec
