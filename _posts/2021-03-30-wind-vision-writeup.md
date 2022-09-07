---
layout: post
title: "Click here for free TV! Chaining bugs to takeover Wind Vision accounts"
excerpt: "Last year, while playing around with the <a href='https://play.google.com/store/apps/details?id=gr.wind.windvision'>Wind Vision mobile application</a>, I noticed that the login process was implemented in a potentially risky way. I decided to take a look, which led to an in-depth analysis over the course of two months. In brief, I found a way for a malicious app to takeover the victim's Wind Vision account, by chaining a series of otherwise unimportant bugs, starting with just one wrong click. As a note, the issues have already been <a href='https://labs.f-secure.com/advisories/wind-vision/'>responsibly disclosed</a> to Wind and the software vendor, and the app was recently updated to prevent the attack.<br/><br/>This post aims to highlight the caveats of authentication flows and inter-process communications (IPC) for mobile application developers, and to also outline the overall risk imposed by these flaws to end users.<br/><br/>"
categories:
  - "blog-posts"
tags:
  - wind
  - android
  - apps
  - zappware
  - tv
  - streaming
  - takeover
last_modified_at: 2021-12-24T15:13:00
# post-img: assets/img/windvision-xpand.jpeg
post-img: assets/img/diagram-transparent-blackfont.png
---

{% include note.html title="Note:" content="This blog post was originally published on [F-Secure LABS](https://labs.f-secure.com/blog/wind-vision-writeup/)" %}

{% include toc.html %}

## Intro 

