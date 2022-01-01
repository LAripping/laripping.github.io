---
layout: post
title: "A Noob's guide to OSEP"
excerpt: "OSEP is a new cert. It was introduced by Offsec in November 2020 and it immediately felt like they would finally address the gap in their certs for the netsec area, while simultaneously throwing Offsec in the market of Red Team related certs/courses.<br/><br/>"
categories:
  - "blog-posts"
tags:
  - osep
  - offsec
last_modified_at: 2021-12-21T15:59:00
post-img: assets/img/offsec.png
---

{% include toc.html %}

## Intro

OSEP is a new cert. It was introduced by Offsec in November 2020 and it immediately felt like they would finally address the gap in their certs for the netsec area, while simultaneously throwing Offsec in the market of Red Team related certs/courses. 

I was always an Offsec fan, I like how they shape their exams, their labs, the never-expiring certificates they hand out, even the controversial Try Harder motto, and the overall recognition by the industry. Just by looking at the associated "PEN-300 - Evasion Techniques and Breaching Defences" course's [syllabus](https://www.offensive-security.com/documentation/PEN300-Syllabus.pdf) one can quickly realise it focuses primarily in Windows/AD/Kerberos concepts, with a twist of Linux-based DevOps solutions commonly found (and exploited) in these sorts of networks. It's also a great and rare source for all the methods and techniques used in the remaining stages of the killchain i.e. beyond Enumeration, Exploitation & Privilege Escalation which are covered by OSCP. I'm talking things like Post Exploitation (Persistence, Reconnaissance) and Lateral Movement, for which you might find some resources here and there, but not an organised, zero-to-hero kind of course like the Offsec ones. Finally, most of the content is coated with AV evasion techniques and progressively more details into the internals of these solutions, instead of just a one-off chapter of theory. 
You'll also be facing fully patched, up to date machines, which is kinda cool if you ask me, no vulnerable services listening with known exploits / MSF modules.

Now let's talk about the elephant in the room: 

There's a big debate on whether this is or isn't a red team cert. Not a red-teamer myself, I can't say whether it prepares you for what you'll see in the field performing these kinds of exercises, but I can certainly confirm that it does skill you up sufficiently in the areas that are most relevant. Here's how Offsec themselves put it: 

> *The main purpose of a red team test is to test or train the security personal in the client organization, which are referred to as the blue team. While many techniques between penetration tests and red teams tests overlap, the goals are different.<br/><br/>PEN-300 will peovide the knowledge and techniques required to perform advanced penetration tests against mature organizationswith a developed security level.* ***It is not a Red Team course***


One way or another, the foundation and knowledge gained through the course and labs will hopefully help noobs like me to get into such practices of consulting like purple teams, red teams, Active Directory reviews. In my case, I now feel much more ready to raise my hand when the request comes in, and actively chase such opportunities. Whether that's enough to get me where I want to be remains to be seen :crossed_fingers:

Without further ado, let's get into the meat and potatoes of this thing.


## The Course & Labs

After you sign up you'll receive a welcome pack including: 

1. a whopping 700-page PDF
2. a best-of DVD collection of pre-recorded walkthroughs of the exact same thing on a computer - Narrated by a guy with a really cool voice and no emotion
3. a VPN profile unlocking a magic network - all for yourself! No sharing, no random reverts by other students, flags getting deleted / boxes being reconfigured or broken - All thanks to the power of the cloud! :cloud:

While some say it's best to interleave reading the PDF / watching the videos and trying the challenges as you go or when bored,  I personally prefer to keep theoretic and practical activities separate. This helped me track my overall progress and manage my excitement for what I would be working on at a given date. Hence, I started with the PDF, doing (almost) all the exercises after each module -especially the Extra Mile ones- and then switching over to labs once finished. The PDF was never closed though, even in the exam. So it can be considered as a good source for reference in the off chance you don't keep any notes in general. Also,

{% include tip.html content="Don't be intimidated by the sheer size of the PDF. Although it covers lots of things, the main reason it's too big is because of the gradual buildup of the final exploit for each module, going though each defence progressively, augmenting the final payload step-by-step, instead of handing it out for you to use. Beyond all these code blocks It's also full of images and references which are also very helpful but add up to the final page count. " %}


With regards to the age old question: *But I don't know programming, is C# knowledge a requirement?* 

