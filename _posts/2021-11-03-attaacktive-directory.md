---
title: TryHackMe - Attacktive Directory
author: Zach Crosman
date: 2021-11-03 18:32:00 -0500
categories: [TryHackMe, Active Directory]
tags: [impacket, secretsdump, psexec, kerberos, kerbrute]
toc: false
pin: false
math: false
mermaid: true
---

TryHackMe Room: [Attactive Directory](https://www.tryhackme.com/room/attacktivedirectory)


## Initial Enumeration
```bash
sudo nmap --top-ports 1000 -sV 10.10.146.23

Nmap scan report for 10.10.146.23
Host is up (0.14s latency).
Not shown: 987 closed ports
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2021-11-03 00:49:22Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: spookysec.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: spookysec.local0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
Service Info: Host: ATTACKTIVEDIREC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

## Enumeration of Kerberos

With the nmap scan, we identified above that this host is running Active Directory so we can try abusing Kerberos. If we already had valid credentials, we could use GetUserSPNs to get TGS hashes. So far we don’t have credentials, so we have to find another method. For this lab, a user list was provided to enumerate user accounts. Kerbrute userenum can be used to validate users on the domain. Once we have a valid list of users, GetNPUsers can be used to identify if any of these accounts do not require Kerberos preauthentication set (`UF_DONT_REQUIRE_PREAUTH`).

kerbrute userenum is used to identify valid user accounts

![Desktop View](/images/attacktive_directory/GetNPUsers.png)
_GetNPUsers is used to get TGTs_

![Desktop View](/images/attacktive_directory/hashcat.png)
_The credentials for backup were cracked with hashcat_

## Initial Access

![Desktop View](/images/attacktive_directory/smbclient.png)
_Enumeration of the smb shares_

![Desktop View](/images/attacktive_directory/backup_creds.png)
_backup_credentials in the backup smb share_

![Desktop View](/images/attacktive_directory/creds.png)
_Contents of backup_credentials.txt_

The string found in the text file looks like it could be base64 encoded. Even if I think I already know how a string could be encoded, I like to use cyberchef to decode it. The site has many recipes on string encoding/decoding. One of my favorite features is the “Magic” recipe. This recipe will use different types of manipulation available on the site. It wasn’t needed in this case, but it will also attempt to identify multiple layers of string manipulation. With the string found in backup_credentials.txt, cyberchef identified that it was encoded with base64. This revealed the credentials of the backup account.

![Desktop View](/images/attacktive_directory/cyberchef.png)

## Privledge Escalation

Since we now have a low-level user account we can now try to dump hashes with secretsdump.py. After the hashes are obtained, we can get administrative access to the domain controller with psexec.

![Desktop View](/images/attacktive_directory/secretsdump.png)
_Dumping hashes of the DC_

![Desktop View](/images/attacktive_directory/psexec.png)
_Administrator access with psexec.py_