Last year, while playing around with the [Wind Vision mobile application](https://play.google.com/store/apps/details?id=gr.wind.windvision), I noticed that the login process was implemented in a potentially risky way. I decided to take a look, which led to an in-depth analysis over the course of two months. In brief, I found a way for a malicious app to takeover the victim's Wind Vision account, by chaining a series of otherwise unimportant bugs, starting with just one wrong click. As a note, the issues have already been [responsibly disclosed](https://labs.f-secure.com/advisories/wind-vision/) to Wind and the software vendor, and the app was recently updated to prevent the attack. 

This post aims to highlight the caveats of authentication flows and inter-process communications (IPC) for mobile application developers, and to also outline the overall risk imposed by these flaws to end users.


## Wind Who?


[Wind Vision](https://www.wind.gr/gr/gia-ton-idioti/vision/) is a digital television service by WIND Hellas, a major telecommunications provider in Greece. It is the next generation solution for TV streaming which was traditionally provided via satellite, and basically comprises of a Set Top Box (STB) connected to a screen via HDMI. All digital content is retrieved over IP networks, and the STB only needs a typical home broadband (ADSL) connection. Similarly, TV content can be fetched "on the go" from smartphone devices over any internet connection using Wind Vision's mobile application. At the time of writing, the Android version of the Wind Vision mobile application was installed to more than 50.000 devices.

With the necessary introductions taken care of, let's proceed with how it all started. 

## URL Schemes 


Having installed the app, I focused on the feature that initially attracted my attention: The application logged users in by opening a browser tab, where the credentials would be submitted and if correct, the user would be navigated back to the application. This transition was subtle:

<img src="/assets/img/wind-mini-gif.gif" style="height:25em"/>

Instantiating the webview is a trivial task, but interception of the server's response from the application layer requires some Inter-process Communication (IPC). Wind Vision achieved this by declaring a [Deep Link](https://developer.android.com/training/app-links/deep-linking) that handled specific URL schemes, as can be seen on its manifest file attached below:

```xml
<activity android:name="com.zappware.nexx4.android.mobile.ui.startup.login.LoginActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:host="pridp.wind.gr"
            android:path="/AuthCallback"
            android:scheme="nexx4" />
```

As can be seen in the above snippet, this IPC mechanism is implemented insecurely, as it lacks the `android:autoVerify="true"` field in the `<intent-filter>` element, which would make it an [App Link](https://developer.android.com/training/app-links/verify-site-associations.html) instead of a Deep Link. In brief, the difference between Deep Links and App Links lies in the fact that the latter can only be opened by the designated app and this association can be verified at install time.

When multiple apps declare the same URL scheme, users will be prompted with a dialog box as seen below, to select the "handler" application. A wrong click there could lead to the wrong app receiving sensitive data that was intended for the legitimate application.


<img src="/assets/img/imgur-handlers.png" style="height:25em"/>


A malicious application could also trick users into setting itself as the “Preferred" handler, disabling all future prompts... Historical incidents have shown that relying on users for security decisions can be a bad practice and this is why App Links were introduced, to skip this handler altogether and thus eliminate the attack.

In and of itself, insecure URL schemes is not a new subject in mobile security... The risk has been known for quite a while but it is not commonly addressed, even in cases where such functionality is used for critical operations such as authentication. Although relevant vulnerabilities are raised by testers, a practical attack to demonstrate the impact is usually prevented by other means, or there's nothing particularly useful for an attacker to "steal" using this technique. As we'll see below however, in this occasion something of real value was exposed by this flaw: OAuth Authorisation Tokens.

In order to see how these URL schemes were used and what data was exchanged with the webview, one would have to proxy the application's network traffic. Although certificate pinning was employed, I was able to bypass this with Frida and a standard [codeshare script](https://codeshare.frida.re/@pcipolloni/universal-android-ssl-pinning-bypass-with-frida/) as seen below: 

```bash
device$ cp /sdcard/burp.crt  /data/local/tmp/cert-der.crt
device$ chmod 777 /data/local/tmp/cert-der.crt
icarus$ frida -U --codeshare pcipolloni/universal-android-ssl-pinning-bypass-with-frida -n gr.wind.windvision
...
[o] App invoked javax.net.ssl.SSLContext.init...
[+] SSLContext initialized with our custom TrustManager!
```

## Examining the Authentication Flow


Resuming the analysis of the login process, now able to examine it end-to-end, I took a quick note of what is eventually required by the application server. By repeating an API request, progressively stripping request parameters it came down to just two HTTP headers which were required by the "GraphQL" Wind Vision server:

```
Authorization: AWS4-HMAC-SHA256 Credential=ASIAUN....
Device-Id: R2pNRE...mZHhEWQA99
```

> More details about this `Device-ID` field will be described later on in this post...

A simplified walkthrough of the authentication and authorisation flow observed is attached below, redacted when necessary. In OAuth speak, this is known as a [OpenID Connect](https://openid.net/connect/) flow with an [Authorization Code](https://www.oauth.com/oauth2-servers/server-side-apps/authorization-code/) grant.

1. **Configuration Request**, issued by the native application (OkHttp client), which retrieves the URL to be loaded in the webview.
    
   ```
   GET https://pridp.wind.gr/.well-known/openid-configuration
    
   200 OK 
   {
       “version”:”1.0”,  
       “authorization_endpoint”:”https://pridp.wind.gr/oauth2/v1/authorize“,
       ...
   ```
   
2. **Authorization Request**, issued by the webview instantiated by the application, to kickoff the login flow.
   ```
   GET https://pridp.wind.gr/oauth2/v1/authorize?
           response_type=code&
           redirect_uri=nexx4://pridp.wind.gr/AuthCallback&
           scope=openid offline_access profile IPTVUserID&
           client_id=CLIENT_ID
    
   302 Found
   Location: /my.policy
   ```

3. **Policy Request**, which retrieves the URL of the Login page.
   
   ```
   GET https://pridp.wind.gr/my.policy
    
   200 OK
   <html>
       <body>
       ...
           <script>
               document.external_data_post_cls.action = unescape(“https://www.wind.gr/wind/v2/myTv/login/myTvLogin.jsp/“);        
               document.external_data_post_cls.submit();
           </script>
       </body>
   </html>
   ```

4. **Login Page Request**, to retrieve and render the HTML form contained in the Login page. Also includes multiple JavaScript files, one of which includes the URL to submit the credentials to.
   
   ```
   POST https://www.wind.gr/wind/v2/myTv/login/myTvLogin.jsp/
   
   200 OK
   <html>
       <body>
           <form>
           ....
   ```

5. **Login Requests**, submitting user’s credentials to the aforementioned URL, performed by JavaScript code executing inside the webview.
   
   ```
   POST https://www.wind.gr/ATGWebservicesProxy/PayTVLogin/
    
   username=USERNAME&password=PASSWORD&...
    
   200 OK
   { 
      “zapwareIDs”:[ZID], 
      “status”: “LOGIN” or “ERROR” 
   }
   ```

   The status result field dictates whether the second request below will be performed, re-submitting credentials to the policy endpoint. In case of successful authentication, the webview redirects the flow back to the native application using a URL schemes registered as the one highlighted below:

   ```
   POST https://pridp.wind.gr/my.policy?_DARGS=/wind/myTv/login/myTv.jsp.2
    
   username=USERNAME&password=PASSWORD&customeriptvid=ZID...

   302 Found
   Location: nexx4://pridp.wind.gr/AuthCallback?code=CODE
   ```

6. **Token Request**, issued by the native application code, to exchange the authorization code with a valid session token. The `client_id` and `client_secret` values can be found on the application source code after basic reverse engineering, as no obfuscation was employed.

   ```
   POST https://pridp.wind.gr/oauth2/v1/token
    
   code=CODE&grant_type=authorization_code&redirect_uri=nexx4://pridp.wind.gr/AuthCallback&
   scope=openid offline_access profile IPTVUserID&client_id=CLIENT_ID&client_secret=CLIENT_SECRET

   200 OK 
   {    “id_token”: “eyJh....”
    ```

7. **Access Key Request**, finally issued by the native application code against a previously defined AWS endpoint, to exchange the `id_token` to an access key which can then be used  in all subsequent GraphQL requests towards the Wind Vision server.
    
   ```
   POST https://cognito-identity.eu-west-1.amazonaws.com/
   {
       “IdentityId”:”eu-west-1:1a18ca6b-0c60-404a-b196-9667b460fc17”,
       “Logins”:{
           “pridp.wind.gr”:”eyJh...”

   200 OK
   {
       “Credentials”:{
           “AccessKeyId”:”ASIA....”
   ```

8. **GraphQL API Calls**, like the one below, all subsequent GraphQL requests to the API server are authenticated with this Access Key retrieved previously, attached in the Authorization header:
   
   ```
   POST https://client.tvclient.wind.gr/secure/v1/graphql/
   Authorization: AWS4-HMAC-SHA256 Credential=ASIA....
   {
      “query”:”query User { me { ...”,
          “operationName”:”User”,
      “variables”:{}
   }

   200 OK
   {     “data”:{          ....
   ```

As can be seen in steps 2 and 6 however, no further claims need to be known by the initiator when requesting authorization, which is directly granted a token by the server, without any identity verification. As a result, anyone in possession of a valid authorization code such as the `CODE` value retrieved in step 5, can exchange it for an OAuth token, and subsequently an API access key. As the authorization code was sent across applications using the aforementioned insecure URL Schemes, steps 6-8 can be reproduced by a malicious third party application who would intercept this code. This would eventually lead to the compromise of the user’s account, as all other parameters involved were constants.

This flaw could be avoided if the OAuth [Proof Key for Code Exchange](https://oauth.net/2/pkce/) (PKCE) extension was used, which prevents interception of authorization requests by requiring cryptographic verification. Briefly, the extension works by first including a secret in the initial request, which is then transformed and re-sent when exchanging the authorization code obtained for an access token. This way if just the code is intercepted, it will not be useful to the attacker since it the subsequent token relies on this initial secret which the attacked could not be in possession of. React Native developers have [described](https://reactnative.dev/docs/0.62/security#oauth2-and-redirects) this concept well in the following diagram, where the highlighted `state`, `code_challenge` and `code_verifier` parameters are the key factors:


<img src="/assets/img/pkce.png" style="height:40em"/>


## Coding Time!


To demonstrate how exploitation of the issues above could be combined, I created a PoC Android application. The resulting source code can be found on Github:  

[https://github.com/FSecureLabs/WindVision-PoC-app](https://github.com/FSecureLABS/WindVision-PoC-app)

To make the attack more realistic, the dialog shown by this demo application when claiming the Deep Link could be stylised exactly like the Wind Vision application, to make it as convincing and confusing for the victim user as possible:


<img src="/assets/img/urlschemes.jpeg" style="height:25em"/>
<p style="text-align:center; font-style:italic"> Which would you choose? </p>


If the copycat application is selected (spoiler: it's not as *loud* as its target), our application will obtain the OAuth authorization code, and will be able to exchange it, behind the scenes, with a valid session token on behalf of the victim user. This will in turn be used to authenticate on Wind's GraphQL server, effectively allowing all user functionality to be accessed by the attacker. For the purposes of demonstration our application will stop there, after issuing just one such GraphQL API call to fetch some user-specific data from the server, ideally sensitive, and display it in a scary Toast notification. But which one?

The Wind Vision service restricted the user into accessing the application from one of only 3 pre-registered devices. These 3 devices could be displayed and configured from within the application, so to finish off the demonstration, I chose this API call, that listed the user's registered devices. However, this API call must also originate from a registered device... So how does the server recognise a registered device anyway? Can this be "reproduced" somehow? The focus now shifts to the device identification mechanism.


## (Re) Generating a Device ID


By inspecting the traffic and some trial-and-error in Burp's Repeater, I soon realised that the GraphQL server only accepted requests with a valid "Device ID" header, one that has been previously uploaded after the device registration, rejecting requests if it's not present or not previously registered. This identifier was locally generated. 

Back to our malicious application perspective, an attacker would therefore have to either guess a valid Device ID or register a new one using the session token obtained, potentially replacing an existing device in the process. Although intrusive, the latter approach would typically be the only way forward, however the Wind Vision application allowed for the former as well...

Examining the decompiled Java code of the application revealed the routine responsible for the generation of the Device ID, a simplified version of which is attached below:

```java
private void calculateDeviceId() {
        UUID UUID = new UUID(-1301668207276963122L, -6645017420763422227L);
        byte[] deviceUniqueID = new byte[0];
        deviceUniqueID = new MediaDrm(UUID)
            .getPropertyByteArray(MediaDrm.PROPERTY_DEVICE_UNIQUE_ID);
        String id = Base64.encodeToString(deviceUniqueID, 2)
            .replaceAll("=", "99")
            .replaceAll("/", "88")
            .replaceAll("\\+", "77");
        if (id.length() >= 100) {
            id = id.substring(0, 99);
        }
        Log.d("DLA", "ID calculated is: "+id);
}
```


As can be seen above, the Wind Vision application code relied primarily on a property of the system's [MediaDRM](https://developer.android.com/reference/android/media/MediaDrm#PROPERTY_DEVICE_UNIQUE_ID) service and a few more constant values, to generate the Device ID. Instead, the generation of unique identifiers should in general employ cryptographically secure randomness, to  prevent reproduction of the values created, also known as "pre-image" attacks, or "collisions". Due to the observed implementation however, the value created could be deterministically re-created in subsequent executions, even in the context of a different application. The only requirement for this would be to execute this function on the same device as the victim, to make use of the same MediaDRM service instance. This routine was thus copied to the example application, to generate a valid Device ID and attach it to the last, forged request to the server.


## Locking the User Out

With all things now in place to forge authenticated GraphQL requests to the server, I took a look at my proxy history searching for any other interesting API calls. This is where another API call came up, one that retrieved a 4-digit code whose purpose was not immediately apparent. The HTTP request and response are attached below:

```http
POST /secure/v1/graphql/ HTTP/1.1
Authorization: AWS4-HMAC-SHA256 Credential=ASIA...
Host: client.tvclient.wind.gr
Device-Id: R2pN...
Content-Type: application/json; charset=utf-8

{
    "query": "query User { me { __typename id firstName ... guestMode masterPincode trackViewingBehaviour ... }",
    "operationName": "User",
    "variables": {}
}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8

{
    "data": {
        "me": {
            "__typename": "User",
            "masterPincode": "0000",
            "fingerprintId": "GkXzO2p77",
            "eventLoggingOptions": null,
            "quickGuideVideo": null
        }
    }
}
```


This code was the "master PIN code" that the Wind Vision application requested from the user, before allowing configuration of certain settings, including management of registered devices. This journey consisted of 3 steps shown below, from the Settings, clicking "Devices / Account", and entering the PIN before changing the devices in "Device Management":

| Step 1 | Step 2 | Step 3 |
| ------ | ------ | ------ |
| <img src="/assets/img/pin1.jpeg"/> | <img src="/assets/img/pin2.jpeg"/> | <img src="/assets/img/pin3.jpeg"/> |


Exchanging device-level authentication credentials presents a dangerous practice. Second-level mobile authentication in general, replaces the username:password web application standard, striking a balance between protecting user data from mobile-specific threats such as device theft, while also relieving the user of having to type in their credentials too often. Unlike traditional username:password pairs though, device-specific authentication credentials should be generated and checked locally, and must never leave the device or be exchanged with the server. Secure platform features must be used instead to store these securely such as the [Android Keystore](https://developer.android.com/training/articles/keystore). In short, the advice can be summarised to:

<!-- > *Local authentication credentials should be generated, stored and checked locally.* -->

{% include tip.html title="Tip:" content="*Local authentication credentials should be generated, stored and checked locally*" %}
 

Back to the Wind Vision implementation though, at first thought sending it over HTTP might appear unimportant and of minimal risk as a design approach - considering that interception of authenticated traffic could only be performed by the legitimate user itself.

In this instance however, we've already discovered and proven an attack path in which an authenticated session can also be achieved  by a malicious third party... Therefore, this final flaw further aggravates the impact of the full attack, as the adversary can now exfiltrate the code like the authorisation code and session token to then enter it in their own application, "unlocking" the Device Management screen and "locking the user out", by removing their devices. 

 
## The Full Chain


Let's summarise the attack chain so far: an attacker in the form of a third-party application installed on the victim's device, could hijack the user's Wind Vision session if the wrong option is clicked. Note that both the stolen session token and the reproduced Device ID could also be exfiltrated instead of used locally as in our PoC application. This essentially means that the final steps of the takeover **could take place from a remote location, allowing an adversary to stream TV content using the victim's Wind Vision account, and then lock the victim out**, by using their PIN to remove previously configured devices. For the more visual readers, here's a diagram that depicts this attack path:

<img src="/assets/img/diagram-transparent-blackfont.png" />

The account takeover attack is demonstrated end-to-end in the following video. The credentials of the account used for the demonstration have been redacted. 

<video controls="controls" style="display:block; margin:auto; height:30em">
  <source src="/assets/video/wind-demo.mp4" type="video/mp4">
</video>


## Bonus Round: Bypassing Update Restrictions

On a final note before concluding this post, it is worth mentioning that a new version (10.0.16) of the application was made available on Play Store while developing the attack chain, and the version under test (10.0.15) disallowed further usage until the application was updated. Bypassing this was necessary for the continuation of the analysis. However, note that both versions were also subject to the vulnerabilities described in this post. 

To locate the enforcing code, I started from the dialog shown when access was disallowed due to the update. This was found in file res/values/strings.xml within the application's resources:

```xml
<string name="popup_invalid_version_title">App version is invalid</string>
```

This string was cross-reference just once in the code, inside the class `com.zappware.nexx4.android.mobile.utils.r`:

```java
public static Dialog b(Activity activity, String str) {
    if (str == null) {
        str = activity.getPackageName();
    }
    return new AlertDialog.Builder(activity)
            .setTitle(R.string.popup_invalid_version_title)
            .setMessage(R.string.popup_invalid_version_message)
        .setCancelable(false)
        ....
```


Function `b()` seen above, was in turn called by a function of the  `com.zappware.nexx4.android.mobile.ui.startup.a.ai` class, which is annotated below:

```java
public void a(Activity activity, VersionCheckResponse versionCheckResponse) {
    String b2 = new u(Nexx4App.a()).b();
//parseInt (installed Version) - getVersion (upstream Version)  
    this.e = (b2 != null ? Integer.parseInt(b2) : -1) >= versionCheckResponse.getVersion();
    if (this.e) {
        // true
        // parseInt >= getVersion
        c(activity);
    } else {
        // false
        // parseInt < getVersion
        r.b(activity, versionCheckResponse.getPackageName()).show();
    }
}
```


The "if" clause effectively meant that the dialog would appear when the condition would be False. Modification of this condition at runtime using Frida was possible, however I opted for the application repackaging method instead, as a more elegant solution. The Wind Vision application was therefore re-compiled, re-signed and re-installed to the device to proceed with testing after editing just one line in the following smali file after disassembly:

```
# file smali_classes2/com/zappware/nexx4/android/mobile/ui/startup/a/ai.smali
.method private synthetic a(Landroid/app/Activity;Lcom/zappware/nexx4/android/mobile/data/remote/models/VersionCheckResponse;)V
    ...
# if-lt v0, v1, :cond_1
    if-ge v0, v1, :cond_1
```

The complete list of commands to perform this repackaging step in order for the update enforcement to be bypassed is attached below:

```bash
device$ pm list packages | grep -i wind# find package name
device$ pm path gr.wind.windvision# find APK path
host$ adb pull $APK_PATH# copy original APK
host$ apktool d $APK_PATH -o out# disassemble original APK
host$ # edit .smali file
host$ apktool b out -o repackaged.apk# re-assemble new APK
host$ apksigner sign --ks keystore.ks -o signed.apk repackaged.apk# re-sign APK
host$ adb install resigned.apk# re-install "backdoored" application
```

## Conclusion


To summarise, we demonstrated how an attacker could target Wind Vision users, takeover their accounts by combining exploitation of a series of security flaws, and stream free TV on their behalf.  All starting from one wrong click. This highlights the importance of secure app linking or inter-process communication in general, and how it can all go wrong when not implemented correctly. With this story concluded, here are some key takeaways to be considered:

For mobile application developers out there, issues such as the ones outlined above can be prevented by:

- Implementing secure URL-schemes using App Links, instead of Deep Links.
- Choosing the PKCE extension for Oauth2 flows in mobile apps.
- Employing randomness when generating unique identifiers to prevent re-creation.
- Generating, storing and checking local authentication credentials locally.

Finally, some general tips for us, mobile application users to protect our accounts against abuse. As always, we should make sure to: 

- Scrutinise applications before installation.
- Only choose approved stores and avoid third party sources. 
- Exercise caution when prompted to choose handler applications.