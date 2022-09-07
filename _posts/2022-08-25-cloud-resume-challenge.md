---
layout: post
title: 'Taking On The Cloud Resumé Challenge'
excerpt_separator: "<!--more-->"
excerpt: "I strongly believe that to learn is to do. Having the longing itch to cover my deficiency in cloud security and cloud in general, I had a big A-Ha moment when I read about <a href='https://cloudresumechallenge.dev/'>the Cloud Resume Challenge</a> in Nick Jones' awesome &quot;Breaking Into Cloud&quot; checklist.<br/><br/>Essentially, the Cloud Resume Challenge is to create a simple web application, but fully cloud-native and using modern DevOps practices. The app will be a statically hosted website showing your Resumé (or just Resume from here onwards, might also use the term CV interchangeably) and will include some backend functionality in the form of a visitor counter. All of this will be created through 15 well-defined tasks.<br/><br/>" 
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

I strongly believe that to learn is to do. Having the longing itch to cover my deficiency in cloud security and cloud in general, I had a big A-Ha moment when I read about [the Cloud Resume Challenge](https://cloudresumechallenge.dev/) in Nick Jones' awesome "Breaking Into Cloud" checklist.


<blockquote class="twitter-tweet">
  <p lang="en" dir="ltr">How to get into cloud security based on my own experiences, and on mentoring and hiring over the last few years. I&#39;ve focused on how to make yourself a success in the field, rather than just the technical knowledge required: <a href="https://t.co/oEhrYHCVda">https://t.co/oEhrYHCVda</a>
  </p>
  &mdash; Nick Jones (@nojonesuk) <a href="https://twitter.com/nojonesuk/status/1531336352972824580?ref_src=twsrc%5Etfw">May 30, 2022</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Essentially, the Cloud Resume Challenge is to create a simple web application, but fully cloud-native and using modern DevOps practices. The app will be a statically hosted website showing your Resumé (or just Resume from here onwards, might also use the term CV interchangeably) and will include some backend functionality in the form of a visitor counter. All of this will be created through 16 well-defined tasks. Who doesn't love checklists?   

![](/assets/img/crc-list-logo.png)

All three public cloud vendors are supported. I picked the AWS flavor, hence this post will only refer to Amazon clouds. The last step is to write a blog post about the journey what I learned and link it in the app, which brings us right here. So let's get to it!


## Why

Having a public CV  in a nice format which I'd have total control over (instead of being limited to the confines of a LinkedIn profile) was something I've been pondering for a while. This was hence a golden opportunity to kill two birds with one stone. So this is how it turned out in the end - introducing my cloud-hosted resume page: https://resume.laripping.com

If familiarisation with cloud is one side of the Cloud Resume Challenge, doing it the DevOps way is the other. The course therefore takes good care to steer you towards concepts like Infrastructure as Code (IaC), Continuous Integration and Delivery (CI/CD) through automated Test suite execution and Github Actions pipelines. Although these concetps are nowadays more ubiquitous than ever, I had never *deliberately* practiced them, so the challenge was the best time and place to do so. 

For example, I find it fascinating that my Cloud Resume App is now in a state that both the frontend and the backend can be updated in seconds, reliably and securely. All without the traditional burdens of buying space in a web server, SSH-ing into boxes using all sorts of credentials personally maintained, and tracking architecture components in READMEs while breaking everything with just a simple Syntax Error... Instead the process now looks more like this: 

1. A `git clone` to fetch the code of both the content AND the infra 
2. Easy setup of a disposable dev env locally
3. Experimental changes can be introduced with a good old `git commit & push`
4. And rolled back to a safe state on error, by putting them through automatically launched Test suites, equipped with the necessary credentials (or secrets) fetched from safe places 
5. Good code (or Infra!) is re-deployed globally on the cloud, refreshing caches



Again, this happens **in seconds** with minimal effort and is yet more robust than ever. No initial expenses or contracts. Everything is hosted where it should be. I find all these elements very impressive.  

It's interesting to note that the challenge steps were laid out in such an order that one is expected to go through this entire flow at least once for both the frontend and the backend, so it subtly forces you to get it right! 



## Challenge Overview

So I embarked on the journey, which lasted around 3 months, ended up using around 50 AWS resources in total, and it all cost me around a penny, as the usage per AWS service utilised was well within the [AWS Free Tier](https://aws.amazon.com/free/) ranges. The only real fee that's expected is for a domain name, but I chose to host it under as subdomain of a domain name I already had.   

![](/assets/img/cost.png)
*&pound;0 cost for the 3 months of experimentation confirmed through the AWS Cost Explorer service*

{% include note.html title='' content='*To track the resource count I started tagging everything I would create through the console with `Project: cloud-resume`. Initially I thought this way it would be easy to tear them all down at once in the end, by listing all such tagged resources through the  [Resource Groups](https://docs.aws.amazon.com/ARG/latest/userguide/resource-groups.html)  service. Down the road, however, I grew fairly proud of the app and decided to keep it all up there.<br/><br/>This tagging approach turned out to be rather naive though as later on, when things were being created automatically, any tags could only be applied to the resource container i.e. the "Stack"...' %}



Another fun fact is that throughout the entire journey, with all the googling involved, I tried to avoid any places like Stack Overflow questions or blog posts that seemed too similar to my problem, as these could come from other challenge takers'. The point was not only to veer away from any "shortcuts" that would hinder my learning, but to also keep my Cloud Resume 100% true to the intended plan. The internet is a helpful place, but it's doable. 

Looking at the whole picture, it's important to make a special note to the **security** aspect, or "Priority Zero" as Amazonians refer to it. As a security practitioner I had this extra concern throughout the journey as this was a rare occasion where I was building something from the ground up, something that would sit "in production", out there for the world to see. The [AWS shared responsibility model](https://aws.amazon.com/compliance/shared-responsibility-model/) is there to remind any cloud-builder that some of the risk is now assigned directly to you, not the hosting provider, and that the fancy pay-for-what-you-use pricing model can also be turned against you in the case of malicious use - should you omit to address this.

![](https://pbs.twimg.com/media/FYPaHC-WQAIwfDE?format=jpg&name=900x900)


As such, I wanted to summarise some thoughts about the Security of the end product, as some sort of basic security model, to make note of the related AWS services coming into play whether visibly or behind the scenes:

|                   Threat                    |                          Mitigation                          | Responsibility |
| :-----------------------------------------: | :----------------------------------------------------------: | -------------- |
| Distributed Denial of Service (DDoS) Attack | AWS Shield Standard enabled **by default** <br/>on the CloudFront distribution | AWS            |
|   API Abuse -> Multi-service Usage Costs    | Configured Lambda Concurrency Control<br/>Configured DB capacity units,<br/>Set up AWS Budgets to be alerted on unexepcted costs | Mine           |
|             Supply-chain Attack             | Sub-Resource Integrity (SRI) enabled on 3rd party JS libraries | Mine           |
|           Leaked IAM Credentials            | Followed "Principle of Least Privilege", <br/>Used different users/roles for different functions | Mine           |



<!-- 

* IAM: Principle of least privilege. To nail this, I had the privilege of trial-and-error to find manually the absolute minimum number of IAM perms needed by each entity to perform it's purpose, like the ([auto-generated](https://github.com/LAripping/cloud-resume-backend/blob/f442e6341d7a8d87934d585868f2c2502af0ea32/template.yaml#L24)) `FetchVisitorsFunctionRole` which only needs the ` dynamodb:PutItem` and `dynamodb:Scan` permissions against only our table, to allow the backend code to talk with the DB.... Other times the least necessary policies is known in advance, like for my `sam-cli-user` user, which the SAM CLI uses when building the backend, thus only needing [specific permissions](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-permissions.html#sam-permissions-managed-policies). 
 -->
  



## Stops Along the Way

In the next sections I'll share some more specific impressions of each stop of the journey and document any business decisions taken. Roughly grouped by the aspect of the application that the relevant steps covered, these might come in handy to any inspired readers that will go ahead and try the challenge themselves.



### The Frontend - HTML / CSS / JS

In the early steps (2,3 and 7) I went well overboard and snatched the opportunity to make my web Resume look exactly like what I had it in mind. Namely, I added: 

- a "Download as PDF" button (which required some research but turned out good enough) 
- with a basic spinner while the generation is cooking
- and a sticky sidebar to host all the buttons featuring Font Awesome icons

All totally unnecessary for the challenge itself but fun to experiment with!  



### The Stage - DNS / HTTPS / CDN

When it comes to the steps covering the Infrastructure (Steps 5 and 6), things got a bit blurry. My end goal was slightly more complicated than the one presented as I was bringing-my-own DNS provider instead of using the proposed Route53, and used a subdomain instead of an apex one. All while using CloudFront to familiarise with Edge delivery.  

Nevertheless, it all fit into place and I had a pretty solid topology looking like this:

![](/assets/img/mermaid-frontend.png)



Additionally, it was very interesting to see how **Domain Name validation** is performed when issuing an HTTPS cert. A process I deemed important yet had never gone through and was thus always curious to see

It's the age of Let's Encrypt and HTTPS certs are generated with 2 clicks, so how does it work; this so-called verification of the owner we were taught at school as very important for the whole web-of-trust system?  

When this moment finally came, the relevant service - Amazon Certificate Manager (ACM) offered validation via Email or DNS. While the Email way was fairly striaghtforward, expected and traditional, the DNS way performed [via specially configured CNAME records](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html) absolutely blew my mind as a genius idea for the problem! After some trial and error (and annoying waits for global DNS infra to catch-up) I had my shinny new cert:

![](/assets/img/cert.png)
*"It even has a watermark"<br/>— Patrick Bateman, American Psycho (2000)*



### The Backend - Lambda / DynamoDB / API gateway

![](/assets/img/mermaid-api.png)

Here things made much more sense as the topology proposed by the challenge for the application's backend (steps 8,9 and 10) was actually fairly common, if not recommended even. As the Lambda-DyamoDB-API Gateway combo is simple, scalable and serverless!  



![](/assets/img/aws-flow.png)
*It's right there on the Lambda Service "Getting Started" page*



When designing the Python logic, the first step was to decide how to "count visitors". The challenge description was coarse enough to allow for creative interpretations of what a unique "Visitor" really is and how individual sessions should be captured and counted from a server.  The definition I came up with is the following:   

`Visitor = browser (UA) || IP`: *Essentially, to count as a different "visitor" each new combination of IPv4 address and User-Agent performing the API request that triggers the Lambda function.*

After this decision, the rest of the application's backend was built from the inside-out. Starting from the simple, 2-column database, then figuring out the exact DB queries needed to perform the ops needed through the AWS CLI, and translating those queries into Python using the [boto3](https://aws.amazon.com/sdk-for-python/) SDK for the Lambda function. 

But where do I write this Python code for my function? For several reasons like IDE sugar, Test-integration, versioning plugins etc. the Management Console's built-in editor (AWS Cloud9) didn't seem the right option, yet no hint was given by the challenge on how to do this right... After pondering around trying to stitch-together a flow using the ["AWS Toolkit" PyCharm plugin](https://aws.amazon.com/pycharm/) and local containers I gave up and decided to do it the wrong way, writing the prototype Python code on-the-fly via the web editor. What was missing was revealed later down the road, as the glue for all the moving parts (development-deployment-testing) was the **SAM CLI**, covered a bit more in the next section. Even the typical SAM CLI use case demonstrated everywhere was the same as ours: API Gateway + Lambda. Cheeky!  

Finally, I designed the simple 1-endpoint API as "the contract" that both the backend and the frontend would have to abide by:   


![](https://github.com/LAripping/cloud-resume-backend/raw/f442e6341d7a8d87934d585868f2c2502af0ea32/apispec-expand.png)
*Explore the API specification [here](https://petstore.swagger.io/?url=https://raw.githubusercontent.com/LAripping/cloud-resume-backend/master/apispec.yml)*







###   DevOps! - SAM / Github Actions / Tests


Having manually written a prototype for both the frontend and backend, the challenge for these steps (11,12,13 and 14) was to re-write it all make this whole process automatic, speeding deployments and enabling rapid experimentation. For this, the challenge suggested I use SAM, or Serverlesss Application Model, and the corresponding CLI. I learned that SAM is a concept allowing for serverless cloud infrastructure to be described in a YAML format, like code (which is well-defined, can be versioned, yada yada). More importantly though, it's supposed to simplify operations against this codebase via simple commands like `sam init && sam deploy ` through a wrapper SAM CLI. 

{% include note.html title="Note:" content="My understanding is that the SAM CLI is a wrapper over Amazon's IaC offering - &quot;CloudFormation&quot;, which wasn't really dev-friendly or otherwise easy for the user (or scripts) when accessed through the traditional AWS CLI" %}

The road here presented a fork, where the challenge taker would have to

1. either re-write everything built so far as YAML code to form a SAM stack
2. or import these precious but rigid resources into an empty new stack

While I initially tried to cheat through option #2 I quickly figured out that importing existing CloudFormation resources in stacks generated and managed through SAM is a massive pain and probably not doable at all. Thinking back to it, this could as well be a deliberate design by Amazon to force you down the right path ;)  

Instead, curtain #1 involved [starting with a working "hello-world" template](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-getting-started-hello-world.html#serverless-getting-started-hello-world-deploy) and slowly refactoring it towards our use-case, step by step `git commit`ing small changes along the way, to ensure it all integrates smoothly. The only sacrifice of this approach was the loss of the DB which at that point counted 25 visitors already. Not the end of the world, as I developed a quick-and-dirty Python script to migrate these entries into the new, SAM-managed table.  

https://gist.github.com/LAripping/3119af3513058fb2b0e325b47e6f9d04 ==TODO embed==

Retrospectively, I found out that apparently this is the typical progression of most businesses starting with (not migrating into) the cloud - first they develop everything by hand, clicking-through the console, only to realise after 3 months or so the fragility and rigidness of this, and embracing IaC. At that point though they're forced to rewrite everything from the ground up, as simply lifting-and-shifting into a template is just not gonna work. It is thus once again definitely interesting that the course has you go down the exact same path. The lesson is learned in this safe space so that your future enterprise infrastructure is built the right way from the start. 

Through the Cloud Resume Challenge I also had the opportunity to finally get down to write some **tests** for my code. Another black hole in my coding practices which I've consistently been pushing under the rug for every project I had come to write - whether small or large, serious or for funzies. Here was a great place to start and get some experience writing tests *the right way*.

Some testing practices I studied and am proud to [have applied](https://github.com/LAripping/cloud-resume-backend/tree/f442e6341d7a8d87934d585868f2c2502af0ea32#design-decisions) are: 

* clear separation of the "Arrange -> Act -> Assert" steps
* organised Unit and Integration suites (see image below)
* refactoring the code into distinct blocks honoring the Single Responsibility Principle
* diligent coverage of each distinct code block with the necessary unit tests
* re-usable Fixtures to isolate test case preparation data
* experimentation with "expected-to-fail" cases
* effective use of Mocks to distinguish between code-under-test and dependencies


Here's a screenshot of my complete test suite

![](https://raw.githubusercontent.com/LAripping/cloud-resume-backend/f442e6341d7a8d87934d585868f2c2502af0ea32/tests-organisation.png)



##  Cert & Exam

A big component of the Cloud Resume Challenge was the "AWS Certified Cloud Practitioner" certificate it challenges you to take and I wanted to write a few words about this as well.

When I started with the challenge I decided I wouldn't do the cert, as it would requires some more serious effort which at the time I wasn't willing to put in. This step also wasn't free. The exam costs 100$, so it presents more as an investment if your goal is to get into cloud to make a living out of it. My purpose instead was more of a "let's skill-up in this area which I'm not so comfortable at now that I have the chance".

The challenge creator [makes a good point](https://cloudresumechallenge.dev/docs/faq/#does-completing-the-cloud-resume-challenge-guarantee-i-will-get-a-job-in-the-cloud) about the cloud skills shortage in the industry and the opportunity to land a well-paying job by proving that you've acquired them. I agree with that sentiment 100% and can confirm this skills shortage through my brief experience in the industry, just like so many challenge-takers [testimonies](https://cloudresumechallenge.dev/halloffame/) also do. So absolutely do go ahead and take this course and invest in the cert if you're starting in tech.

However, as I was getting close to the end of the challenge perfectionism kicked in. I concluded it would be nice to have tangible evidence for this initiation into the cloud world, as a blog post and a simple web page wouldn't really tell the story in a language the industry -looking at you, LinkedIn- speaks: *certs*.  

So in late August, after finishing all the other 15 technical steps, I checked that Task #1 box too!

![](/assets/img/awscert.png)



I prepared for the exam by completing a couple of free courses offered by "AWS Training & Certification" (aka Skills Builder):

- [AWS Cloud Practitioner Essentials](https://explore.skillbuilder.aws/learn/course/134/aws-cloud-practitioner-essentials) 
- [Exam Prep: AWS Certified Cloud Practitioner](https://explore.skillbuilder.aws/learn/course/9449/exam-prep-aws-certified-cloud-practitioner) 

And did the demo exam in [AWS Certified Cloud Practitioner Official Practice Question Set (CLF-C01 - English)](https://explore.skillbuilder.aws/learn/course/12483/aws-certified-cloud-practitioner-official-practice-question-set-clf-c01-english). 

These were basically all the resources officially recommended by AWS for the exam. But the most important factor in passing was doing the Cloud Resume Challenge and getting that first-hand experience building in AWS.

The exam wasn't difficult but it did contain a few questions (and answers) which I had never seen before anywhere in my preparation, so I'm still not 100% sure these resources were sufficient. For any future certs I'm thinking of giving the paid "A Cloud Guru" courses a try. 



## Conclusion

All in all, the Cloud Resume Challenge is an excellent opportunity to start building in the cloud. The guidance offered hits the right balance between presenting the requirements clearly but also allowing for personal research and experimentation. Through these 3 months I learned a lot and this was by far the most valuable item in my cloud skill-up journey. 

Thanks for reading!   