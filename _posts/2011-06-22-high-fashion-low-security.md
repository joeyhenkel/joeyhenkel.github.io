---
title: "High Fashion, Low Security"
published: false
---

>NOTE: This post was originally made on June 22, 2011, in a unedited form. After discussions with law enforcement, I have redacted the content, and reposted it here for archiving.

Recently, my girlfriend had an interview with a local fashion designer, and was telling me about it at work one day. I was bored, and being fashion illiterate, I Googled the designers name to look the designer up. Apparently, he’d done work for the Kardashians, Lindsay Lohan, and Paris Hilton among others; a pretty big name in the industry apparently.

Then I found his websites actual Google entry. Suddenly, fashion seemed much more interesting to me…

<a href="/assets/highfashion/google_out.PNG"><img src="/assets/highfashion/google_out.PNG"></a>

On visiting the site, I saw several instances of this on the URL’s for various pages.

<a href="/assets/highfashion/unsecurehttps.PNG"><img src="/assets/highfashion/unsecurehttps.PNG"></a>
<a href="/assets/highfashion/url1.PNG"><img src="/assets/highfashion/url1.PNG"></a>
<a href="/assets/highfashion/url2.PNG"><img src="/assets/highfashion/url2.PNG"></a>

I knew the db’s were vulnerable, but I had to get a bit of background info on the target server. I ran a WhOIS on the site, and a reverse-ip search to see if it was hosted on a shared server. If it was hosted, I might get databases for more then one site, no need to compromise more then one target. Luckily, they have their own hosted server.

Eager to see what I could do with this, I fired up sqlmap in BT5. I knew there were lots of possible pages as targets, so i went with the following:

*[SQLMAP commands redacted]*

My purpose wasn’t to grab the entire servers contents, only the initial list of db’s, to determine which one had useful info.

It took a bit, and not surprisingly, found injectable parameters on almost every page on the site. It also spit out my db names as expected.

<a href="/assets/highfashion/dblist.PNG"><img src="/assets/highfashion/dblist.PNG"></a>

So I had the db names. One of them was what I was looking for, as it implied the db used for the site. It was probably also the current db, being that I came in via the website. Now I needed a listing of the tables, and any info on that db:

*[SQLMAP commands redacted]*

This is the same command, but instead of searching the db names, it searches the current db for the table names it carries. It did it’s job.

<a href="/assets/highfashion/tables.PNG"><img src="/assets/highfashion/tables.PNG"></a>

 So mostly website design type tables, but a few gems. Admins, users, and transactions all look juicy. Now to dump the contents of these:

*[SQLMAP commands redacted]*

At this point, I’m no longer concerned with seeing the structure, I want the beef. So this command dumps the contents of the entire current db. This takes awhile, but with 5 threads going, I found it bearable.

It still took awhile however, since apparently, they load every bit of text on the site into the SQL db, and it took awhile to grab it all. The exclude-systemdbs flag does just as it says, ignores any system db’s and dumps only the db’s that are user created.

>**NOTE**: run `sqlmap.py -h` to get a listing of all the options available.

In the end, I had the hashes for the admins table, full customer info from the customers table, as well as the full output of the transactions and users tables (which included MD5 hashes of CC numbers!!!). The hashes were all 64-bits from a MySQL db, which means they were probably SHA256.

Hashcat does a great job of cracking these given a good wordlist and GPU. It had been awhile since I had done any password cracking, so I had to reassemble some good wordlists from the web. I put the admin hash into a text file alone, as I had no interest in cracking all the customer passwords, or the MD5 hashes of the CC info; I’m not looking for trouble.

I ran Hashcat, expecting for a good hour or more wait, assuming the db admin had used a good password. It was done not even 5 seconds later. Thinking it was a fluke, I checked the output. The password it gave me was pathetic. Let’s just say it was less the 8 characters, all numeric, and sequential.

I could have brute-forced it by hand in less then a minute. I felt like a fool for putting so much time into prepping Hashcat, but that's the point of a hash, you don’t know what’s on the other side. I tested the password with a few online hash creators, and it matched up. I had the admins username and pathetic excuse for a password in my hands, along with all their customer info. Time to let them know about it.

I couldn’t find a specific name on the WHOIS data, only a generic email provided by the web host, which in theory should forward to the admin contact. Not wanting to take a chance that my warning would go unread, I also put in an email address that was listed under the admins table in the db, figuring it was for the admin or developer himself.

My first email to the automatic forwarder came back DOA, so I sent another to just the admin email I found. (On doing some Google-fu, I found he’s a freelance web designer from Argentina, which wouldn’t make my job any easier.) I waited a couple of days, but no response. I sent the same email again, but added the same text in Spanish as well, in case there was any language barrier.

In searching through the db’s, I found the email address for what appears to be Julian Chang himself. I included him in the email just to be thorough. After all, If my site was designed this poorly and leaked info like a running faucet, I’d want to know.

Lo and behold, a couple more days passed without answer, so I called the local shop. The woman who answered the phone sounded like she was expecting it to be a prank. She took down my information, and told me the “person who handles the website” would be calling me. I decided not to hold my breath.

I waited another week, then sent a final email to the developer, Julian Chang, and the woman who answered the phone when I called (found via `TheHarvester`, another great tool for another post.) I let them know what was at stake, and that if they didn’t contact me by 6/22, I would be posting this on my blog as a vulnerability disclosure, and it would be available to the public, including the bad guys.

6/22 came about, and I posted my original blog post about this. Since then, I called again and left voice mails, and also sent more emails. Overall, I sent 6 emails, and called 3 times; all with no response whatsoever. Frustrated, I began trying to find someone to help me get this problem resolved. I contacted the BBB, FTC, as well as US-CERT. Also, I took the extra step and contacted the appropriate law enforcement officials to help pressure the organization to fix the issue.

As of 8/25, I’m waiting to see if the extra pressure put on the organization makes them rethink their policies about storing and securing information. I have found that Julian Chang has an active twitter account *@julianchang*. Feel free to ask him why he’s ignored all my warnings and emails about this.