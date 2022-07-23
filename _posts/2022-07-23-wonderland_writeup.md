---
layout: post
title: Wonderland Write-Up
comments: true
---
[Wonderland](https://tryhackme.com/room/wonderland)  is a medium difficulty TryHackMe box, doing this box for the first time was a lot of fun. I learned about exploiting imports.

As usual I start the machine by using nmap to view the open ports.

`nmap -sV 10.10.89.189`

First off there is an http server. When we navigate to the website there is a note: "Follow the White Rabbit". My first thought reading this note is to fuzz the site to see if there are any interesting files. I will be using gobuster.

`gobuster dir -u http://10.10.89.189/ -w /usr/share/wordlists/dirb/common.txt`

In the output, the only interesting thing that we found is a directory named "r". Navigating to that directory we have another message: "Keep Going". I will continue to fuzz in this directory with gobuster

`gobuster dir -u http://10.10.89.189/r -w /usr/share/wordlists/dirb/common.txt`

In the output we get "a", again going to that directory it continues to tell us to keep going. From here I figured this would eventually spell out rabbit. So I just went to http://10.10.89.189/r/a/b/b/i/t.

I tried to fuzz this directory but got nothing interesting from it. My next idea was to use inspect element.

In inspect element I saw there is a hidden paragraph tag, in this tag there are ssh credentials for Alice.

`ssh alice@10.10.89.189`

Doing a quick `ls` in Alice's home directory we have a root.txt and a walrus_and_the_carpenter.py. Seeing the root.txt in Alice's home directory I became very confused. I went to the try hack me page and the hint states: "Everything is upside down here" I got very curious looking at this tried to cat out a user.txt in the root directory.

`cat /root/user.txt`

It worked! flag 1 completed.

Next, I wanted to check what commands I could run with sudo.

`sudo -l`

From that output it showed I can run the walrus_and_the_carpenter.py file with rabbit. let's see what happens when that code is run.

`sudo -u rabbit python3.6 walrus_and_the_carpenter.py`

The code seems to just print lines from a story or something. I then opened the file in vim, we can see that the lines it prints are from a poem. Looking at the imports of the script we can see it imports random. Recently I learned that we can exploit imports by creating a file with the same name as the import (random.py) in our HOME directory and run code that we aren't supposed to.

On [GTFO Bins](https://gtfobins.github.io/) we can get code to spawn a shell with python. so lets add that to a file named random.py

`echo 'import os; os.system("/bin/sh")' > random.py`

now if we run the walrus_and_the_carpenter.py file as rabbit, we will get a shell as rabbit.

`sudo -u rabbit python3.6 walrus_and_the_carpenter.py`

if the script asks for alice's password, it is the same one used to ssh into the machine.

Now that we have a shell as Rabbit, go to Rabbit's home directory. In the directory there is a file named teaParty, to check what kind of file this is run file.

`file teaParty`

The output of file tells us that teaParty is an executable so let's try and run it.

`./teaParty`

The file gives a note and it shows a date. I would assume it is running the date command, so like we did with python we can make a file named date in our PATH and run code from it.

```
export PATH=/tmp:$PATH
echo $PATH
```

Now we write a script in /tmp named date.

use any text editor and put the following code into the file.

```
#!bin/bash
/bin/bash
```

finally run chmod to give the file permission to execute code

`chmod +x /tmp/date`

now if we run teaParty we get a shell for Hatter. In his home directory we have a password. ssh into his account with that password.

Doing some enumeration we find that perl as the ability to set our uid.

`getcap -r / 2>/dev/null`

To exploit this run

`perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'`

If we run whoami we can see we have root! cat the root.txt in Alice's home directory to get the flag!
