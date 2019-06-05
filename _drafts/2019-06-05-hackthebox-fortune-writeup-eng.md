---
layout: post
title: HackTheBox Fortune Writeup [eng]
author: 0xSaiyajin
tags: [writeup,ctf,certificates]
categories: [writeup]
---

Greetings!

With solving **Fortune** machine, I finished half of the number of machines on HackTheBox. At present, **Fortune** has not retired yet. But I decided to write it's writeup. I will share this blog post when the machine is retired. So, if you are reading this blog post right now, it means you are looking into the past.

Everytime I'm making same mistake. I think the best way of writing a writeup is keeping some notes about machine that you're trying to solve. Every time, I forget this. Therefore, I'm solving machine again while I was preparing new blog post. 

So we're starting..

```
Target: 10.10.10.127 [Fortune]
System: Other [OpenBSD]
Difficulty: [6.2/10]
```

What I learned from this machine:
+ Enumeration
+ More Enumeration...
+ Basis of SSL Certs
+ Steps of mounting NFS shares.
+ Abusing the authorized\_keys file.
+ Analysis of PostgreSQL dump files.

At first, we are starting with port enumeration. 

```nmap 10.10.10.127 -sC -sV -p-```


```
Starting Nmap 7.70 ( https://nmap.org ) at 2019-06-05 05:12 +03
Nmap scan report for 10.10.10.127
Host is up (0.084s latency).
Not shown: 65532 closed ports
PORT    STATE SERVICE    VERSION
22/tcp  open  ssh        OpenSSH 7.9 (protocol 2.0)
| ssh-hostkey: 
|   2048 07:ca:21:f4:e0:d2:c6:9e:a8:f7:61:df:d7:ef:b1:f4 (RSA)
|   256 30:4b:25:47:17:84:af:60:e2:80:20:9d:fd:86:88:46 (ECDSA)
|_  256 93:56:4a:ee:87:9d:f6:5b:f9:d9:25:a6:d8:e0:08:7e (ED25519)
80/tcp  open  http       OpenBSD httpd
|_http-server-header: OpenBSD httpd
|_http-title: Fortune
443/tcp open  ssl/https?
|_ssl-date: TLS randomness does not represent time

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 263.14 seconds
```

Okay, three ports are open:
+ 22
+ 80
+ 443

Before analyzing these ports, the machine is looking like it has web application on it.

We are continuing with connecting HTTP port via browser. When we connect to it, a simple page with some options are greetings us.

![fortune]({{site.url}}/assets/images/fortune/fortune_http.png)

If we select one of these options and submit, we can see that application is selecting random fortune for us.

For example:

![first fortune]({{site.url}}/assets/images/fortune/first_fortune.png)

Cool one, right?

When we analyze what is running on the background with burp tool, we are detecting **"/select"** endpoint. It is using **POST** method and **"db"** parameter.

![fortune select]({{site.url}}/assets/images/fortune/fortune_select.png)

At first look, **db** parameter confused my mind. I thought there could be SQL Injection in here or something like this. For this reason, I wasted my 1 hour on it. After that, I searched about these database names on Google. It led me to a github page about "Fortune Cookie Databases". In README page, I saw OpenBSD tool named with **fortune**. Finally, name of the machine was looking meaningful to me. I installed exact tool and started to working on it.

We are running that tool with argument as db name which I saw on web application.

```
fortune zippy
fortune startrek
```

So what it means if same system is running on the web application?
**Remote Code Execution!**

![fortune tool]({{site.url}}/assets/images/fortune/fortune_tool.png)

All we have to do was appending a goddamn **PIPE** to our db parameter all this time. I was overthinking about SQL Injection. 

Next step, we are enumerating the system with privileges of **\_fortune** user.

```uid=512(_fortune) gid=512(_fortune) groups=512(_fortune)```

While I was enumerating the system, I saw another application. It is **sshauth**.

```db=zippy| ls -la ../sshauth```

```
total 68
drwxr-xr-x  4 _sshauth  _sshauth    512 Feb  3 05:08 .
drwxr-xr-x  5 root      wheel       512 Nov  2  2018 ..
-r--------  1 _sshauth  _sshauth     61 Nov  2  2018 .pgpass
drwxrwxrwx  2 _sshauth  _sshauth    512 Nov  2  2018 __pycache__
-rw-r--r--  1 _sshauth  _sshauth    341 Nov  2  2018 sshauthd.ini
-rw-r-----  1 _sshauth  _sshauth  15006 Jun  4 17:55 sshauthd.log
-rw-rw-rw-  1 _sshauth  _sshauth      6 Jun  4 16:01 sshauthd.pid
-rw-r--r--  1 _sshauth  _sshauth   1799 Nov  2  2018 sshauthd.py
drwxr-xr-x  2 _sshauth  _sshauth    512 Nov  2  2018 templates
-rw-r--r--  1 _sshauth  _sshauth     67 Nov  2  2018 wsgi.py
```

If we inspect **sshauthd.py** file, we can see that there is another endpoint: **"/generate"**.

```db=zippy| cat ../sshauth/sshauthd.py```

![sshauthd.py]({{site.url}}/assets/images/fortune/sshauthdpy.png)

Also, I checked "sshauth" directory from website. As I expected, I saw another application that we inspected and it is using **/generate** endpoint.

![sshauthapp.py]({{site.url}}/assets/images/fortune/sshauthapp.png)

But, the problem with this application is we can't use **/generate** endpoint from this port. It redirects to **404 - Page Not Found** every time. I will keep this information in my mind.

We are continuing to enumeration process.
