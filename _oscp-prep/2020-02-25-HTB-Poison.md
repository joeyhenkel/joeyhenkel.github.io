---
title: "Hack The Box - Poison"
date: 2020-02-25
tags: [oscp, htb, freebsd]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "LFI, SSH port forwarding, and VNC"
picture: /assets/htb-poison/machine_info.png
author_profile: false
---

![](/assets/htb-poison/machine_info.png)

## Enumeration

Nmap scans show only TCP ports 22 and 80 open, running SSH and HTTP. The web server is Apache 2.4.29, running on FreeBSD.

```
# Nmap 7.80SVN scan initiated Tue Feb 25 10:37:37 2020 as: nmap -A -Pn -p- --reason -oN scans/tcp_full 10.10.10.84
Nmap scan report for 10.10.10.84
Host is up, received user-set (0.046s latency).
Not shown: 65533 closed ports
Reason: 65533 conn-refused
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 7.2 (FreeBSD 20161230; protocol 2.0)
| ssh-hostkey: 
|   2048 e3:3b:7d:3c:8f:4b:8c:f9:cd:7f:d2:3a:ce:2d:ff:bb (RSA)
|   256 4c:e8:c6:02:bd:fc:83:ff:c9:80:01:54:7d:22:81:72 (ECDSA)
|_  256 0b:8f:d5:71:85:90:13:85:61:8b:eb:34:13:5f:94:3b (ED25519)
80/tcp open  http    syn-ack Apache httpd 2.4.29 ((FreeBSD) PHP/5.6.32)
|_http-server-header: Apache/2.4.29 (FreeBSD) PHP/5.6.32
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
Service Info: OS: FreeBSD; CPE: cpe:/o:freebsd:freebsd
```

Navigating to the webpage gives a fairly basic page, listing some PHP pages, and a field to insert a file to test.

![](/assets/htb-poison/port80_home.png)

Additional web directory search with `dirsearch` found no additional directories or files.

---

## Initial Shell

### LFI on PHP test page

When we put in the first test file in the list, `ini.php`, we're redirected to the contents of the file. Not much interesting besides a few directory names here. However, that URL parameter looks like a great spot for an LFI. By the way, viewing the source on these pages makes them much more readable.

![](/assets/htb-poison/possible_lfi.png)

We can test this by inserting `/etc/passwd` into the `file` parameter in the URL. Sure enough, we have a solid LFI here, that can read out files on the server. From the output of `/etc/passwd`, we can see that we have a username of `charix`.

![](/assets/htb-poison/web_lfi_etcpasswd.png)

### Cracking base64 encoded password

As interesting as the LFI is, we should keep looking at the other listed PHP files.

The `info.php` file gives a `uname -a` readout, containing the OS and kernel info.

![](/assets/htb-poison/info_php.png)

When we open `listfiles.php`, we get another nice surprise. The `pwdbackup.txt` file looks to be in the same location as the PHP files we've been looking at, and should be accessible via the LFI we found.

![](/assets/htb-poison/listfiles_php.png)

Pulling up the `pwdbackup.txt` file in the LFI gives us this block of text.

