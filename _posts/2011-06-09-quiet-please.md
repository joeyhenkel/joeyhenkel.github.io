---
title: "Quiet Please, H4xing in progress..."
published: false
---

>NOTE: This post was originally made on June 9th, 2011. It has been re-posted here for archiving.

I commonly go to my local public library to study and use their free wi-fi. I normally can’t complain, it’s generally quiet, people leave you alone, and you mostly don’t have to deal with the hipsters who frequent the trendy coffee shops. On a recent visit, I got bored of studying, and started wondering what their infrastructure looked like.

I assumed that they had a fairly complex system, with firewalls blocking off the critical internal computer systems from the public wi-fi, or at least independent VLAN’s for each user on wi-fi, thus preventing traffic sniffing and general mischief. I mean, this is a government funded organization after all, why wouldn’t they do it right and take the secure route? But then again, this is Miami-Dade County…

I ran a simple scan with `airodump-ng` on BT5 and found that there are several open networks (common for public wi-fi) and a few closed networks, which I would assume are for internal use. Not a bad setup generally, except for one glaring problem: *The internal networks used simple WEP-40 encryption*.

<a href="/assets/quietplease/facepalm.jpg"><img src="/assets/quietplease/facepalm.jpg"></a>

Wow, I don’t know what’s on the other side of these networks, but I’m hoping it’s nothing important. Any skiddie with the right tools could simply crack this within a few minutes. Granted, the network SSID is not broadcast, but run airodump-ng for a few minutes and it picks up the name from the packets it captures. Security by obscurity at it’s finest…

> UPDATE: Just got a call from another person at MDPLS concerning their WEP issues. He claimed that they just got approval for upgrading to WPA2 because of this posting. He confirmed their reasoning for it was old devices (even an old Windows 2k device). From what he said, by September, the WEP encryption should be gone, and WPA/WPA2 should be in place on their internal networks.

I then realized, that if their using early-2000′s standards for Wi-Fi encryption, they probably have nothing to prevent MiTM attacks or traffic sniffing on the public networks. There wasn't.

<a href="/assets/quietplease/double_facepalm.jpg"><img src="/assets/quietplease/double_facepalm.jpg"></a>

