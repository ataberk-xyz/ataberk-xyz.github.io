---
layout: post
title: TSG CTF - Obliterated File 1-2 Writeup [eng]
author: 0xSaiyajin
tags: [writeup,ctf,git,forensics]
categories: [writeup]
---

Hello again. We participated to TSG CTF two days ago. The challenges were really hard. Estimated difficulty of TSG CTF was **Intermediate - Hard** on ctftime[dot]org. 

I had solve only two forensics challenge. Both of them were related with **git** protocol. 

## Obliterated File (First Challenge)

Description of the first challenge:
>This problem has unintended solution, fixed as "Obliterated File Again". Original problem statement is below. Working on making a problem of TSG CTF, I noticed that I have staged and committed the flag file by mistake before I knew it. I googled and found the following commands, so I'm not sure but anyway typed them. It should be ok, right?

![obliteratedfile1]({{site.url}}/assets/images/obliterated1/obliteratedfile1.png)


```
$ git filter-branch --index-filter "git rm -f --ignore-unmatch problem/flag" --prune-empty -- --all
$ git reflog expire --expire=now --all
$ git gc --aggressive --prune=now
```

So, it shows that our developer were trying to remove **problem/flag** file which he pushed it to his project accidentally. 

![unzipfile]({{site.url}}/assets/images/obliterated1/unzipfile.png)

At first, I didn't realize what developer did. All I did was to check ```git log``` output. I compared last commit with other commits in order.
Then, I found important detail on one commit luckily. Flag file was there. I created new branch from that commit.

```git checkout 8412ed -b flag```

![gitlog]({{site.url}}/assets/images/obliterated1/gitlog.png)

I've used ```file flag``` command to understand what is this file. It was zlib compressed file. I got help from Python.

![zlibdcmp]({{site.url}}/assets/images/obliterated1/zlibdcmp.png)

```
python -c "import zlib;f=open('flag','rb').read();print(zlib.decompress(f))"
```

Got first flag!

```TSGCTF{$_git_update-ref_-d_refs/original/refs/heads/master}```

## Obliterated File Again (Second Challenge)

Description of the second challenge:
>I realized that the previous command had a mistake. It should be right this time...?

![obliteratedfile2]({{site.url}}/assets/images/obliterated2/obliterated2file.png)

```$ git filter-branch --index-filter "git rm -f --ignore-unmatch *flag" --prune-empty -- --all```

If we check what our developer done, we can say that he finally removed flag file from commits.

> Did he?

I tried same method for this file too. But it didn't work. It means, we have no luck with ```git log```.
I had to try different method. After making some research, I found another useful git command.

```git rev-list --objects --all```

This command lists all of the objects on the project in reverse chronological order. Thankfully, it worked like a charm. After running it, I was able to see flag file.

![rev-list]({{site.url}}/assets/images/obliterated2/revlist.png)

```c1e375244c834c08d537d564e2763a7b92d5f9a8 problem/flag```

Finally, all I need was fetching this object.

I've used ```git show``` command for it.

```git show c1e3752 > flag```

After that, I've checked file type and it was zlib too. Executing same Python script led me to take the flag.

Got second flag!

```TSGCTF{$_git_update-ref_-d_refs/original/refs/heads/master_S0rry_f0r_m4king_4_m1st4k3_0n_th1s_pr0bl3m}```

Cya!
