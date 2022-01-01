---
layout: post
title: 'Wind Vision Android Application: Multiple Vulnerabilities'
excerpt: "<a href='https://www.wind.gr/gr/gia-ton-idioti/vision/'>Wind Vision</a> is a digital television service offered by WIND Hellas, a Greek telecommunication provider, allowing for streaming of digital content. The Wind Vision mobile application, available for Android and iOS devices, allows users to watch TV 'on the go' from their smartphone devices.<br/><br/>The Wind Vision Android application is available on <a href='https://play.google.com/store/apps/details?id=gr.wind.windvision'>Google Play Store</a>. The latest version currently available at the time of writing (10.0.16) was found vulnerable to four security issues. The vulnerabilities could be combined into an attack chain that would allow a malicious third party application to takeover a victim user's account. After compromising a legitimate account, an adversary could proceed to download and watch TV content abusing the victim’s subscription, or deny the victim access to their account by changing the PIN code and replacing registered devices."
categories:
  - "cve-advisory"
tags:
  - android
  - apps
  - wind
  - zappware
last_modified_at: 2021-12-22T15:35:00
---

{% include note.html content="This advisory was originally published on [F-Secure LABS](https://labs.f-secure.com/advisories/wind-vision)" %}


| **Product** | Zappware Nexx4 IPTV Solution |
| **Severity** |<span style="color:orange">Medium</span> |
| **CVE IDs** |	CVE-2021-22268, CVE-2021-22269, CVE-2021-22270, CVE-2021-22271[^2] |
| **Type**	| Insecure Authentication, PIN Code Leakage, URL Hijacking, Reproduceable Device ID |

<br/><br/>

{% include toc.html %}

## Introduction

[Wind Vision](https://www.wind.gr/gr/gia-ton-idioti/vision/) is a digital television service offered by WIND Hellas, a Greek telecommunication provider, allowing for streaming of digital content. The Wind Vision mobile application, available for Android and iOS devices, allows users to watch TV "on the go" from their smartphone devices. 

The Wind Vision Android application is available on [Google Play Store](https://play.google.com/store/apps/details?id=gr.wind.windvision). The latest version currently available at the time of writing (10.0.16) was found vulnerable to four security issues. The vulnerabilities could be combined into an attack chain that would allow a malicious third party application to takeover a victim user's account. After compromising a legitimate account, an adversary could proceed to download and watch TV content abusing the victim’s subscription, or deny the victim access to their account by changing the PIN code and replacing registered devices.

Wind’s IPTV infrastructure utilised Zappware’s "Nexx4" solution. Zappware's cross-platform solutions for DVB, IPTV and OTT services are used by telecommunication providers operating across multiple countries, including Wind Hellas. Some of the providers are:

- A1 Croatia
- A1 Bulgaria
- A1 Slovenia
- Orange Belgium
- A1 Austria
- Trinidad and Tobago / Caribbean Amplia

Since the issues identified in the Wind Vision application affect Zappware's software, millions of users worldwide[^1] are potentially at risk.


## Disclaimer

Although Zappware was contacted multiple times in the last months, at the time of writing the Wind Vision Android application has not been updated to include appropriate patches of the discovered vulnerabilities. It is believed that remediation of severe security issues such as the ones described here should be prioritised over the development of new features, to safeguard the privacy of users and protect their accounts. The vulnerabilities were disclosed responsibly to Zappware and Wind, providing detailed remediation instructions and recommendations, to aid the patching process. 

The following section provides a brief overview of the vulnerabilities discovered, while further technical details will be released at a future date.



## Vulnerabilities Discovered

**CVE-2021-22268 : Insecure Authentication**

Wind Vision authenticated users with an Oauth2 "Authorisation Code" flow, utilising a web browser. The chosen flow was insecure as the granted code could be intercepted by third party applications and exchanged for a valid session token. This issue, when combined with the URL Hijacking described below, enabled the takeover of a victim account after tricking the user to click on the wrong handler application. 

**CVE-2021-22269 : PIN Code Leakage**

The "master PIN code" required by the application in order for certain settings to be configured, was leaked in server communications. It was therefore possible to intercept this 4-digit code by analysing network traffic, as the application was not found to employ certificate pinning.

**CVE-2021-22270 : URL Hijacking**

The Wind Vision application made insecure use of the URL-scheme Inter-Process
Communication (IPC) mechanism offered by the Android operating system. The Deep Link implementation is susceptible to "URL Hijacking", an attack that allows malicious third-party applications installed to be launched or to steal sensitive data by tricking the user into accepting the wrong "handler" for a registered URL scheme. 


**CVE-2021-22268 : Reproduceable Device ID**

Wind Vision's users can register a number of devices, which are then tracked by the solution using a Device ID generated locally. This generation process was however found to be reproduceable, since it did not employ randomness. As a result, a valid Device ID could be re-created by third party applications running on the same device. Should such malicious applications also hold a valid session token, they could then issue valid requests against the Wind Vision server accessing all user functionality including TV content streaming. As other vulnerabilities discovered (CVE-2021-22270, CVE-2021-22268) could be chained to obtain a session token, combining exploitation of this issue lead to the complete takeover of Wind Vision accounts.

## Credit

The issue was discovered by Leonidas Tsaousis ([@laripping](https://twitter.com/laripping)) of F-Secure Consulting.


## Timeline

| Date | Summary |
| ---- | ------- |
| 14/11/2020 | Wind Vision Application developer contacted. No response was received. |
| 20/11/2020 | Zappware contacted. |
| 30/11/2020 | Zappware replied acknowledging the vulnerabilities. No patch date provided. |
| 22/12/2020 | Zappware contacted again. No patch date provided. |
| 04/01/2021 | Temporary CVE IDs[^2] reserved by F-Secure CNA. |
| 29/01/2021 | Limited Advisory released |



## Update 12/03/2021

Zappware and Wind have released a new version of the Wind Vision application (10.0.18) on Google Play Store to address the issues. Users are advised to update to the latest version. Additionally, Zappware has clarified that the vulnerable authentication mechanism was not used for clients other than Wind Vision and their customers are therefore not affected.  

[^1]: Zappware's IPTV solution is believed to be *"currently deployed on millions of devices around the world"* ([source](https://www.linkedin.com/company/zappware/about/)).

[^2]: Note that the CVE IDs provided are placeholders due to be replaced after public disclosure is made.