---
title: "Easy Peasy CTF"
date: 2025-08-08 12:00:00 +0300
categories: [Writeups, TryHackMe]
tags: [tryhackme, ctf, writeup, pentesting, security]
description: "Gobuster web fuzzing, steganographic image extraction, hidden hash cracking, and Apache cron job privilege escalation."
image:
  path: /assets/img/posts/easy_peasy_thm.webp
  alt: Easy Peasy CTF
---

# Enumaration

firstly with nmap,

```jsx
└─$ nmap -p- -sT 10.10.194.91 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-08 06:11 EDT
Nmap scan report for 10.10.194.91
Host is up (0.070s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT      STATE SERVICE
80/tcp    open  http
6498/tcp  open  unknown
65524/tcp open  unknown

```

version control:

```jsx
└─$ nmap -p6498,65524 -sV --version-light 10.10.194.91
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-08 06:14 EDT
Nmap scan report for 10.10.194.91
Host is up (0.072s latency).

PORT      STATE SERVICE VERSION
6498/tcp  open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
65524/tcp open  http    Apache httpd 2.4.43 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.93 seconds

```

```jsx
└─$ nmap -p80 -sV --version-light 10.10.194.91
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-08 06:16 EDT
Nmap scan report for 10.10.194.91
Host is up (0.072s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.16.1

```

from this, we need to do directory finding through gobuster with

```jsx
gobuster dir -u link -w wordlist.txt
```

finding hidden and hidden/whatever and with whatever we find first flag

```jsx
next port trying we found a site with apache and in that, robots.txt gives us second flag 
```

third flag is in the source code with apache site

there is a hidden directory, person is hashed it with base 62, so we visit and see another hash

looking at the hash, it seems it was gost hash

```jsx
└─$ steghide --extract -sf binarycodepixabay.jpg 
Enter passphrase: 

```

i am blown away with this, we need to extract the jpg..

with the gost hash

so we get the username, and password with binary

so with these we go in with ssh to 3. port

```jsx
user/txt = synt{a0jvgf33zfa0ez4y}

```

rot 13: flag{n0wits33msn0rm4l}

```jsx
boring@kral4-PC:/var/www$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* *    * * *   root    cd /var/www/ && sudo bash .mysecretcronjob.sh

```

i am not following whats happening…
