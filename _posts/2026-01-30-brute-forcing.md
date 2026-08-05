---
title: "Brute forcing"
date: 2026-01-30 12:00:00 +0300
categories: [Writeups, HTB Academy]
tags: [htb-academy, notes, cybersecurity, pentesting, methodology]
description: "Hydra attack workflows for HTTP POST/GET, SSH, FTP, wordlist manipulation, rate-limit bypasses, and session state handling."
image:
  path: /assets/img/posts/brute_force.jpg
  alt: Brute forcing
---

# Hybrid attack example

- Minimum length: 8 characters
- Must include:
    - At least one uppercase letter
    - At least one lowercase letter
    - At least one number

To extract only the passwords that adhere to this policy, we can leverage the powerful command-line tools available on most Linux/Unix-based systems by default, specifically `grep` paired with regex. We are going to use the [darkweb2017-top10000.txt](https://github.com/danielmiessler/SecLists/blob/master/Passwords/Common-Credentials/darkweb2017_top-10000.txt) password list for this. First, download the wordlist

Hybrid Attacks

```bash
Fau5t@htb[/htb]$ wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/darkweb2017_top-10000.txt

```

Next, we need to start matching that wordlist to the password policy.

Hybrid Attacks

```bash
Fau5t@htb[/htb]$ grep -E '^.{8,}$' darkweb2017-top10000.txt > darkweb2017-minlength.txt

```

This initial `grep` command targets the core policy requirement of a minimum password length of 8 characters. The regular expression `^.{8,}$` acts as a filter, ensuring that only passwords containing at least 8 characters are passed through and saved in a temporary file named `darkweb2017-minlength.txt`.

Hybrid Attacks

```bash
Fau5t@htb[/htb]$ grep -E '[A-Z]' darkweb2017-minlength.txt > darkweb2017-uppercase.txt

```

Building upon the previous filter, this `grep` command enforces the policy's demand for at least one uppercase letter. The regular expression `[A-Z]` ensures that any password lacking an uppercase letter is discarded, further refining the list saved in `darkweb2017-uppercase.txt`.

Hybrid Attacks

```bash
Fau5t@htb[/htb]$ grep -E '[a-z]' darkweb2017-uppercase.txt > darkweb2017-lowercase.txt

```

Maintaining the filtering chain, this `grep` command ensures compliance with the policy's requirement for at least one lowercase letter. The regular expression `[a-z]` serves as the filter, keeping only passwords that include at least one lowercase letter and storing them in `darkweb2017-lowercase.txt`.

Hybrid Attacks

```bash
Fau5t@htb[/htb]$ grep -E '[0-9]' darkweb2017-lowercase.txt > darkweb2017-number.txt

```

This last `grep` command tackles the policy's numerical requirement. The regular expression `[0-9]` acts as a filter, ensuring that passwords containing at least one numerical digit are preserved in `darkweb2017-number.txt`.

Hybrid Attacks

```bash
Fau5t@htb[/htb]$ wc -l darkweb2017-number.txt

89 darkweb2017-number.txt

```

As demonstrated by the output above, meticulously filtering the extensive 10,000-password list against the password policy has dramatically narrowed down our potential passwords to 89.

# hydra stuff

| **Parameter** | **Explanation** | **Usage Example** |
| --- | --- | --- |
| `-l LOGIN` or `-L FILE` | Login options: Specify either a single username (`-l`) or a file containing a list of usernames (`-L`). | `hydra -l admin ...` or `hydra -L usernames.txt ...` |
| `-p PASS` or `-P FILE` | Password options: Provide either a single password (`-p`) or a file containing a list of passwords (`-P`). | `hydra -p password123 ...` or `hydra -P passwords.txt ...` |
| `-t TASKS` | Tasks: Define the number of parallel tasks (threads) to run, potentially speeding up the attack. | `hydra -t 4 ...` |
| `-f` | Fast mode: Stop the attack after the first successful login is found. | `hydra -f ...` |
| `-s PORT` | Port: Specify a non-default port for the target service. | `hydra -s 2222 ...` |
| `-v` or `-V` | Verbose output: Display detailed information about the attack's progress, including attempts and results. | `hydra -v ...` or `hydra -V ...` (for even more verbosity) |
| `service://server` | Target: Specify the service (e.g., `ssh`, `http`, `ftp`) and the target server's address or hostname. | `hydra ssh://192.168.1.100` |
| `/OPT` | Service-specific options: Provide any additional options required by the target service. | `hydra http-get://example.com/login.php -m "POST:user=^USER^&pass=^PASS^"` (for HTTP form-based authentication) |

### **Brute-Forcing a Web Login Form**

Suppose you are tasked with brute-forcing a login form on a web application at `www.example.com`. You know the username is "admin," and the form parameters for the login are `user=^USER^&pass=^PASS^`. To perform this attack, use the following Hydra command:

Hydra

```bash
Fau5t@htb[/htb]$ hydra -l admin -P passwords.txt www.example.com http-post-form "/login:user=^USER^&pass=^PASS^:S=302"
```

# **Targeting Multiple SSH Servers**

Consider a situation where you have identified several servers that may be vulnerable to SSH brute-force attacks. You compile their IP addresses into a file named `targets.txt` and know that these servers might use the default username "root" and password "toor." To efficiently test all these servers simultaneously, use the following Hydra command:

```
        shellsession
Fau5t@htb[/htb]$ hydra -l root -p toor -M targets.txt ssh

```

## Hydra's `http-post-form`

Hydra's `http-post-form` service is specifically designed to target login forms. It enables the automation of POST requests, dynamically inserting username and password combinations into the request body. By leveraging Hydra's capabilities, attackers can efficiently test numerous credential combinations against a login form, potentially uncovering valid logins.

The general structure of a Hydra command using `http-post-form` looks like this:

```
        shellsession
Fau5t@htb[/htb]$ hydra[options] target http-post-form"path:params:condition_string"

```

# **Understanding the Condition String**

In Hydra’s `http-post-form` module, success and failure conditions are crucial for properly identifying valid and invalid login attempts. Hydra primarily relies on failure conditions (`F=...`) to determine when a login attempt has failed, but you can also specify a success condition (`S=...`) to indicate when a login is successful.

The failure condition (`F=...`) is used to check for a specific string in the server's response that signals a failed login attempt. This is the most common approach because many websites return an error message (like "Invalid username or password") when the login fails. For example, if a login form returns the message "Invalid credentials" on a failed attempt, you can configure Hydra like this:

```bash
        bash
hydra ... http-post-form "/login:user=^USER^&pass=^PASS^:F=Invalid credentials"

```

## hydra login form

```bash
example
hydra -L top-usernames-shortlist.txt -P 2023-200_most_used_passwords.txt 
-f IP -s 5000 http-post-form 
"/:username=^USER^&password=^PASS^:F=Invalid credentials"
```

## specific password policy grep

passwords for us, but Jane's company, AHI, has a rather odd password policy.

- Minimum Length: 6 characters
- Must Include:
    - At least one uppercase letter
    - At least one lowercase letter
    - At least one number
    - At least two special characters (from the set `!@#$%^&*`)

As we did earlier, we can use grep to filter that password list to match that policy:

```
        shellsession
Fau5t@htb[/htb]$ grep -E'^.{6,}$' jane.txt| grep -E '[A-Z]' | grep -E '[a-z]' | grep -E '[0-9]' | grep -E '([!@#$%^&*].*){2,}' > jane-filtered.txt

```

# medusa

```bash
 medusa -h <IP> -n <PORT> -u sshuser -P 2023-200_most_used_passwords.txt 
 -M ssh -t 3

```

# username-anarchy

```
        shellsession
Fau5t@htb[/htb]$ ./username-anarchy Jane Smith> jane_smith_usernames.txt

```

Upon inspecting `jane_smith_usernames.txt`, you'll encounter a diverse array of usernames, encompassing:

- Basic combinations: `janesmith`, `smithjane`, `jane.smith`, `j.smith`, etc.
- Initials: `js`, `j.s.`, `s.j.`, etc.
- etc

# CUPP

cupp -i 

for personalized password list