The course doesn't teach C# the traditional "this is a variable, this is a for loop, this is a function" way. Instead, it gives you a working example (in whole, within the PDF, so read the PDF carefully!) to have something to ~~copy-paste~~ start with + some *very* important pointers. The rest can be figured out by the reader with a working brain and some basic programming background. Personally, I had a C/C++ background, comfortable with Python and Java, but knew no C# at all, and I don't think I now know any either, I'm just much more comfortable writing custom payloads for the specific purposes of my job. 

As stated above, the course is heavily Windows-focused and doesn't go deep into the Linux stuff in-scope, which left a bit of a bitter taste... I guess this is reasonable bias though considering that most corporate networks out there are mostly AD-based.  

I also expected a bit more detail in the promising Kiosk Breakouts section which focused heavily on browser escapes. So (not really a)  spoiler: You won't find Citrix stuff :frowning_face:. In all fairness though, a course can only teach you so much, and Offsec does a good job by providing the general methodologies of breakouts while staying product agnostic. 

A tip for time-management? Easy:

{% include tip.html content="No need to tactically break up with your significant other every Friday, just let them know beforehand that you'll be MIA on all evenings for the next 2 or 3 months risking the consequences. " %} 

  <img src="https://i.kym-cdn.com/entries/icons/original/000/022/138/highresrollsafe.jpg"   />



Seriously though, It is a demanding course and although different people will tackle it with different levels of ease, it will certainly take some serious effort and commitment to go through all the material AND the labs. I personally spent all weekday evenings (18.00-22.00) and almost full weekends on it, to achieve some momentum and focus, a lesson I learned from my OSCP endeavours. I signed up for 90 days because I was a noob and thought I had mountains to climb, and I was done with all the challenges and studying a full week or two before the end of it all. 

I scheduled my exam for a mid-week noon (12:00pm), put some holiday in to justifiably close Slack and work email for the next 3 days, made sure not to tell anyone about the exam to act like it never happened should I fail and had a good night's sleep. In the day of the exam, I woke up early, went over my notes one more time, cleaned my desk and laptop and was ready to go. 

<img src="https://media.giphy.com/media/BpGWitbFZflfSUYuZ9/giphy.gif"   />



## The Exam

{% include warning.html content="The section below is heavily redacted. However, if you want to totally avoid any spoilers whatsoever, feel fee to skip this section. On the other side, it's very likely that this is what you're here for, so proceed at your own risk!" %}

It all started off on the wrong foot. Despite joining 30 mins earlier than my scheduled start time, I didn't get through the procedural stuff untill 30 mins after that start time. Apparently my moving around Europe for jobs confused their system a bit and I had to show all sorts of documents on the camera or the screen. Props to the proctors though, for their patience, flexibility and co-operation. They didn't give me a hard time e.g. by asking for paper documents. 

Then it got even worse. I figured out what the attack vectors were very early on. Prepared and ready as I was, with all my payloads tested multiple times in the labs, against updated AVs, ready to be executed with minimal adjustments, something (not so) unexpected happened... I picked the one matching the scenario, modified it, sent it and boom. Nothing happened. A re-try in the test machine, with the AV enabled confirmed my hypothesis. It was being flagged. How? Offsec updates the signature databases of the AVs for test machines periodically so apparently script kiddies kept uploading similar types of payloads as my preped one to Virustotal, until they eventually got blacklisted. UUURGH

This wasn't even the first time this had happened throughout my 90 day engagement with all things OSEP, as another [very nifty tool](https://github.com/calebstewart/bypass-clm) I initially used in one challenge lab ended up being picked up soon afterwards... So here's one solid piece of advice you should already know:

{% include tip.html content="Please, please don't upload your payloads to Virustotal. And don't test them in an internet-connected machine either! If necessary, take the hard (and rewarding) way: Spawn a test VM with no internet access, install all/most prestigious AVs and test there instead. I didn't. But my technical debt *will* haunt me as (hopefully) one day I'll go through the same process for some job." %}

Anyway, back to our story. I tried lots of things to bypass it, including taking the alternative route for initial access. Oh yeah, by the way, the exam includes "multiple" distinct paths to get initial access and presumably different paths through the network from there (fun fact: One of them in my exam looked a lot like a rabbit hole). Deep down on my list was also a means of payload "obfuscation" we're taught in the course. But it was so low because: 

1. I didn't believe this is what led to the detection in first place (in my noob mind it was probably a more serious heuristic like a sus [parent-child relationship](https://labs.f-secure.com/blog/attack-detection-fundamentals-initial-access-lab-1/))

2. If so, I didn't believe a method like this would be effective against any counter-measures like e.g. API hooking in place. 

