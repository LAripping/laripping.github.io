---
layout: post
title: 'Taking On The Cloud Resumé Challenge'
excerpt_separator: "<!--more-->"
excerpt: "I strongly believe that to learn is to do. And having the longing itch to cover my deficiency in cloud security and cloud in general, it came as a big A-Ha moment when I read about <a href='https://cloudresumechallenge.dev/'>the Cloud Resume Challenge</a> in Nick Jones' awesome &quot;Breaking Into Cloud&quot; checklist.<br/><br/>Essentially, the Cloud Resume Challenge is to create a simple web application, but fully cloud-native and using modern DevOps practices. The app will be a statically hosted website showing your Resumé (or just Resume from here onwards, might also use the term CV interchangeably) and will include some backend functionality in the form of a visitor counter. All of this will be created through 15 well-defined tasks.<br/><br/>" 
categories:
  - "blog-posts"
tags:
  - cloud
  - aws
  - python
  - serverless
  - devops
last_modified_at: 2022-08-25T17:19:00
post-img: assets/img/high-level-design-expanded.png
---

{% include toc.html %}


## What is the Cloud Resume Challenge?

I strongly believe that to learn is to do. And having the longing itch to cover my deficiency in cloud security and cloud in general, it came as a big A-Ha moment when I read about [the Cloud Resume Challenge](https://cloudresumechallenge.dev/) in Nick Jones' awesome "Breaking Into Cloud" checklist.


<blockquote class="twitter-tweet">
  <p lang="en" dir="ltr">How to get into cloud security based on my own experiences, and on mentoring and hiring over the last few years. I&#39;ve focused on how to make yourself a success in the field, rather than just the technical knowledge required: <a href="https://t.co/oEhrYHCVda">https://t.co/oEhrYHCVda</a>
  </p>
  &mdash; Nick Jones (@nojonesuk) <a href="https://twitter.com/nojonesuk/status/1531336352972824580?ref_src=twsrc%5Etfw">May 30, 2022</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
<br/>


Essentially, the Cloud Resume Challenge was to create a simple web application, but fully cloud-native and using modern DevOps practices. All three public cloud providers were supported but I picked the AWS flavour, hence this post will only refer to Amazon clouds. The app was supposed to be a static website showing my Resumé (or just Resume from here onwards, might also use the term CV interchangeably) and had to include some backend functionality in the form of a visitor counter. Everything created through 16 well-defined tasks because who doesn't love checklists?   

![](/assets/img/crc-list-logo.png)

The end result, my very own cloud-hosted Resume can be observed at [https://resume.laripping.com](https://resume.laripping.com).  

As part of the last step, I had to write a blog post about my journey to document what I learned and share my thoughts about it, which brings us right here. So let's get to it!


## Why Take It

Having a public CV in a nice format which I'd have total control over, instead of being limited to the confines of a LinkedIn profile, was something I've been pondering for a while. The challenge was hence a golden opportunity to kill two birds with one stone. 

Moreover, if familiarisation with cloud is one side of the challenge, doing it the DevOps way is the other. The course therefore takes good care to steer you towards concepts like Infrastructure as Code (IaC), Continuous Integration and Delivery (CI/CD) while also covering important aspects of the DevOps movement like a strong unit Testing regime. Although these conceps are nowadays more ubiquitous than ever, I had never *deliberately* practised them, so once more the challenge was the best time and place to do so. 

For example, I find it fascinating that my cloud resume app is now in a state that both the frontend and the backend can be updated in seconds, reliably and securely. Free of traditional burdens like signing with a hosting provider, buying space in a web server, SSH-ing into boxes using all sorts of personally maintained credentials where architecture components are tracked in READMEs and one simple Syntax Error is enough for everything to fall apart. Instead the process now looks more like this: 

1. A `git clone` to fetch the code of both the content AND the infra 
2. Easy setup of a disposable dev env locally
3. Experimental changes can be introduced with a good-old `git commit & git push` and rolled back to a safe state upon encountering any errors
4. By putting every little change through fine-grained Tests automatically, where the necessary credentials/secrets are equiped from safe places 
5. Good code is re-deployed on a global scale, refreshing caches, available instantly


This whole process happens **in seconds** allowing rapid experimentation. In fact, take another look at the challenge steps and you'll see that they're laid out in a way that requires this process to be performed at least once for each aspect (frontend/backend), subtly forcing you to get it right. Genius!  


## Challenge Overview

So I embarked on the journey, which lasted around 3 months, ended up using around 50 AWS resources in total, and it all cost me around a penny, as the usage per each AWS service utilised was well within the [AWS Free Tier](https://aws.amazon.com/free/) ranges. The only real fee that was expected for the application is for the domain name, but as something I would keep, I chose to host it under as subdomain of a domain name I already had.   

![](/assets/img/cost.png)
*&pound;0 cost for the 3 months of experimentation confirmed through the AWS Cost Explorer service*

To track the resource count I started tagging everything I would create through the console with `Project: cloud-resume`. Initially I thought this way would make it easy to tear it all down when finished, by deleting all tagged resources at once through the [Resource Groups](https://docs.aws.amazon.com/ARG/latest/userguide/resource-groups.html) service. While it seemed like a good idea, later down the road this tagging approach turned out to be rather naive as things were being created automatically as part of "Stacks", and any tags could only be applied to the resource container. 

Even from the early steps it became apparent that the challenge was quite popular, as things I would Google would lead to many results some of them looking suspiciously similar to my problem! It was therefore tricky -but ultimate possible!- to avoid any spoilers from places like Stack Overflow questions or blog posts to both veer away from "shortcuts" that would hinder my learning, and also to keep my cloud resume 100% original and true to my authentic design. 

<!-- Another fun fact is that throughout the entire journey, with all the googling involved, I tried to avoid any places like Stack Overflow questions or blog posts that seemed too similar to my problem, as these could come from other challenge takers'. The point was not only to veer away from any "shortcuts" that would hinder my learning, but to also keep my Cloud Resume 100% true to the intended plan. The internet is a helpful place, but it's doable.  -->

Looking at the challenge from the viewpoint of the security practitioner, the **security** aspect or "Priority Zero" as Amazonians refer to it was one of main focal points throughout the journey. This was after all a rare occasion where I was building something from the ground up, something that would sit "in production", out there for the world to see. The [AWS shared responsibility model](https://aws.amazon.com/compliance/shared-responsibility-model/) was there to remind me, like any cloud user, that some of the risk involved was assigned to me, not entirely a concern of the traditional hosting provider, and that the fancy pay-for-what-you-use pricing model could also be turned against me in the case of malicious use - if not addressed appropriately.

![](https://pbs.twimg.com/media/FYPaHC-WQAIwfDE?format=jpg&name=900x900)
*Image courtesy of [@fullStackRacc](https://twitter.com/fullStackRacc/status/1550322516115218433)*

As such I wanted to summarise some thoughts on the actions I took towards a basic security model of the entire solution as a whole, and touch on the related AWS services involved whether visible or operating behind the scenes:
<!-- 
As such, I wanted to summarise some thoughts about the security of the solution. The actions I took towards a basic security model, touch on the related AWS services coming into play whether visibly or behind the scenes: -->

{% include crc-table.html %}



<!-- 

* IAM: Principle of least privilege. To nail this, I had the privilege of trial-and-error to find manually the absolute minimum number of IAM perms needed by each entity to perform it's purpose, like the ([auto-generated](https://github.com/LAripping/cloud-resume-backend/blob/f442e6341d7a8d87934d585868f2c2502af0ea32/template.yaml#L24)) `FetchVisitorsFunctionRole` which only needs the ` dynamodb:PutItem` and `dynamodb:Scan` permissions against only our table, to allow the backend code to talk with the DB.... Other times the least necessary policies is known in advance, like for my `sam-cli-user` user, which the SAM CLI uses when building the backend, thus only needing [specific permissions](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-permissions.html#sam-permissions-managed-policies). 
 -->
  



<!-- ## Stops Along the Way -->

In the next sections I'll document the lessons learned for each logical group of challenge steps and discuss any business decisions taken. 

<!-- Roughly grouped by the aspect of the application that the relevant steps covered, these might come in handy to any inspired readers that will go ahead and try the challenge themselves. -->



### The Frontend - HTML / CSS / JS

In the early steps (2,3 and 7) I went well overboard and snatched the opportunity to make my web Resume look exactly like what I had it in mind. Namely, I added: 

- a "Download as PDF" button (which required some research but turned out good enough) 
- with a basic spinner while the generation is cooking
- and a sticky sidebar to host all the buttons featuring Font Awesome icons

All totally unnecessary for the challenge itself but much fun to experiment with!  



### The Stage - DNS / HTTPS / CDN

When the time had come to focus on the internet infrastructure (steps 5 and 6), things got a bit blurry and it took some digging to figure out where each component would fit in. Complicating factors were that I was bringing-my-own DNS provider (Namecheap.com) instead of using the proposed Route53, and had decided to use a subdomain instead of an apex one. At the same time, both HTTPS and Amazon's CloudFront CDN had to be somehow involved.  

<!-- When it all started taking shape, the picture was much clearer  -->
It was a necessary entaglement however, as it closely simulated typical production environments I had seen, with complex topologies of load balancers, caching servers and global CDNs. Building an example such topology from the ground up certainly helped understand the why's and how's.

![](/assets/img/mermaid-frontend.png)
*Sketch from my notes when I finally put the pieces together*


Additionally, it was very interesting to finally see how **Domain Name validation** is performed when issuing an HTTPS cert. A process which, as we were taught at school, is very important for the fragile web-of-trust of the Internet as we know it, yet one I had never actually gone through myself and was therefore curious about. In the age of 2-click Let's Encrypt certs, this process would always somehow end up being abstracted from the user, to enable large-scale adoption.  

So when this moment finally came, [AWS Certificate Manager (ACM)](https://aws.amazon.com/certificate-manager/) -the relevant service for the purpose- offered two ways to do validation, via Email or DNS. While the Email way was fairly striaghtforward, expected and traditional, the DNS way -performed [via specially configured CNAME records](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html)- absolutely blew my mind as a genius solution for the problem! After some trial and error (and annoying waits for global DNS infra to catch-up) I had my shinny new cert:

![](/assets/img/cert.png)
*"It even has a watermark"<br/>— Patrick Bateman, American Psycho (2000)*



### The Backend - Lambda / DynamoDB / API gateway

<!-- ![](/assets/img/mermaid-api.png) -->

Here things made much more sense as the topology proposed by the challenge for the application's backend (steps 8,9 and 10) was actually fairly common, if not recommended even. As the Lambda-DyamoDB-API Gateway combo was simple, scalable and serverless!  



![](/assets/img/aws-flow.png)
*It's right there on the Lambda Service "Getting Started" page*


Probably the most important decision of the entire challenge, for the Python function step I had to decide how to "count visitors". The challenge description was coarse enough to allow for creative interpretations of how a unique session could be counted from a server.  The definition I came up with was the following:   

`Visitor = UA || IP`  

Essentially, to count as a different "visitor" each new combination of User-Agent and IPv4 address where the request hitting my API was coming from. The Python code of the Lambda function would have access to that request to extract these details.

After this decision, the rest of the application's backend was built from the inside-out. Starting from the simple, 2-column NoSQL database, then figuring out the exact database queries needed to perform the ops needed through the AWS CLI, and translating those queries into Python using the [boto3](https://aws.amazon.com/sdk-for-python/) SDK for the Lambda function. 

But where to write this Python code for the function? For several reasons like IDE sugar, Test-integration, versioning plugins etc. the Management Console's built-in editor ([AWS Cloud9](https://aws.amazon.com/cloud9/)) didn't seem the right option, yet no hint was given by the challenge on how to do this right... After pondering around trying to stitch-together a flow using the ["AWS Toolkit" PyCharm plugin](https://aws.amazon.com/pycharm/) and local containers I gave up and decided to do it the wrong way, writing the prototype Python code on-the-fly via the web editor. What was really missing was revealed later down the road, as the glue between Development and Deployment would be the **SAM CLI**, covered a bit more in the next section. 

Finally, I designed the simple 1-endpoint API as "the contract" that both the backend and the frontend would have to abide by:   


![](https://github.com/LAripping/cloud-resume-backend/raw/f442e6341d7a8d87934d585868f2c2502af0ea32/apispec-expand.png)
*Explore the API specification [here](https://petstore.swagger.io/?url=https://raw.githubusercontent.com/LAripping/cloud-resume-backend/master/apispec.yml)*



###   DevOps! - SAM / Github Actions / Tests


Having manually written a prototype for both the frontend and backend, the challenge for the next and final group of steps (11,12,13 and 14) was to re-write it all to make this whole process automatic, speeding deployments and enabling rapid experimentation. For this, the challenge suggested I use SAM, or [Serverlesss Application Model](https://aws.amazon.com/serverless/sam/), and the corresponding CLI. I learned that SAM is a concept allowing for serverless cloud infrastructure to be described in a code-like YAML format, which is well-defined, can be versioned, collaborated on etc etc. More importantly though, the SAM CLI could simplify operations against this codebase via simple commands like `sam init && sam deploy`. A kind of flexibility that its precursor, the [AWS CloudFormation](https://aws.amazon.com/cloudformation/) service, accessed through the traditional AWS CLI calls, could not offer to developers.  

The road here presented a fork, where I would have to either:

1. re-write everything built so far as YAML code to form a SAM stack, or
2. import these precious but rigid resources into an empty new stack

The lesson was only learned here after initially trying to cheat through option #2. I quickly figured out that importing existing CloudFormation resources in stacks generated and managed through SAM is a massive pain and probably not even doable at all. 

{% include note.html title="Note:" content="Thinking back, this could as well be a deliberate design by Amazon to force you down the right path ;) " %}  

Instead, curtain #1 hid the money, as [starting with a working template](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-getting-started-hello-world.html#serverless-getting-started-hello-world-deploy) and slowly refactoring it towards our use-case seemed to be the way forward. Single-unit, downstream-only flow, through `git commit`ing of small changes, to ensure it all integrates smoothly. The only sacrifice of this re-build approach was the loss of the DB which at that point counted 25 visitors already. Not the end of the world, as I developed a quick-and-dirty Python script to migrate these entries into the new, SAM-managed table.  

<script src="https://gist.github.com/LAripping/3119af3513058fb2b0e325b47e6f9d04.js"></script>

<!-- https://gist.github.com/LAripping/3119af3513058fb2b0e325b47e6f9d04 ==TODO embed== -->

Retrospectively, I found out that apparently this is the typical progression of most businesses starting with (not migrating into) the cloud - first they develop everything by hand, clicking-through the console, only to realise after 3 months or so the fragility and rigidness of this, and embracing IaC. At that point though they're forced to rewrite everything from the ground up, as simply lifting-and-shifting into a template is just not gonna work. It is thus once again definitely interesting that the course follows the exact same path. The hard lesson is learned in this safe space so that any future enterprise infrastructure is built the right way from the start. 

Through the Cloud Resume Challenge I also had the opportunity to finally get down to write some **tests** for my code. Tests, whether unit or integration, was another black hole in my coding practices which I was comfortable ignoring for every project I had come to write so far - whether small or large, serious or for funzies. This was thus a great place to start writing tests *the right way* and get some experience with it.

Some testing practices I studied and am proud to [have applied](https://github.com/LAripping/cloud-resume-backend/tree/f442e6341d7a8d87934d585868f2c2502af0ea32#design-decisions) are: 

* clear separation of the "Arrange > Act > Assert" steps
* organised Unit and Integration suites (see image below)
* refactoring the code into distinct blocks honoring the Single Responsibility Principle
* diligent coverage of each distinct code block with the necessary unit tests
* re-usable Fixtures to isolate test case preparation data
* experimentation with "expected-to-fail" cases
* effective use of Mocks to distinguish between code-under-test and dependencies


![](https://raw.githubusercontent.com/LAripping/cloud-resume-backend/f442e6341d7a8d87934d585868f2c2502af0ea32/tests-organisation.png)
*Screenshot of my complete test suite*


##  AWS Skills Certification

An important aspect of the Cloud Resume Challenge worth covering is the "AWS Certified Cloud Practitioner" certificate it challenges you to  earn. Forrest Brazeal ([@forrestbrazeal](https://twitter.com/forrestbrazeal)), the challenge creator, [makes a good point](https://cloudresumechallenge.dev/docs/faq/#does-completing-the-cloud-resume-challenge-guarantee-i-will-get-a-job-in-the-cloud) about the cloud skills shortage and the opportunity to land a well-paying job by proving that you've acquired them. I agree with that sentiment 100% and can confirm this skills shortage through my brief experience in the industry, just like so many challenge-takers also do through their [testimonies](https://cloudresumechallenge.dev/halloffame/). So absolutely do go ahead and take this course and invest in the cert if you're starting in tech and are excited about the cloud!

Back when I started the challenge I decided I wouldn't do the cert, as it would require a little more effort than what at the time I was willing to put in. This step also wasn't free, since sitting the exam costs 100$, so it presents more as an investment if your goal is to get into cloud to make a living. My purpose instead was more of a "let's skill-up in this area which I'm not so comfortable at now that I have the chance". However, as I was getting close to the end of the challenge perfectionism kicked in. I concluded it would be nice to have tangible evidence for this initiation into the cloud world, as a blog post and a simple web page wouldn't really tell the story in a language the industry -looking at you, LinkedIn- speaks: *certs*.  

So in late August, after finishing all the other 15 technical steps, I checked that Task #1 box too!

![](/assets/img/awscert.png)



I prepared for the exam by completing a couple of free courses offered by "AWS Training & Certification" (aka Skills Builder):

- [AWS Cloud Practitioner Essentials](https://explore.skillbuilder.aws/learn/course/134/aws-cloud-practitioner-essentials) 
- [Exam Prep: AWS Certified Cloud Practitioner](https://explore.skillbuilder.aws/learn/course/9449/exam-prep-aws-certified-cloud-practitioner) 

All it takes for these to complete is watching through 10hours or so of videos. I also did the demo exam in [AWS Certified Cloud Practitioner Official Practice Question Set (CLF-C01 - English)](https://explore.skillbuilder.aws/learn/course/12483/aws-certified-cloud-practitioner-official-practice-question-set-clf-c01-english) a couple of times this time taking some notes. These were basically all the studying resources *officially* recommended for the exam. But the most important factor in passing was hands-down the Cloud Resume Challenge and getting that first-hand experience building in AWS.

The exam wasn't difficult but it did contain a few questions (and answers) which I had never seen before anywhere in my preparation, so I'm still not 100% sure the resources I used were sufficient. For any future certs I'll probably give the paid "A Cloud Guru" courses a try. They're also created by Forrest!  


## Conclusion

All in all, the Cloud Resume Challenge is an excellent opportunity to start building in the cloud. The guidance offered hits the right balance between presenting the requirements clearly but also allowing for personal research and experimentation. Through these 3 months I learned a lot and this was by far the most valuable item in my cloud skill-up journey. 

Thanks for reading!   