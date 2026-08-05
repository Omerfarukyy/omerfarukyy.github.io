---
title: "Agent Sudo"
date: 2025-08-24 12:00:00 +0300
categories: [Writeups, TryHackMe]
tags: [tryhackme, ctf, writeup, pentesting, security]
description: "User-Agent header spoofing, FTP steganography extraction, SSH cracking, and CVE-2019-14287 Sudo privilege escalation."
image:
  path: /assets/img/posts/agent_sudo_thm.png
  alt: Agent Sudo
---

# Enumaration

## Nmap

```jsx
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ef:1f:5d:04:d4:77:95:06:60:72:ec:f0:58:f2:cc:07 (RSA)
|   256 5e:02:d1:9a:c4:e7:43:06:62:c1:9e:25:84:8a:e7:ea (ECDSA)
|_  256 2d:00:5c:b9:fd:a8:c8:d8:80:e3:92:4f:8b:4f:18:e2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Annoucement
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.95%E=4%D=8/22%OT=21%CT=1%CU=34592%PV=Y%DS=2%DC=T%G=Y%TM=68A864D
OS:8%P=x86_64-pc-linux-gnu)SEQ(SP=100%GCD=1%ISR=108%TI=Z%CI=I%II=I%TS=A)SEQ
OS:(SP=100%GCD=1%ISR=10D%TI=Z%CI=I%II=I%TS=A)SEQ(SP=106%GCD=1%ISR=106%TI=Z%
OS:CI=I%II=I%TS=A)SEQ(SP=107%GCD=1%ISR=10C%TI=Z%CI=I%II=I%TS=A)OPS(O1=M508S
OS:T11NW7%O2=M508ST11NW7%O3=M508NNT11NW7%O4=M508ST11NW7%O5=M508ST11NW7%O6=M
OS:508ST11)WIN(W1=68DF%W2=68DF%W3=68DF%W4=68DF%W5=68DF%W6=68DF)ECN(R=Y%DF=Y
OS:%T=40%W=6903%O=M508NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=
OS:)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T
OS:=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=
OS:0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=
OS:164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 554/tcp)
HOP RTT      ADDRESS
1   65.48 ms 10.9.0.1
2   65.55 ms 10.10.116.238

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.37 seconds

```

![image.png](/assets/img/writeups/agent-sudo_image.png)

![image.png](/assets/img/writeups/agent-sudo_image_1.png)

```jsx
└─$ hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.10.21.186
[21][ftp] host: 10.10.21.186   login: chris   password: crystal

```

“We got 3 files, download them to your local machine using following command `mget *`” this is also useful

`zip2john 8702.zip>output.txt` This will give hash of the password in txt file.

`john --format=zip output.txt` This will get us the password

i really did not much understand these but useful features.

“Open the txt file and we got another hash in it. I used `hashid <hash>` to find which type of hash it is but it seems it gives the wrong answer. After some time I used [https://www.tunnelsup.com/hash-analyzer/](https://www.tunnelsup.com/hash-analyzer/) and came to know its “base64”. Decode it using BurpSuite and we got “Area51” as our answer.”

i do something wrong probably and it is not working for me

```jsx
8702.zip/To_agentR.txt:$zip2$*0*1*0*4673cae714579045*67aa*4e*61c4cf3af94e649f827e5964ce575c5f7a239c48fb992c8ea8cbffe51d03755e0ca861a5a3dcbabfa618784b85075f0ef476c6da8261805bd0a4309db38835ad32613e3dc5d7e87c0f91c0b5e64e*4969f382486cb6767ae6*$/zip2$:To_agentR.txt:8702.zip:8702.zip

```

[https://futureboy.us/stegano/decinput.html](https://futureboy.us/stegano/decinput.html) for steogn- whatever stuff

i am detached mentally but, we first found these, with jack decoded zip and later hash that comes with it, then later messed with photos again and other photo has the password

john hackerrules!

”Download the `Alien_autospy.jpg` file using `sudo scp james@10.10.235.17:Alien_autospy.jpg ~/` Now do a reverse image search on google and check the fox news website (I used the hint for this one).”