So the time is passing and I get more and more frustrated as it went, terrified by the more and more possible scenario of failure without even getting initial access :grimacing:. By the way, as you've probably read in more serious reviews elsewhere, I can confirm that a prepared "student" (to use an Offsec term), who manages to get Initial Access in a reasonable time window (the first 20 minutes of the exam) will probably get passing grade in 8 hours or so of net testing time. Which is a great thing given that the exam is scheduled for 48 hours  and we're all professionals with lives and jobs, even during a lockdown(ish) period. So around 9 hours after my start time, the fear was getting real and the disappointment was mounting... This is when I decided to give it a go, although not believing for a single moment that *this method* it would be enough. For extra leetness I took it one step further by taking care of strings and variables. Gave it another go in the test machine against its AV (which remember, I *could only assume* at that point would also be running on the target machine) and it was once again detected - of course. Surprisingly though, it was picked up as a different "Trojan" or "RAT" than the initially indicated... Weird, but let's try it anyway. Reverted the whole lab (not that this would help in detection - more like a standard step for this particular payload as I had figured out in the labs). Sent my payload. And voila, without making any sense whatsoever, it gave me a meterpreter shell back. 

<img src="https://media.giphy.com/media/l4Ep3mmmj7Bw3adWw/giphy.gif"   />

So a first implicit piece of advice everyone gives which is worth re-iterating:

{% include tip.html content="Revert often. You're given plenty of reverts for 48 hours, and some delivery methods specifically want a clean slate to work. So it never hurts waiting a bit more before hitting the Go button, just to save your future self from hours of debugging." %} 

Fun fact, I finished the exam with only just one more revert. Two in total.

However, a more important tip here is to just: 

{% include tip.html content="Trust the process. Do what the course teaches you to, patiently go through all the bypasses you've learned, be methodical and persistent, and the test lab will reward you." %}

With that out of the way, and after 3 hours of sleep to freshen things up, I set out to tackle the actual lab.



## The Actual Exam

After a couple of quick `proof.txt`'s I stumbled again, with no apparent way forward. The least technical solution ended up being the successful one, which involved using credentials in all possible ways. Manually. 

Actually, looking back now, a decent part of the exam (and challenges) would just boil down to this solution, just make sure you do proper post-exploitation enumeration:

<img src="https://www.thesecurityblogger.com/wp-content/uploads/2016/07/mimikatz_sticker.png"   />

There are always ways to automate this so don't be like me:

{% include tip.html content="Set up [crackmapexec](https://github.com/byt3bl33d3r/CrackMapExec) if you haven't, and familiarise yourself with it. I knew of it before but never actually sat down to properly use it until the end of the test. It will come in handy when you'll have lots and lots of credentials and several machines / ports to test them against." %}


Anything after that was pretty straightforward, exactly or almost exactly like in the challenges, especially the 3 final ones which kinda merge together several techniques in a progressively more realistic network, so make sure to go through them and pop all the boxes in each before taking the exam. I didn't even realise I got into the final machine as described in the Exam objective before I found `secret.txt`. 

With that out of the way and -*in theory*- a passing condition, I focused on two things for the remaining hours left:

1. Trying to pop a few more boxes to also achieve the second passing condition, 10 or more flags. - I was at 9 at this point. 
2. Convert an RDP session that got me into one of the boxes along the way to a "fully interactive remote" alternative to comply with the rules

About the former, I didn't manage to get a final tenth box, which however further proves that passing the exam is possible by achieving just one of the objectives. As was described in the guide. 

About the latter, If you're taking the course after March 2021 you'll notice a new [red annotation](https://help.offensive-security.com/hc/en-us/search?category=360004329172&filter_by=knowledge_base&query=remote+interactive+shell&utf8=%E2%9C%93) in the official Exam Guide. Thanks to an ex-colleague (Martin, cheers if you're reading this! :wave:), they've now clarified in the rules that RDPing into a machine with a backdoor user, to take the screenshot of the flag <u>is not accepted</u>, and will lead to 0 points out of the 10 for the box. Without giving too many details away, I can state that it's (apparently) acceptable to RDP in and then "misconfigure" the machine on purpose to go in the right way before `type`ing the flag for the screenshot :man_shrugging:  

## Epilogue

Having gone through it all, I still have nothing but the best to say about the whole experience. I greatly enjoyed the course, refined my skills and increased my knowledge on important areas.

Sooo this is it. Hopefully this was a pleasant read and/or will help you decide whether this is or isn't for you. Alternatively, I hope this provided the necessary insight to feel mentally prepared ahead of your exam and manage your expectations. 

Thanks for reading!