```
This password is secure, it's encoded atleast 13 times.. what could go wrong really.. Vm0wd2QyUXlVWGxWV0d4WFlURndVRlpzWkZOalJsWjBUVlpPV0ZKc2JETlhhMk0xVmpKS1IySkVU bGhoTVVwVVZtcEdZV015U2tWVQpiR2hvVFZWd1ZWWnRjRWRUTWxKSVZtdGtXQXBpUm5CUFdWZDBS bVZHV25SalJYUlVUVlUxU1ZadGRGZFZaM0JwVmxad1dWWnRNVFJqCk1EQjRXa1prWVZKR1NsVlVW M040VGtaa2NtRkdaR2hWV0VKVVdXeGFTMVZHWkZoTlZGSlRDazFFUWpSV01qVlRZVEZLYzJOSVRs WmkKV0doNlZHeGFZVk5IVWtsVWJXaFdWMFZLVlZkWGVHRlRNbEY0VjI1U2ExSXdXbUZEYkZwelYy eG9XR0V4Y0hKWFZscExVakZPZEZKcwpaR2dLWVRCWk1GWkhkR0ZaVms1R1RsWmtZVkl5YUZkV01G WkxWbFprV0dWSFJsUk5WbkJZVmpKMGExWnRSWHBWYmtKRVlYcEdlVmxyClVsTldNREZ4Vm10NFYw MXVUak5hVm1SSFVqRldjd3BqUjJ0TFZXMDFRMkl4WkhOYVJGSlhUV3hLUjFSc1dtdFpWa2w1WVVa T1YwMUcKV2t4V2JGcHJWMGRXU0dSSGJFNWlSWEEyVmpKMFlXRXhXblJTV0hCV1ltczFSVmxzVm5k WFJsbDVDbVJIT1ZkTlJFWjRWbTEwTkZkRwpXbk5qUlhoV1lXdGFVRmw2UmxkamQzQlhZa2RPVEZk WGRHOVJiVlp6VjI1U2FsSlhVbGRVVmxwelRrWlplVTVWT1ZwV2EydzFXVlZhCmExWXdNVWNLVjJ0 NFYySkdjR2hhUlZWNFZsWkdkR1JGTldoTmJtTjNWbXBLTUdJeFVYaGlSbVJWWVRKb1YxbHJWVEZT Vm14elZteHcKVG1KR2NEQkRiVlpJVDFaa2FWWllRa3BYVmxadlpERlpkd3BOV0VaVFlrZG9hRlZz WkZOWFJsWnhVbXM1YW1RelFtaFZiVEZQVkVaawpXR1ZHV210TmJFWTBWakowVjFVeVNraFZiRnBW VmpOU00xcFhlRmRYUjFaSFdrWldhVkpZUW1GV2EyUXdDazVHU2tkalJGbExWRlZTCmMxSkdjRFpO Ukd4RVdub3dPVU5uUFQwSwo= 
```

Interesting. It seems they base64 encoded a password, and repeated it at least 13 times to better obfucate it. Let's break it down. I saved the base64 text to a file called `b64.pass`, and used the below command to iterate thorugh it.

```bash
data=$(cat b64.pass); for i in $(seq 1 13); do data=$(echo $data | tr -d ' ' | base64 -d); done; echo $data
```

Running the command gives us a password of `Charix!2#4%6&8(0`.

![](/assets/htb-poison/charix_pw.png)

We can now use this password on the SSH server as `charix`.

![](/assets/htb-poison/ssh_charix.png)

We can read `user.txt` from `/home/charix/user.txt`

![](/assets/htb-poison/user_proof.png)

---

## Privilege Escalation

### Secret.zip file

When we look in the `/home/charix` directory, we can see the `secret.zip` file.

![](/assets/htb-poison/secretzip.png)

After transferring it locally, we can try to unzip it, but it seems it requires a password.

![](/assets/htb-poison/secretzip_pwneeded.png)

Before jumping into cracking the ZIP, let's just try to reuse the password for the `charix` account that we already have.

![](/assets/htb-poison/secret_unzip.png)

It worked! It seems the ZIP contained another file called `secret`, which looks to be a small binary file of some sort. Not really useful right now, but let's keep it in mind.

### TightVNC Service

In our enumeration, the `ps -aux` command reveals that there is a TightVNC service running as `root`.

![](/assets/htb-poison/xvnc_service_psaux.png)

A look at `netstat -an` reveals that indeed, it's running on local ports 5801/5901, which is why we didn't see them with our initial enumeration.

![](/assets/htb-poison/netstat.png)

Since these are local ports, we can use our existing SSH credentials as `charix` to open an SSH tunnel for connection to them using our local machine.

The command `ssh -L 5901:127.0.0.1:5901 charix@10.10.10.84` will open a new SSH connection, that will forward all traffic from port 5901 on our machine, to port 5901 on the target machine.

So remember that `secret` file we unzipped? We can use it as a credential for VNC. The below command will open a VNC session using the forwarded port, and captured credential.

```
vncviewer 127.0.0.1:5901 -passwd secret
```

![](/assets/htb-poison/tightvnc_connections.png)

We can read `root.txt` at `/root/root.txt`.

![](/assets/htb-poison/root_proof.png)