I used [this script](https://www.backtrack-linux.org/forums/showthread.php?t=27509) from the BT5 How-To page, which grabs packets, redirects them through sslstrip, prints the info to my machine, and sends it to the end-user with a spoofed source. Within 30 minutes, I had at least 5 different passwords for FB, Twitter, G-mail, and others. 

> None of this info was saved BTW, I’m not into hacking peoples FB, I passed HS years ago.

Obviously, this is a more sophisticated effort, as it took a few more tools and concepts that a normal skiddie wouldn’t be aware to most times. But nevertheless, I was still able to do it, and with a pre-made script publicly available online. I’m sure the number of user/pass combos would have jumped by a huge amount if I had done this during Finals/Midterms week.

What really struck me however, was the physical aspect of the attack. Even if there was an IDS present on the network, all they’d be able to capture would be my MAC address, giving them either a generic 11:22:33:44:55 (faked MAC obviously) or a Acer MAC address, which would still be difficult to pick out in a crowd of dozens of people, each on their own laptops. (Obviously, this is a danger at most public wi-fi locations, not just this location. See end of post for prevention.)

These weren’t even the worst parts…I finished studying, and headed home. I was intrigued to see what info I could find out about their network from the outside this time. I ran a whois on mdpls.org, the main website of the library system. No netblock info, but it gave me the IP. So I ran a whois on that.

<a href="/assets/quietplease/netblock_info.png"><img src="/assets/quietplease/netblock_info.png"></a>

No dedicated netblock for MDPLS, but interesting that it was hosted by the neighboring counties school district. I realized I was going to have to use some google-fu to find the servers for the Library system alone, probably on sub-domains, or use some tools from BT5 to enumerate the servers.

I assumed maybe an OWA (Outlook Web Access), their website, and a Citrix MetaFrame for employee login, just like alot of organizations have. I fired up BT5 and opened up `fierce`, my favorite DNS enumeration tool.

If you’ve never used it before, it basically searches the DNS servers listed for the given domain, then tries a zone-transfer against them, and if it finds nothing, brute-forces the sub-domain names to find them. It’s a great way to save time during a pen-test, instead of hunting through google, and hoping for hits with `nmap`.

<a href="/assets/quietplease/fierce_zt.png"><img src="/assets/quietplease/fierce_zt.png"></a>

My first though was, *“You're kidding…”* I’ve done this on dozens of different domains, and never gotten this. I thought it must be a fluke at first. I mean, who would leave a DNS server for such a big organization open to the public Internet? So I fired up `DNSenum`, another BT5 tool that does mostly the same thing.

<a href="/assets/quietplease/dnsenum_zt.png"><img src="/assets/quietplease/dnsenum_zt.png"></a>

<a href="/assets/quietplease/triple_facepalm.jpg"><img src="/assets/quietplease/triple_facepalm.jpg"></a>

So they had mis-configured one of their DNS servers to dump the IP/DNS structure when asked. I obviously took it no further then this, as I wasn’t looking to do any damage, I was simply curious. But what If I had been malicious in nature? What if I had some crazy vendetta against the library over those damned late fees? That IP/DNS directory would be a gold mine.

I wouldn’t have to randomly hunt sub-domains, and waste time guessing. It was all right there, laid out for me to start port-scanning for vulnerable services. It’s the same as posting your home address on a public forum.

Sure, people can’t get in right away, but once they know where you live, and how many doors/windows you have, they can plan their attack a lot better.I held this information for a couple of months, not doing anything with it.

However, [I read a post at n00bz.net](http://n00bz.net/blog/2011/5/24/27-sqlmap-a-newspaper.html), where he found a vulnerability in one of our local newspapers and reported it right away and got it fixed. It made me realize that I had a responsibility to tell somebody about this, even more so as the library system is publicly funded. If they decided to do something about it or not was up to them, but if I ignored it, and they were hacked, I’d be just as guilty as the guy who hacked it.

I looked up the WHOIS info again on the domain, and found their tech contact. The number on the WHOIS led nowhere, so I called the main library switchboard and asked to speak with him. I called a few times with no luck, and sent several emails. He was busy, but after a couple of weeks got back to me.

I explained the problem I found with the DNS server, and the WEP encryption. He explained that they sometimes have an external audit done for open ports and such, and it never caught anything. (I find that either hard to believe, or the audit team wasn’t very thorough. I picked it up in literally 30 seconds.)

He seemed skeptical at first, until I started reading off internal addresses of servers from the list and sent him the output from fierce. When asked how they planned to fix it, he said he couldn't disclose because it was an internal matter. Understandable.

I told him again the danger of this, and he said the WEP problem was not his dept, but he would forward it to the correct person. All in all, he didn’t give me any sort of timeline for when it would be fixed, and didn’t take responsibility for it, although he did say that he was personally responsible for the DNS servers I scanned.

I understand that he would be skeptical of my findings, but what bothered me was the fact that it took almost 2 full weeks of emails and phone calls to even get him to call me back. Even when I spoke with him, he seemed extremely skeptical about my findings until I sent him the list.

I can understand how he would be skeptical of a random person calling with something like this, but as I’m writing this a few days after speaking with him, I have fierce up on the screen next to me and the zone transfer is still successful. Nothing has been done. I gave him a couple of days to get it fixed before I posted, so as to not compromise their system.

Again, I can understand that this takes time to fix, but in the end it’s a setting on the Server Role, and really shouldn’t take more then a few hours to rectify, much less a full 2 days. To me, it’s a classic example of real security taking a back seat to other things. When will people understand that without security, the other things are irrelevant? You can have the biggest, most expensive house on the block; but if you leave the front door open, it’ll soon be the emptiest house as well.

> UPDATE: Got an email from the DNS admin I had contacted, asking to scan again. Ran `fierce`, unable to do ZT, only normal brute-force. Looks like it was patched. Also, no hard feelings towards MDPLS, or the DNS admin who was my contact in this. This blog is for demonstrations of real-world security vulnerabilities, not to mock people and make them look bad. I’m sure, with the county being involved, that budgets, bureaucracy, and plenty of red-tape were involved and encountered to patch this problem. I’ve used to work at FIU, I know all about the BS that it takes to solve a small problem.

So how can all this be prevented and rectified? Well, the most obvious way to rectify the WEP issue is to use WPA/WPA2 encryption on the internal Wi-Fi networks. They may be using WEP to provide compatibility with older devices such as printers or some other machines.

Honestly, I’ve never seen any device newer then 2006 that doesn’t support WPA/WPA2. If this is the case, they need to seriously think about updating their devices, or even making their internal network non-wireless, and fire walled away from the public wi-fi.As for the public network’s, I do acknowledge this has little to do with the library.

Although, most modern AP’s now come with a setting to VLAN each device away from each other. Below is a screencap from my work device highlighting the setting.

Note, ours is obviously turned off in the office because we need to share files with each other, and it’s a trusted environment. If this was a public device, this could be set to enabled, and each user would be on their own VLAN, thus preventing them from sniffing anything but their own traffic.

<a href="/assets/quietplease/wifi_settings.png"><img src="/assets/quietplease/wifi_settings.png"></a>

Also, note the WPA2-PSK setting in the above image. On this, like all AP’s, it really is simply a drop down box to select the encryption type. Although the library should take steps to protect users, it ultimately is the users responsibility to protect themselves. There are plenty of ways to protect yourself while on a public network.

[Lifehacker has had a great number of articles on the topic](https://lifehacker.com/how-to-secure-and-encrypt-your-web-browsing-on-public-n-5763170), and it really isn’t too hard to do. Even if the article is too techy for you, just be safe. Don’t enter any passwords over unsecured wi-fi networks. Better safe then sorry.

The DNS Zone Transfer problem takes a bit of configuration on the DNS server. Most modern server OS’s will lock down features like that by default, which tells me that it’s either someone incompetent behind the wheel, or an older Windows OS (2000 or even NT) that has been configured correctly.

However, if you noticed in the screencap above, one of the DNS servers didn’t respond to the zone transfer, but the other did, which means that the leak was not intentional, and is probably the result of a misconfigured setting.

## Lessons learned:

- ALWAYS use common sense on a public network. Don’t enter passwords for ANYTHING unless you know the data will be secure in transit.



- Hiding the SSID is a joke, as is MAC filtering. 
- Don’t be lazy. WPA/WPA2 cost very little, if nothing at all, to implement, and it will save you tons of heartache.
- Always make sure to do an external pen-test of your infrastructure, with a reputable firm. If this had been done here, any security pro (or even student in my case) would have picked this up right away.
- If contacted with a security vulnerability, take it seriously, no matter how random or trivial it may seem. In the end, *"A pint of sweat saves a gallon of blood."* - Patton
- Use the most up to date OS possible. Modern server systems come with all features locked down out of the box. It’s the server admins job to enable what they need. Older systems come with alot of their features enabled by default, and thus open to exploitation if not taken care of. (Note: This applies mostly to Windows, as Unix systems have always locked down features, or required the install of additional modules. Server 2008/R2 lock down all features, where 200/NT do not. I’m not sure on 2003.)

*Thanks to Jason @ [n00bz.net](http://n00bz.net/) for valuable info on submitting the vulnerability.*





