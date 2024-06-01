---
layout: post
title: "A Hacker's guide to OSDA"
excerpt: "While scrolling my timeline on some social medium one day quite some time ago, I bumped into a post by (then) Offensive Security advertising a new version of Kali called 'Kali Purple'. I watched the <a href='https://youtu.be/3UMxOsdexK8'>linked</a> video and realised it was actually a plug for their new course, 'SOC-200' targeting... blue teamers! OffSec doing DefSec?, I thought, that would be interesting... 
<br/><br/>
Many moons later, when the itch for a new challenge reemerged, I came to the conclusion that some technical upskilling in defensive ops would probably be more beneficial than an offensive one (you're next, CRTO) given the amount of time I was now spending with client SOCs. After some discussions with the bosses, I was given the thumbs up for this new and unknown OffSec offering, along with an extra request to note down some thoughts if I were to take it, in case others would be interested in the future but wanted an extra view.
<br/><br/>
So in May 2023, when the first gap in schedule appeared, I activated my 3-month lab access immediately. A large-scale Windows-focused Purple team had just been scheduled for late July and in the back of my mind this posed as a deadline, as it would require all the prep and upskilling I could get. 
<br/><br/>
And so the adventure began. After exactly 2 months and 2 days, I passed the exam and got my third shiny OffSec cert (figuratively, as they sadly no longer do paper certs). Once the obligatory celebrations were out' the way, it was time to write it all down.<br/><br/>"
categories:
  - "blog-posts"
tags:
  - osda
  - offsec
  - "purple team"
  - "soc-200"
last_modified_at: 2024-06-01T17:07:00
post-img: assets/img/offsec-newlogo.png
---

{% include toc.html %}

{% include note.html title="Throwback:" content="This article contains references and generally adheres to the same format as the similarly titled <a href='/blog-posts/2021/04/29/noobs-guide-to-osep.html'>A noob's guide to OSEP</a> from  2021 " %}  


## Preamble 

While scrolling my timeline on some social medium one day quite some time ago, I bumped into a post by Offensive Security (prior to the rename/rebrand) advertising a new version of Kali called "Kali Purple". I watched the [linked video](https://youtu.be/3UMxOsdexK8) and realised it was actually a plug for their new course, "SOC-200" targeting... blue teamers! OffSec doing DefSec?, I thought, that would be interesting... 

Many moons later, when the itch for a new challenge reemerged, I came to the conclusion that some technical upskilling in defensive ops would probably be more beneficial than an offensive one (you're next, CRTO) given the amount of time I was now spending with client SOCs. After some discussions with the bosses, I was given the thumbs up for this new and unknown OffSec offering, along with an extra request to note down some thoughts if I were to take it, in case others would be interested in the future but wanted an extra view.

So in May 2023, when the first gap in schedule appeared, I activated my 3-month lab access immediately. A large-scale Windows-focused Purple team had just been scheduled for late July and in the back of my mind this posed as a deadline, as it would require all the prep and upskilling I could get. 

And so the adventure began. After exactly 2 months and 2 days, I passed the exam and got my third shiny OffSec cert (figuratively, as they sadly no longer do paper certs). Once the obligatory celebrations were out' the way, it was time to write it all down. Well, not *everything*, as that would violate OffSec's rules. So we'll have to do with a heavily redacted version. With this disclaimer out of the way, let's dive right in.

## The Course

The first challenge was essentially finding where the actual course material is. :facepalm:

A lot of things had changed since 2021. Other than the new logo & colour scheme, the course material is no longer bundled solely in the mega PDF. Nowadays, starting your course doesn't give you access to the Document + lab portal + Videos.zip triplet, but a set of course-specific pages, including the material in HTML & the standard videos - now mostly narrated by a female voice without the chilling, legendary "excellent" exclamations 😭

### A Sour First Taste

Overall, the materials were rough around the edges, and were giving out a "Beta version" feeling. More often than not, they were all over the place:
- From discrepancies between the order of sections in the PDF and the reader...
  ![Content Discrepancy](/assets/img/discrepancy.png)
- ...to the In-browser Kali instance not working...
  ![No Kali](/assets/img/no-kali-instance.png)
- ...to missing scripts on exercise boxes 
- ...and the most frustrating: It took me more than a week to figure out where I could launch the exercise boxes from, before finding the "Start" button buried in the HTML
  ![Exercises](/assets/img/exercises.png)

The actual content focuses on the fundamental security monitoring technologies & concepts for each prevalent attack surface ([syllabus](https://www.offsec.com/courses/soc-200/download/syllabus)):

For **Windows**, it covered Windows Events and Sysmon. A lot of Sysmon. Writing PowerShell scripts from scratch to access & browse events programmatically. The relevant detection engineering exercises zoomed in on the artifacts generated by Privesc/Persistence TTPs and became somewhat monotonous after a few iterations. Note that the attack execution was all prep'ed up in ready-to-run scripts or binaries, de-emphasising offensive tradecraft. 

{% include tip.html title="Fun Fact:" content="OffSec alumni might recognise that most of the attack scripts are actually those used in the PEN-300 course, sometimes with the exact same names and matching timestamps. " %}

Despite the detection analysis methodology becoming straightforward quickly, I found more enjoyable the act of re-discovering well-known TTPs and getting proficient with important well-known Event IDs. 

When it comes to **Linux**, it covered the systemd/journald framework and then lightly touched on the audit framework (auditd). Once again, the course was weak on Linux coverage compared to Windows. However I didn't particularly mind that, given the dominance of Windows in corporate settings, which in fact kind of justifies OffSec's decision. It felt, initially, that auditd should be covered in depth, given its capabilities. Personally, I can't recall an organisation I've worked with that didn't use auditd on the Linux estate, so I expected at least a primer on its features, configuration, some queries maybe and why not internals. Eventually, it turned out some of these were in fact needed for the challenges, and you have to study & find out yourself. Give a man a fish, as they say....

Differentiating those two sections as endpoint-specific, there followed a section about **servers** where prominent software (Apache, IIS etc) and their logging capabilities were being discussed. Then, touching on **network** monitoring, the course walked though the simple yet popular Snort IDS/IPS. Things got suddenly more interesting as it got into very specific detections for Zerologon and Empire's C2.

Other than this introduction to the fundamentals, there's not much to say about those 16/18 chapters. Certain parts were ~~a bit pointless~~ clearly aimed at students very new to infosec or IT in general. For example: 
- one exercise basically required writing a Python script for what a simple `grep` command would do
- a video about Windows Lateral Movement where the only thing happening was an `mstsc.exe` command being executed with the takeaway being "if connections come from multiple places, we should investigate" 

*There is one particular exception to this monotony though...*

### The Real Value

Somewhere in this heap, breadcrumbs of other SOC processes and theoretical concepts were thrown into the mix, such as a subtle reference to the **Pyramid of Pain**. I bet that for most offensive practitioners like me, this concept would serve as the first real introduction to Detection Engineering. Let me expand on why I found this very valuable: 

Prior to SOC-200, my exposure to defensive ops was minimal. Despite having worked in an actual SOC for a brief period (though arguably not a well functioning one) and having tried a few half-baked blue/purple-team focused courses, none of those *really* laid any theoretic groundwork. Top-down frameworks, if you prefer, in which all the bits and bobs you pick up along the way would fit into. The challenges defenders face are not new and I always thought there must be a body of knowledge out there somewhere, that each newcomer goes through. Maybe in boring classrooms or introductory book chapters that noone reads. My gut feeling was that this was the stuff of courses like SANS, but those are not only ultra-expensive but also not best suited for offensive security practitioners and the needs of *our* jobs. A much better fit for this demographic, I've come to believe, is instead OffSec's SOC-200 course. 

Digression Over - back to the course material. After those 16 chapters, finally the Elastic stack (formerly, ELK stack) was introduced. There, things got a lot more exciting, and log querying was now rapidly faster and easier than the clunky and cumbersome PowerShell or Python scripts, which you had to run *on each host* til then. For a moment I questioned why all those previous chapters were even necessary, but quickly realised that the value was to get hands-on at a low level, to look at data sources directly and re-invent the wheel. And this is a truly valuable baby step before one jumps all the way up to the god-mode, all-observing, eagle-eye capabilities of a modern SIEM.  

With all the fundamentals introduced and with scaled-up monitoring tools at our disposal, attacks increased in complexity and encompassed all surfaces. Always opaque to the student who needn't worry about implants not pinging back or payloads not getting past Defender. Instead one would only have to push a button on this amazing contraption called "Hacker Simulator", which was exactly what it sounds like: A lazy one-click trigger of pre-scripted attacks:

![Hack9000](/assets/img/hacker-sim.png)

Other exciting moments within that last chapter was the rundown of the basic ELK features a SOC analyst should be familiar with, including of course alert rules. And how to write simple query ones but also slightly more complex correlation or threshold ones. 

![Florian Respect](/assets/img/roth.png)

## The Labs

Once the course material was done with, it was time for some real learning. Ahead were 12 labs of increasing size and complexity, deployable and restartable on-demand with just one click on the portal. All mine, with no interference from other players. Which might sound weird to readers new to OffSec but only a few years back this luxury was unheard of. And if you're keen to peek behind the curtain, you might just find that it's all deployed with Kubernetes and maybe get your hands on the scripts & tools used to "install" the vulnerabilities onto the environment, but this is of course highly illegal stuff and trespassers will be prosecuted.

![Peter Parker Being Blind](/assets/img/parker.jpg)

In terms of pace and progress, I had opted for the 90-day lab access and the course took around a month to finish with moderate study. Everyone has their own pace but for reference, I was spending around three or four hours a day, and more or less same on weekends. The exam I had already scheduled for July 16th, right on the 60-day mark after course activation (and very close to my birthday). Rushed, but the idea was to have the knowledge and the cert-holder's boost before that pivotal job in the end of the month. As a result, coming back from holidays on July 1st and with just 16 days to tackle 12 labs, ~~panic started kicking in~~ the studying intensified.   

Fortunately, before jumping into the labs, I had caught wind of the OffSec Discord servers and had a scroll through the forums, finding what turned out to be two highly important resources for the challenges: **Gervin's videos** and the **hints bot**. Apparently OffSec ([@_anans3](https://twitter.com/_anans3)) delivered OSDA training a while back and the 10-or-so recordings of which were uploaded on the portal, under a different "Course Name" ("OSA-SOC-200") as one-hour videos. This is truly a great resource, although deliberately incomplete at times, as it only covers specific areas of interest for each lab. Additionally, once enrolled into the right Discord channels, one can make use of [a chat bot](https://help.offsec.com/hc/en-us/articles/360049069012-OffSec-Community-Chat-User-Guide#h_01H64753PR7EVT58SR9922MKMZ) specifically created to provide hints. These can serve as a sanity check that you haven't missed anything in each attack phase. 

Like in the exam, each lab consisted of a number of distinct attack "phases" sequentially launched on demand via the click of a button on the portal, resulting in the execution of one or more observable attacker actions. So the flow I was following for every lab, was more or less the following: 
1. Run each phase, 
2. Observe, investigate & document - taking however long it would take
3. Get the hint before moving to the next phase - to confirm observations and ensure nothing was missed
4. Hit the "Next Phase" button *and wait up to 10' before jumping into ELK* - this is important, as a lot of things can be missed due to the brief period it took for data to be fully ingested and indexed
5. Repeat from 2
6. Once the lab is finished, watch the OSA video - to confirm & verify 
7. If stuck or can't find anything, a good tip I read somewhere was to move on to the next phase anyway, and any unanswered questions would be solved later down the line. And that worked, not only in the labs but in the Exam too. 
 
Are all these confirmations necessary, you might ask? I soon figured they were. 

### The True Challenge

Like with every OffSec course, the labs is where the real self-learning begins. Where the "Try Harder" saying becomes a practice, through the problems thrown at you to tackle, with minimal preparation preceding. In this case of this course, things got tougher in the following ways: 
- Within the sea of logs from e.g. appsrv3, browsing through all events manually meant it was easier to miss attacker actions visible in only 2 events 
- There's no flag to concretely let you know that "you've passed", and instead I would sometimes get the right answer to the "what happened" question from the bot (!), after completing the whole lab with a false sense of success 
- Any alerts I would write and test in one lab would never fire in another - for reasons unknown to this day 
- Researching all those random events seen in the chaos of logs would have me bumping into aaaall those forums where panicked IT admins would desperately ask "is it malware?" for every piece of unheard .exe you can think of, only to realise that I was basically in the same boat
- It was harder than anticipated to map logs generated back to the attack tools used, even when these tools woulwere well-known and non-evasive post-reveal. Finding out eventually that e.g. the complex sequence of logins caused by the well-known and noisy `impacket-psexec`, would destroy my confidence. Without any guidance or knowing where to look for tool indicators, even a simple shellshock payload from exploit-db wasn't that obvious to spot from the other side of the lens... Yes blue teamers, I felt your pain. 

This last one puzzled me a bit more, as a question I kept pondering on more and more as I was closing in on the exam was "what level of reporting detail is considered good enough"?  And subsequent questions such as "how many logons does a  `wmiexec` operation result in?", so that I'm able to tie X logons back to this particular tool (spoiler alert: a lot more than one, all with very subtle differences) or "is there evidence I should be finding to separate between an SMB login and an RDP login?" (NB: there's not. Not from an endpoint-only perspective - IR bros please correct me here if you're reading this).  If the concern is not clear by now, think only that OSCP and OSEP required developing attacking tradecraft in order to pass, while here the attack would happen anyway once the button would be pressed, the evidence were there in the SIEM for anyone to see. How many events are enough to copy-paste into your report before you could claim you've "proved what happened"?    

### A Question of Time

Speaking of reporting, this course had an interesting relationship with time, stemming from the fact that it was hands-on and reproducible... Let me explain:  

From my limited interaction with blue teams, in the real world, the only constant is time. Hence, time can be used to frame all the pieces of an incident around a unique and unanimous arch. Behold, the most intuitive story-telling tool: *The timeline*. In the SOC-200 labs & exam however, time was controlled by the student! Phase 3 of an attack could as well be observed *before* Phase 2 on wall clock time, if for some reason you would choose to revert, should e.g. something fail (please keep reading for more on this paradox). As a result, you could easily end up with a report including timestamped events indicating first initial access, then a persistence action needing SYSTEM and finally a privesc operation to SYSTEM!!? And this is why I believe OffSec never & nowhere suggests structuring your report based on timestamps. For OSDA, time is not linear. 

Speaking of time, prepare for frustration when it won't be possible to change the SIEM's timezone to GMT, and you'll have to offset all events seen from US Pacific time. Even after setting your timezone in the lab and everywhere as advised.

In the nick of time before the exam (bad pun, I admit) and in the last OSA video. Gervin, as promised, guides you through the creation of ELK dashboards. A feature briefly touched upon in the SIEM introduction chapters but yet so powerful. That's because a dashboard can highlight spikes or anomalies "at a glance", especially when all else you have is failing alerts. The dashboards I came up with were nothing fancy, but for the log sources and attacks at play, they proved super valuable. 

  ![Dashboard Pic 1](/assets/img/dashboard.png)
  ![Dashboard Pic 2](/assets/img/dashboard2.png)

Before going any further, it's worth mentioning that the lab (& exam) rules explicitly prohibit logging into hosts directly for analysis. Instead, you're restricted to monitoring only from the SIEM - but this is not to say you only have passive investigation capabilities. Using this as a good use case for sneaking [OSQuery](https://www.osquery.io) into the course material, the relevant ELK plugin is introduced and students are granted a valuable active investigation tool, through the myriads of databases modelling system state in all aspects. The power of OSQuery is hinted upon through Gervin's videos too, with a particular example being the point where he spotted a C2 channel be established, using OSQuery only (active network connections), due to the lack of other network monitoring/deep process inspection solutions in the environment.  

## The Exam

Having secured a comfortable Sunday afternoon slot to start the 24-hour marathon, I took the Monday off in advance. Mostly for contingency, as I felt pretty confident that I'd be done with both the exam & reporting by Sunday. Of course things didn't really turn out that way. 

Staying true to OffSec tradition, the exam was hard, and I admit I had underestimated it: 
- The attacks were more advanced. Most operations were now executed in-memory instead of through dropped binaries like in the labs. This required you look into all the different log sources to figure out what happened, rather than just simply seeing it in process creation events 
- The log volume & lab network size was *substantially larger*. With 13 boxes in the exam - 7 more than the largest lab thus far - you had to deal with around 15k log messages generated within a 10 minute period. Needless to say that my silly labs approach of scrolling through each and every event like a mad man, would obviously not work here 
- The overall attack chain(s) spanned across an exhausting number of 10 phases! 4 more than the largest lab. 
- The hefty little diagram that you were provided for every challenge lab and depicted the hosts involved & network zones, was not given here. Not sure if that was just to make things harder or to simulate a blue team not knowing their own network. The ASM hype probably suggests that inventory management is hard.     

But on the plus side of things, the dashboards I imported came in handy and the small set of alerts pre-populated by OffSec in the exam's ELK stack provided a good place to start in most cases. As well as few rabbit holes. Other instances of usual OffSec trickery included an attacker action which although seen in its entirety on the recorded PowerShell script block logs (e.g. `New-Service`), it had actually failed to run, probably due to access rights. And this failure was only evident through the lack of the relevant events (no service installation event 4697) and a runtime listing of services through OSQuery - stuff that would result in dropped points,  had I stuck with the PS code and not investigated further.
  
As a result of all these, I was moving through the Phases a lot slower than what I was used to in the labs, reaching the halfway mark after 9 hours - and then disaster struck.  

### System Failure

After soldiering along through these first five phases, the glorified hacker simulator broke down. And naturally, for a moment or two, I was feeling the heat.

![Uncertainty Strikes](/assets/img/notsure.jpg)

Phase 6/10 ran but there was nothing to be found, other than minimal synthetic noise which -in the absence of anything more useful-  could be misinterpreted as subtle IoCs. As per the protocol, I thought I'd move on to the next phase anyway, but the situation was similar for both the seventh and eight phase triggered. At which point I raised it with the OffSec proctors who in turn looped in Support, saving me from the hassle of emailing / raising tickets etc. Luckily, this scenario was something I had read about in one of the reviews out there before taking the exam. Otherwise, who knows what I would've documented as "attack activity".

After an hour or so of twiddling my thumbs, OffSec confirmed it was indeed broken and started redeploying everything. Remember that each phase should be left to run for at least 10', hence if I was to redeploy it all myself I would have to go through a very frustrating extra hour. Thankfully, they offered to do it for me, and with the full view behind the scenes they had, it all finished a lot faster than that. At this point, I asked for my lab time to be extended by a fair two hours for compensation, but they were laaaarrge, and -as per the rules, apparently- they instead gave me six and a half (!) more hours, as an apology. Fair play OffSec! 

Regardless, my timestamps (and nerves) were all over the place after this. It took me one more hour to recover, and even with the lab in the expected state, connecting previous actions from when they were first reported to the current (post-redeployment) state was very very confusing. Think of a registry key documented as being set at 03:41 by PID 6679, and then a few hours later, scrolling through the SIEM events to see it being set at 04:20 by PID 2841. So much for keeping it real. 

So, given this extension, I ended up finishing with phase 10 after 18 hours in total, including around two hours of cumulative break time for naps, sustenance, and leg-stretching. That left me 36 hours for reporting which I couldn't of course use in full, because of work. One looong, stress-free, refreshing sleep break later, I powered through the final 10-hour sprint with hardcore reporting, documenting everything perfectly while the lab was still accessible and all the events still saved in ELK. It shouldn't be a pro tip but this ended up saving my A$$, as having the option to run a few more queries while going through my findings unearthed some crucial inaccuracies that would have otherwise cost points.          

### Baby's First IR Report

Having made such an effort on documentation, I want to talk a bit about the final report, as it was the first of this kind I had ever created, in any context. Combined with the perfectionism which had started kicking in, I felt it was necessary to come up with a format solid enough that would match a professional IR report - despite having never seen any. Effectively, the challenge here was to make it all look a bit less like this: 

![FBItime](/assets/img/paranoia.png)

From OffSec's perspective, there's no guidance or requirement as per the format selected, so long as it's "structured". And at least they give you the Word template to get started. I ignored all that and went with good ol' markdown (Obsidian ftw) as with all my OffSec reports. 

Using the fact that no network diagram was provided as an opportunity, I put my expert PowerPoint skills to use - even branding it with fancy WithSecure icons being a professional report and all. The network zoning I inferred from the IP addresses & network masks, which could be retrieved from OSQueries, as well as the Operating Systems. 

{% include warning.html title="Sorry!" content="There used to be a very nice diagram here, but it was too accurate to include. So unfortunately it didn't pass the censoring board" %}

Depicting it like this was actually very useful for me as well, being the visual type, to understand the progression of the attack while working through the report. Hence, I kept updating it throughout the phases, to indicate the network's status following each attack step. 

Despite the lab revert messing up the timeframes big time, I felt that a master timeline of key events was necessary, in the initial section, before drilling deeper into each phase's analysis. A REDACTED snip of the timeline I provided is shown below: 

![Timeline](/assets/img/tl-key.png)

For a few minutes, I also explored options to visualise this timeline and came up with the following Mermaid graph, before succumbing to the exhaustion.

![Timeline but Visual](/assets/img/timeline.png)

With some extra CSS and some experimentation with Mermaid attributes, you could space the boxes so that they suggest an actual chronological order, but after 30 hours of work the brain was well overcooked for that.   

Worth repeating here that aaall those visuals were of course completely unnecessary. Along with the increased amount of detail -just to be on the safe side- the report was inflated to a whopping 70 pages. 

## Conclusion / Afterthoughts 

Summarising, OSDA is a journey I very much enjoyed and would highly recommend. 

This challenge was among other things an ideal opportunity to get familiar with defensive technologies and concepts, to learn the basics around detection engineering, to spend some hands-on time in a SIEM and learn the fundamentals. All, in a controlled environment, free from the pressure of a client-facing job (should I had opted to instead wing it and "learn on the job"), but at the same time through a structured and guided way.   

I did mention in the beginning that job which played a part in expediting this journey. Looking back on this decision, it was well worth the extra push. If it can somehow be quantified, I'd say OSDA provided around 50% of the technical expertise I used in that project, especially when onsite. I was surprised to see myself spotting mistakes in alert queries written by professional SOC analysts and quoting Windows Event IDs from memory. Or describing the ins and outs of Kerberoasting and why it's a tricky TTP to detect, before coming up with four different methods on the fly for how to approach it. 

All in all, this cert might sound like a tongue-in-cheek move by OffSec, or an attempt to expand their customer base to more defenders. But in my humble opinion it's a unique option at the moment, due to the industry-acclaimed offensive mindset OffSec have built throughout the years. And don't let the 200 grading fool you, it's a tough cert, but it will reward you, one way or another: 

![Try Hard Coins](/assets/img/Inkedtryhardcoins-hl.jpg)

Stay safe! 

