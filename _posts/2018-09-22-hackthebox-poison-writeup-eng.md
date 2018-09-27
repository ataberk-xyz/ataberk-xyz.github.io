---
layout: post
title: HackTheBox Poison Writeup [eng]
author: 0xSaiyajin
tags: [writeup,ctf]
categories: [writeup]
---

Greetings! 


In this blog post, you will understand how to take privileges step-by-step on Poison machine of HackTheBox Pentesting Labs.
I choose this machine because of it's retired since 1 week.

If you are new in the HackTheBox platform, the first lesson you need to learn is **enumeration.**
Also, you need to know basics of pentesting methodology.

Anyway, you already know that. Let's start.

```
Target: 10.10.10.84 [Poison]
System: FreeBSD
```

Firstly, we need to check which ports are open and which services are using these ports.

```nmap -sC -sV -T5 10.10.10.84```

![nmap output]({{site.url}}/assets/images/poison/poison-nmap.png)

When we try to access to port 80 via browser, a php file is encountering us.

![http]({{site.url}}/assets/images/poison/http.png)

If you try to submit any file names on above to input box, you will see that it's opening selected file. 
Also, if you try wrong folder name as an input it will raise an error.

![lfi]({{site.url}}/assets/images/poison/http-error.png)

As you can see, **include()** function used here. It's gonna occur *Local File Inclusion* vulnerability.
You can read "/etc/passwd" for user enumeration.

![passwd]({{site.url}}/assets/images/poison/etcpasswd.png)

So, we got usernames. (charix; insteresting one. Let's keep it in our mind.)
You can check another files but i'll check **listfiles.php**

![listfiles]({{site.url}}/assets/images/poison/listfiles.png)

I found that pwdbackup.txt pretty attractive.

![pwdbackup]({{site.url}}/assets/images/poison/pwdbackup.png)

It says you need to decode it 13 times.(base64)
*easy,peasy /w* **python**

```
import base64
encoded = "Vm0wd2QyUXlVWGxWV0d4WFlURndVRlpzWkZOalJsWjBUVlpPV0ZKc2JETlhhMk0xVmpKS1IySkVU bGhoTVVwVVZtcEdZV015U2tWVQpiR2hvVFZWd1ZWWnRjRWRUTWxKSVZtdGtXQXBpUm5CUFdWZDBS bVZHV25SalJYUlVUVlUxU1ZadGRGZFZaM0JwVmxad1dWWnRNVFJqCk1EQjRXa1prWVZKR1NsVlVW M040VGtaa2NtRkdaR2hWV0VKVVdXeGFTMVZHWkZoTlZGSlRDazFFUWpSV01qVlRZVEZLYzJOSVRs WmkKV0doNlZHeGFZVk5IVWtsVWJXaFdWMFZLVlZkWGVHRlRNbEY0VjI1U2ExSXdXbUZEYkZwelYy eG9XR0V4Y0hKWFZscExVakZPZEZKcwpaR2dLWVRCWk1GWkhkR0ZaVms1R1RsWmtZVkl5YUZkV01G WkxWbFprV0dWSFJsUk5WbkJZVmpKMGExWnRSWHBWYmtKRVlYcEdlVmxyClVsTldNREZ4Vm10NFYw MXVUak5hVm1SSFVqRldjd3BqUjJ0TFZXMDFRMkl4WkhOYVJGSlhUV3hLUjFSc1dtdFpWa2w1WVVa T1YwMUcKV2t4V2JGcHJWMGRXU0dSSGJFNWlSWEEyVmpKMFlXRXhXblJTV0hCV1ltczFSVmxzVm5k WFJsbDVDbVJIT1ZkTlJFWjRWbTEwTkZkRwpXbk5qUlhoV1lXdGFVRmw2UmxkamQzQlhZa2RPVEZk WGRHOVJiVlp6VjI1U2FsSlhVbGRVVmxwelRrWlplVTVWT1ZwV2EydzFXVlZhCmExWXdNVWNLVjJ0 NFYySkdjR2hhUlZWNFZsWkdkR1JGTldoTmJtTjNWbXBLTUdJeFVYaGlSbVJWWVRKb1YxbHJWVEZT Vm14elZteHcKVG1KR2NEQkRiVlpJVDFaa2FWWllRa3BYVmxadlpERlpkd3BOV0VaVFlrZG9hRlZz WkZOWFJsWnhVbXM1YW1RelFtaFZiVEZQVkVaawpXR1ZHV210TmJFWTBWakowVjFVeVNraFZiRnBW VmpOU00xcFhlRmRYUjFaSFdrWldhVkpZUW1GV2EyUXdDazVHU2tkalJGbExWRlZTCmMxSkdjRFpO Ukd4RVdub3dPVU5uUFQwSwo="
for i in range(13):
	encoded = encoded.decode('base64')
print encoded
```

Could it be credential? Why not? > Charix!2#4%6&8(0
```
Username : charix
Password : Charix!2#4%6&8(0
```

Remember the port 22 !

```ssh charix@10.10.10.84```

We're in.

![gotuser]({{site.url}}/assets/images/poison/usertxt.png)

Got user.txt and from now on we are trying to achieve root.txt. So, the enumeration step that i said starting know.
You can download **LinEnum.sh** for enumeration. I'll start with the basics. 
These basic questions are:

+ which technologies are running on this machine?
+ are there any suid bit binaries for unprivileged user can use?
+ any editable system/user files?
+ which processes are running right now?
+ which processes are running as a root user?

I looked for all answers for this questions. So, i will focus to answer of the last question.

```ps aux``` optionally: ```ps aux | grep 'root'```

![processes]({{site.url}}/assets/images/poison/psaux.png)

Xvnc is working as root privileges. We saw that on nmap output. Xvnc is server for sharing screen display of host machine to another user on same host.
So, if we figure out any way for accessing this application, we can get root privileges.

When we try to connect to Xvnc server from port 5902, it says "Connection was refused by the Computer".

If you check full parameters ps aux output for xvnc, you will also see that this application is only accessable from localhost. (-localhost , -rfport 5901)
That's not problem. You know, we can access to ssh. If we can tunnel 5901 port to our local machine, we can access to Xvnc server.

```ssh -l charix -L 5903:localhost:5901 10.10.10.84```

Connection established. Port forwarded.

![vncviewer]({{site.url}}/assets/images/poison/vncviewer.png)

We still need password for authenticate to vnc server.
Let's rewind the time for a little.
There was a secret.zip in home directory. 

```scp charix@10.10.10.84:secret.zip /home/saiyajin/Desktop```

Password protected file; password: Charix!2#4%6&8(0

```vncviewer -passwd /path-to-secret localhost:3```

![gotuser]({{site.url}}/assets/images/poison/roottxt.png)

```
Got r00t!
```
