---
title: "Startup THM"
date: 2025-11-09 12:00:00 +0300
categories: [Writeups, TryHackMe]
tags: [tryhackme, ctf, writeup, pentesting, security]
description: "Anonymous FTP upload to web shell, Wireshark PCAP traffic analysis, and cron job bash script privilege escalation."
image:
  path: /assets/img/posts/startup_thm.png
  alt: Startup THM
---

# Enumaration

## nmap

```
└─$ nmap -sV -sC --min-rate 10000 startup.thm  
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-09 10:00 EST
Nmap scan report for startup.thm (10.10.226.27)
Host is up (0.072s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.23.176.115
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp [NSE: writeable]
| -rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
|_-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b9:a6:0b:84:1d:22:01:a4:01:30:48:43:61:2b:ab:94 (RSA)
|   256 ec:13:25:8c:18:20:36:e6:ce:91:0e:16:26:eb:a2:be (ECDSA)
|_  256 a2:ff:2a:72:81:aa:a2:9f:55:a4:dc:92:23:e6:b4:3f (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Maintenance
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.91 seconds

```

3 ports open

before heading into http, i want to check anonymous ftp

we can get 2 things

```
└─$ cat notice.txt         
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY. 
People downloading documents from our website will think we are a joke!
 Now I dont know who it is, but Maya is looking pretty sus.
                                                                                                                    
```

and photo. in the photo there seems to be a sort of a passphrase or something. we will look into that later

## web crawling

### gobuster

we see that /files is available, and when we go there we see ftp files.

```bash
└─$ gobuster dir -u startup.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt 
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://startup.thm
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/files                (Status: 301) [Size: 310] [--> http://startup.thm/files/]
Progress: 48411 / 87665 (55.22%)^Z

```

we upload a php reverse shell

```bash

```

```bash
cat recipe.txt
Someone asked what our main ingredient to our spice soup is today.
 I figured I can't keep it a secret forever and told him it was love.

```

when looking at the clues some file immediately gains attraction 

```bash
ls
bin   home            lib         mnt         root  srv  vagrant
boot  \incidents/       lib64       opt         run   sys  var
dev   initrd.img      lost+found  proc        sbin  tmp  vmlinuz
etc   initrd.img.old  media       recipe.txt  snap  usr  vmlinuz.old

```

we can see a pcap file that we can see with wireshark

**click Follow>TCP Stream**

and we can see an interesting interaction

```bash
lennie:c4ntg3t3n0ughsp1c3
```

```bash

listening on [any] 4444 ...
connect to [10.23.176.115] from (UNKNOWN) [10.10.226.27] 52240
bash: cannot set terminal process group (1927): Inappropriate ioctl for device
bash: no job control in this shell
root@startup:~# whoami
whoami
root
root@startup:~# cd /root
cd /root
root@startup:~# ls
ls
root.txt
root@startup:~# cat root.txt
cat root.txt
THM{f963aaa6a430f210222158ae15c3d76d}

```
