---
layout: post
title: UTSA Cyber CTF - Linux2 Writeup [eng]
author: 0xSaiyajin
tags: [writeup,ctf]
categories: [writeup]
---

Ah, we meet again. I was working as a Project Manager in a company. I left the job 2 days ago. In this time, I decided to take OSCP certification.
So, I have to work hard and try hard for cert. When I was working, I was also interested with Cyber Security.

Before I left the job, we**(isDebuggerPresent)** joined UTSA Cyber CTF and took 19th place. Our rank is not that bad because we participated in that competition on the first 3 days.
I am writing a writeup for this challenge because I had a lot of fun solving the question. This question tested my penetration testing skills.

So, let's start..

In the description of this challenge, login credentials are given.

```
ssh user@35.231.176.102
password: utsacyber
```

After connecting to SSH service, CSACTF banner welcomes us. The first thing that comes to my mind is checking which processes are running on the host.

![pslist]({{site.url}}/assets/images/utsalinux2/pslist.png)

Six processes is running. We could be in container structure. We can analyze it with few commands. 
I will pick this one.
```cat /proc/<pid>/cgroup```

![docker]({{site.url}}/assets/images/utsalinux2/docker.png)

We can see that we are in Docker container. I forgot to tell that there was a readme.txt in home directory.
It says: "get root". What a great hint for "getting root" access. Our purpose is reading flag. We have to continue with this idea. 
After that, I checked which binaries are using root privilege.
```find / -perm -u=s -type f 2>/dev/null```

![checksudo]({{site.url}}/assets/images/utsalinux2/checksudo.png)


In the output, we can see that **"nmap"** is working with sudo privilege. It's interesting, right? Let's quickly validate it.

![nmapps]({{site.url}}/assets/images/utsalinux2/nmapps.png)

After making some searches, I found that we can use nmap in interactive mode.
It interacts with bash. So, it means we can get root easily. Let's check it.

![interactive]({{site.url}}/assets/images/utsalinux2/interactive.png)


Bad luck. Current version is not supporting "interactive" mode. Then, I realized something. If we can run any script which we wrote then we could be able to get root access.
All we have to do is understanding how nmap scripts are working. 

Nmap scripts are using ".nse" extension. They are using "Lua" language. We are living in "Computer" era. So, we can access every knowledge with quick search. We need to create three sections for making our script work.
These are: 

+ Head
+ Rule
+ Action

In the **Head** section, we can identify some fields like "author", "description" etc.

**Rule** section is necessary for identifying some conditions for port status. For example, if port is using "tcp" as protocol then do <action>.

**Action** section is where we inject our command. If condition that we wrote on "Rule" section is met, then it will run the function that we write in this section.

![examplense]({{site.url}}/assets/images/utsalinux2/examplense.png)

Also, we are using **"io"** library for reading some data from file. I picked "/etc/shadow" file for testing that are we able to read root privileged file. If our assumptions are correct, we could be able to do everything on the host.
Okay, we are going.

```nmap --script flag.nse --open 127.0.0.1```

![shadow]({{site.url}}/assets/images/utsalinux2/shadow.png)

Cool. So, it says we have to get root for achieving to the flag. I made one more assumption. 

![flagnse]({{site.url}}/assets/images/utsalinux2/flagnse.png)

![flagoutput]({{site.url}}/assets/images/utsalinux2/flagoutput.png)

Got flag! I was the 6th solver of this challenge. We got **498 points** from this question.

