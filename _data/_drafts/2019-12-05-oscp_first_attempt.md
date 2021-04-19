---
title: "OSCP Exam - First Attempt"
published: false
---

So, my first attempt at the OSCP exam has come and gone. Honestly, it was a crash and burn. I really thought I was prepared, and ready to knock it out.


I'll lay out my prep for the exam here, along with my spoiler-free thoughts and problems I ran into. I'll also review what I think I need to improve, and my plan for going forward.


## Prep

In early-August, I reached Pro level on HackTheBox. Granted, much of this was easier boxes, and took a good amount of discussions with other users to help me figure out how to get through the boxes. I knew that I still had a ton to learn, but felt that I had a good foundation of the process thanks to HTB.


I figured that if my goal was the OSCP, HTB would only get me so far. So I bit the bullet, and bought 90-days of lab time, since I would have limited time each day to work on the course, due to family and work obligations. My lab time was due to end Nov 29th, and I pre-scheduled my exam for December 2nd at 11 AM.


## PWK Course & Lab 

Jumping into the PWK lab and course is a bit overwhelming. The coursework is all condensed into a single PDF file, which can make it a bit difficult to navigate. Even worse, the videos, while great in quality, are only given names like pwk-10.mp4, which makes it difficult to pick and choose specific topics to review. They do give you an HTML page that should play the videos embedded, but I was never able to get it to work right.


The labs were well setup, with a functional control panel for reverting machines, and entering network-secret proofs, that allow access to additional subnets. Most of the boxes in the labs are related to each other, and it makes it interesting to navigate your way through, using hints from other machines to help take down others.


Like most people have mentioned though, the lab machines are OLD. Most windows machines are running Windows XP, and even some still run Server 2000. The newest Windows box in the lab that I saw was Windows 8. The services also tend to be on the older side, with most not being updated since at least 2013-2014.


I get it. If you can learn the process, the age of the service or OS shouldn't matter. My problem was that with such old software, most of what you'll find is easy kernel exploits, especially with privilege escalation. If Offensive Security intended to teach you a certain tool or method with each different box, having so many easy wins makes it hard to narrow your gaze to a specific attack or exploit. Looking back, having so many easy outs does make you really need to focus on what they are trying to teach you.


In total, I was able to take down 23 out of 50 machines in the labs, and gained access to both the DEV and IT subnets. The labs are a lot of fun, because you have full run of them. While HTB limits you to a single machine at a time, you can (and often need to) use previously rooted targets to take down other machines. While this isn't part of the exam (which is a shame), it helps cement your understanding of networking and pivoting.


## The exam

My game plan going in was to hit the buffer overflow box first, while running autorecon on the remaining 4 boxes. After BOF, I would take down the simple 10-point box to get the easy wins out of the way, then tackle the 2x 20-point machines. Getting these 2 would get me 75 points, which was enough to pass. I could then turn my attention to the remaining 25-point box, to give myself a bit of a cushion in points if I needed.


I can't go into too much detail on the specifics of the boxes I came across in the lab, but they were orders of magnitude different then what I had expected.

 

Here's a spoiler-free breakdown:


### Target 1 - Buffer Overflow (25 points)

This was the oldest machine on the exam, and my only win. It was a Windows 7 pro box, with a vulnerable application running on it, very similar to the vulnserver you target in the lab. They do provide you a debugging machine, which has a pre-made POC for you to base your exploit on.


Nothing special here, except be sure to be thorough with searching for bad characters.


Took me about an hour to get to proof.txt, which is about what I was expecting.


### Target 2 - 10-point box (Linux running CentOS 7)


From everything I had read, this box should have been a simple matter of enumeration, finding the exploit, customizing it for the target, and getting root.


Enumeration easily turned up the service I needed to target, and Exploit-DB showed my various exploits for it. This is where my issues began. I could not, for the life of me, enumerate the exact version of the software running, which made choosing the correct exploit a game of hit-or-miss.


I had one specific exploit that I thought for sure would work, but no matter how much I customized it, it did not want to take.


There were a few other services running that were potentially vulnerable, but I determined them to be rabbit-holes based on the system, and the exploits needed for them.


This machine frustrated more then any of the others, due to it being the "easy" box of the group.


### Target 3 - 20-point Linux box (Debian 10)

This machine gave me a simple landing page, which contained a rabbit-hole right off the bat. It took me a few more good enumeration scans to find the hidden subdirectory running a CMS. I was able to enumerate the CMS and get a username, but even with the hints the CMS was giving me, I wasn't able to get a login. Brute-forcing would have been out of the question, as the username got suspended after about 5 attempts, so I had to revert a few times.


### Target 4 - 20-point Windows box (Windows 10)


This machine had a basic Apache/PHP/MySQL setup, but did provide tons of info via a phpinfo() page I found. I had a username, but honestly, I was so wrapped up in being angry about the 10-point box, that I didn't follow-through on this box as well as I should have. I'm sure there was an easy win there that I overlooked.


### Target 5 - 25-point box (Debian 10)

This box didn't give me much by way of the web, but was running a TON of services. Again, I didn't enumerate too much here as I should have, as it was getting late, I was tired, and very frustrated.


I ended up calling the exam after about 20 hours. I knew that I wasn't going to get much else, mostly due to the anxiety and frustration I had built up in myself.


## What to improve
 
### Enumeration

I need to get out of the habit of finding something interesting, and immediately diving down it's rabbit hole. This leads to a ton of wasted time and effort, when something even better hasn't even been found yet. Instead, start from your basic scans, and work your way down the list. Just grabbing a service version isn't enough. What else can you get it to tell you?


I used autorecon in the labs and exam, and found it to work great. However, there are certain things that an automated scan can't do. I need to get better about following up these failed scans with customized scans and checks.


### Documentation

Yes, you need to write down what you did to get to your target. but what about all the steps that didn't work? Why didn't they work?


I've been using a simple markdown page for notes, but I think I need to find a way to keep better track of things. The obvious choice is CherryTree, but I honestly can't stand it. It looks ancient, screws with formatting on text, craps out image quality, and has crashed on me several times with larger data chunks and images. I need to find something better to take notes with.


### Know your tools

It's so easy to get wrapped up in developing a process, that you only stick to a certain set of tools, and a certain way to use them. Maybe the one tool you love doesn't handle HTTP redirects well. Maybe it craps out when another tool will do a better job in a certain situation.


Know what tools are at your disposal, and experiment with them. Maybe a scan of something with one tool will miss what you'll discover with another.


### Stick to the script

I honestly believe that I psyched myself out during the exam. I got so frustrated that I was hitting walls everywhere, that I lost track of what I should have been doing, which was following my process. The deeper I got into the exam, the worse it got. With that mentality, I had no chance of pulling out a win. Note that the one machine I did get was BOF, which I had a solid, documented process for. Why didn't I have something like this for regular boxes?


## Going forward

I'm taking a bit of time off during the holidays, and just knocking out some simple HTB machines to keep fresh. Once the new year rolls around, I'll buy another 30-60 days of lab time, and try the exam again in late January/early-February.


 






