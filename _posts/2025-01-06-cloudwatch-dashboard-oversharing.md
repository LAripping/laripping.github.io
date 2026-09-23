---
layout: post
title: "CloudWatch Dashboard (Over)Sharing"
excerpt: "A security vulnerability in AWS CloudWatch dashboard sharing allowed unauthorized viewers to access EC2 tags.<br/><br/>"
categories:
  - "blog-posts"
tags:
  - aws
  - cloudwatch
last_modified_at: 2025-01-06T00:00:00
post-img: assets/img/get-the-tags.png
---

{% include note.html title="Disclaimer:" content="This article was originally published in [Reversec Labs](https://labs.reversec.com/posts/2025/01/cloudwatch-dashboard-oversharing) on 6 January 2025." %}

## Introduction

A documented, but little known security risk was introduced to an AWS account by a service often used for... security monitoring: Amazon CloudWatch. In short, **sharing a CloudWatch dashboard publicly could allow third-parties in possession of the link to read instance tags of EC2s within the account which shared the dashboard, and potentially execute Lambda functions.**

Our research revealed that the limited, yet preventable vulnerability in the default configuration was the result of a logic bug in the AWS Console, combined with a "fail open" condition in Amazon Cognito. These issues were disclosed to Amazon Web Services and a fix has been released. This post will aim to raise awareness about this relatively undocumented TTP for cloud initial access, and to provide an in-depth analysis of all aspects surrounding this topic, from discovery to impact evaluation, along with some interesting observations made along the way. Let's dive right in.

## Background

During a recent engagement assessing the security of a customer's AWS account, an issue was raised by a proprietary vulnerability scanning solution that related to a CloudWatch Dashboard, which was set up for monitoring performance metrics. The dashboard was shared publicly and the scanner's finding only echoed the warning in the [documentation page](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-dashboard-sharing.html) – that people with no access to the AWS account will obtain some permissions to it:

![](/assets/img/15ab7f3ba53d7a0093b8663c84a346be7d805d6a.png)

To quantify the true impact of this, we keep reading in the same documentation page and find the exact IAM permissions granted to people viewing the dashboard:

![](/assets/img/ff15fd53624e9d021a514d4487b209be4e596073.png)

CloudWatch Insights, Metric and Alarms are all expected data that a dashboard would normally expose. EC2 tags, on the other hand... sounded unexpected, and slightly more sensitive if true. Even though AWS guidance [advises against putting sensitive data in tags](https://docs.aws.amazon.com/AWSEC2/latest/UserGui%20de/Using_Tags.html) we have encountered many instances where organistations have used tags for this purpose anyway.

But how were these permissions given to dashboard viewers? Once a cloud administrator configures a CloudWatch dashboard as publicly accessible in the Console (note: and *only* via the Console - we'll come back to this later) they are granted a URL similar to the following, to share with people who want to view the dashboard:

```
https://cloudwatch.amazonaws.com/dashboard.html?dashboard=<NAME>&context=eyJSIjoidXMt....<REDACTED>....JsaWMifQ==&start=PT3H&end=null
```

Visiting this URL from any browser indeed, displays the dashboard and its widgets - after a few seconds of loading. No authentication or AWS credentials in sight.

![](/assets/img/175eaf26e48e79f2c25e226372ed81983e601e87.png)

This feature - although handy - set off some red flags immediately: Besides the lack of login enforcement for account-specific data, transferring data through URLs is never a good idea due to the number of different locations URLs are leaked, cached, indexed etc.

Going back to our question of how this works, the answer lay somewhere within this URL, and particularly this Amazon AWS endpoint it pointed to and the parameters passed. An analysis of the web application was in order.

## The Dashboard Application

At this point, it was apparent that this implementation was supported by a publicly facing Amazon AWS solution. Surprisingly however, no mentions about the security implications of this, exploitation instructions, or further detail was found anywhere on the internet, despite hard, *and* targeted Googling...

The first place to examine within that URL was the `context` parameter, as it looked like Base64-encoded JSON. Decoding it revealed the following cryptic (and undocumented) object:

![](/assets/img/2488f89aaa6a3dc85fffdffb75869f567b12c41a.png)

The semantics of some fields were obvious, while others needed some further digging:

- "R" looked like an AWS region, easily confirmed as the exact region that the dashboard was deployed within
- "M" also appeared to indicate the sharing mode, "Public" among the 3 options available
- "D" revealed the AWS account ID containing the dashboard
- "O" was the ARN of an IAM role within this account and particularly, that of a service role

The leakage of the account ID can pose a limited risk already, as it can generally allow [brute forcing](https://rhinosecuritylabs.com/aws/assume-worst-aws-assume-role-enumeration/) of IAM roles for assumable ones, although in this case any assumable roles would also need to have the same identity pool in their [trust policies](https://docs.aws.amazon.com/cognito/latest/developerguide/iam-roles.html#trust-policies). However, the potential to read EC2 tags was more significant, so we decided to dig a bit deeper.

Moving on from the URL, the next place to investigate was the mysterious "Loading" stage, taking a closer look at the API calls carried out behind the scenes. Time to fire up Burp!

Looking at the user agent for the XHR requests discloses another piece of useful info: The application was built using [AWS Amplify](https://aws.amazon.com/amplify/):

```sh
X-Amz-User-Agent:aws-amplify/5.3.6 framework/
```

Further examination of the loading flow confirmed that `dashboard.html` served as the *renderer* component: first receiving a "manifest" document, that defined the "widgets" comprising the dashboard. Then, through a few more requests, the browser was authenticated against the AWS account, allowing the fetching of runtime data to populate those widgets.

But what about the authentication? A quick Google search suggests that Amplify + Cognito is a typical combination for rapidly deploying simple client-side webapps – such as Single-Page Applications (SPAs) supporting user authentication. And this is exactly what happened in this application too. The following two request-response pairs captured revealed how the user authentication was handled by the dashboard application:

![](/assets/img/4f8a387e8b1c1420f7f36474bd2cc404d1f47ef9.png)
Step 1: CognitoIdentity:GetId Request & Response

![](/assets/img/300607c1b1de29d5a70754dcd37f26d07e179773.png)
Step 2: CognitoIdentity:GetCredentialsForIdentity Request & Response

What we're seeing here is a [Cognito-based Authentication Flow](https://docs.aws.amazon.com/cognito/latest/developerguide/authentication-flow.html) and particularly an **Enhanced (simplified) authflow**, which is a pattern for delivering temporary, limited-privilege AWS credentials to an application needing to access AWS resources. This simple sequence consists of 2 API calls, taking as input only an `IdentityPoolId` - which is specified in the "I" field of the `context` URL parameter, exchanging it for an `IdentityId` which in turn is traded for temporary AWS credentials.

Zooming out, and getting back into the shoes of an attacker with only a dashboard URL, looking to access the promised AWS resources of the account, we now have a straightforward exploitation flow which can be reproduced with the `aws` CLI:

```sh
$ aws  cognito-identity get-id --identity-pool-id "us-east-1:5207...0d" --region us-east-1
{
	"IdentityId": "us-east-1:3b5e1.....db8"
}

$ aws cognito-identity get-credentials-for-identity --identity-id "us-east-1:3b5e1.....db8" --region us-east-1
{
	"IdentityId": "us-east-1:3b5e1.....db8",
	"Credentials": {
		"AccessKeyId": "ASIA3...HO",
		"SecretKey": "Mv/3l0uum.....nj",
		"SessionToken": "IQoJb3J...UbIQQ1gc=",
		"Expiration": "2024-07-22T19:26:53+01:00"
	}
}
```

Performing the PoC `ec2:DescribeTags` API call using these credentials, however, leads to a new surprise:

![](/assets/img/9edc469f9518fe2ff2f9f57e49be9db891e413f2.png)

Unauthorised Operation? This was not the expected scenario... Some further investigation was needed, and the hint given in the error message pointed us to the obvious entity regulating access in AWS permissions, which had somehow not yet appeared in the equation: IAM roles.

## Let's Build a Lab

To increase our visibility into the previous steps of the process (the creation and sharing of a dashboard) we need to set up a lab and add some dynamic analysis into the mix. In a sandbox account under our complete control, we created a barebones CloudWatch dashboard and proceeded to hit the "Share" button in the Console. Once again, a hard-to-miss callout was presented:

![](/assets/img/0fbbd544bf0e6f785e20175461c65931758dc730.png)

The callout informed us of the set of resources that would be created as a result of this process:

- A Cognito Identity Pool
- A Cognito User Pool
- A Cognito App Client
- An IAM role

This simple action and result was quite a breakthrough, as it provided the following 3 observations:

1. All these resources are created automatically behind the scenes, by Amazon CloudWatch. No user involvement or opportunities to insert potentially insecure configuration are available.
2. The IAM policy for the dashboard role, was never mentioned, which suggested that as above, it was a fixed set of permissions - those 4 mentioned by the [documentation page](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-dashboard-sharing.html) , including `EC2:DescribeTags`.
3. The remaining 3 resources were all Cognito resources, confirming that all authentication used by the CloudWatch application is handled by Cognito, and Cognito alone.

Past this confirmation step and once the dashboard was eventually shared, the result was visible in the Console, along with yet another warning banner greeting us on every page visit, about the implications of sharing publicly:

![](/assets/img/ceded0829ad15f62e26f36721d963f3dbe6558c2.png)

Having complete visibility now into all solution components allows us to reflect on the implementation of the sharing feature, and particularly to revisit the mysterious `context` parameter.

Here's a reverse engineered listing of each field and its purpose:

| Field | Example Value | Description |
| --- | --- | --- |
| R | `us-east-1` | Region of Cognito resources |
| D | `cw-db-112233445566` | Subdomain of the Cognito User Pool |
| U | `us-east-1_AaBb45dde` | Cognito User Pool ID |
| C | `e18aipaaaabbbbakdm7rc56kk` | Cognito Identity Provider Client ID |
| I | `us-east-1:ab123456-1234-4567-89ab-1234567890ab` | Cognito Identity Pool ID |
| O | `arn:aws:iam::112233445566:role/service-role/CWDBSharing-PublicReadOnlyAccess-DSTM21S9` | IAM Role for the Dashboard in question |
| M | `Public` or `UsrPwSingle` or `SSO` | Sharing type:  Public, Username & Password, SSO respectively. |

This complete visibility now also allowed us to understand and draw out the complete viewer flow:

![](/assets/img/32efa3405f1524504f3dd4c665dfa8c0c856ebd0.png)

1. User visits a Dashboard link carrying the Dashboard Name and Context as URL parameters

2. The Dashboard application's client-side code is retrieved from Amazon CDN

3. The application obtains temporary credentials from Cognito, through the Enhanced authflow

```sh
CognitoIdentity:getId( identityPoolId )
CognitoIdentity:getCredentialsForIdentity( identityId )
```

4. An authenticated request retrieves the Dashboard's Manifest, defining the alarms and metrics resources

```sh
CloudWatch:getDashboard( dashboardName )
```

5. Finally, the data to populate each view is retrieved, through CloudWatch API calls to the respective resource regions

```sh
CloudWatch:describeAlarms( alarmNames,alarmTypes )
CloudWatch:getMetricData( defaults,metrics )
...
```

While these bits of insights are useful, they still didn't explain the issue at hand: **Why was the Cognito-retrieved identity not granted the full set of permissions originally intended by the IAM policy?**

After some more digging, we eventually bumped into an [issue thread](https://github.com/aws/aws-sdk-js/issues/4303) on the open-source, AWS

JavaScript SDK project, initiated by a user experiencing the same issue. After a few replies, the SDK maintainer provided a surprising resolution:

![](/assets/img/239461804c11dd3aedbae43e4d835302c0139df7.png)

This was arguably an interesting observation: It appeared that behind the scenes, the Simplified flow was implemented to deliberately ignore admin-assigned IAM policies, and instead silently grant reduced permissions.

Now knowing where to look, we find out further justification of this design decision, on the linked [documentation page](https://docs.aws.amazon.com/cognito/latest/developerguide/iam-roles.html):

> *"For **additional security protection**, Amazon Cognito applies a **scope-down policy** to credentials that you assign your unauthenticated users in the [enhanced flow](https://docs.aws.amazon.com/cognito/latest/developerguide/authentication-flow.html), using `GetCredentialsForIdentity` . The scope-down policy adds an [Inline session policy](https://docs.aws.amazon.com/cognito/latest/developerguide/iam-roles.html#access-policies-inline-policy) and an [AWS managed session policy](https://docs.aws.amazon.com/cognito/latest/developerguide/iam-roles.html#access-policies-managed-policy) to the IAM policies that you apply to your unauthenticated role."*

This finally explains the original error thrown when trying to retrieve EC2 tags. If we revisit policies and permissions in AWS IAM, the effective set of permissions that are granted to any session are those in the intersection of Identity-based, Resource-based, and Session policies. In the context of dashboard sharing now: The dashboard’s IAM role is granted all 4 permissions (Identity-based policy), and the tags themselves do not restrict listing from that principal (Resource-based policy). However, the viewer’s browser is effectively only granted 3 of these 4 permissions. This is due solely to the Session policy defined by the client:

![](/assets/img/70403b0ad8955e3165359e8fe45f46a832933977.png)

Hypothetically, if the application was using the Basic (Classic) Flow, no scope-down would be applied, and the intersection would include all 4 permissions defined in the auto-configured dashboard role policy:

![](/assets/img/8881ab451fa9f7bafca7e4dd025b1b2b72573339.png)

It should be noted that using the Basic flow would be possible for the dashboard application. Besides contradicting Amazon's own recommended design patterns, this would require a call to Amazon STS by the client-side code, to retrieve temporary tokens directly via `sts:AssumeRoleWithWebIdentity` . This would require specifying the desired IAM role, which is however known to the application via the "O" field of the `context` parameter.

From an exploitation perspective, the question now becomes: Can an unauthenticated dashboard viewer initiate a Basic authflow against the Cognito resources?

### Getting the Tags

Armed with knowledge of the internals, the path needed to carry out this attack was now

straightforward. For an attacker in possession of only a shared dashboard URL, the steps to carry out a Basic authflow against the dashboard's Cognito identity pool and access the account's EC2 tags were the following:

![](/assets/img/59dd0c9ffb12ac717c4a35c5f32ac1d20c724b94.png)

1. Attacker extracts the following from the Context

- IAM role ARN
- Identity Pool ID

2. Attacker acquires an Identity from the Cognito Identity pool, as before

```sh
cognitoIdentity:getId( identityPoolId )
```

3. Attacker requests an OpenID Connect (OIDC) token for this identity

```sh
cognitoIdentity:getOpenIdToken( identityId )
```

4. Attacker trades the OIDC token for temporary credentials of the target IAM role

```sh
sts:assumeRoleWithWebIdentity( dashboardRole, oidcToken )
```

5. Finally, attacker proceeds to leverage the role's permissions by reading EC2 tags

```sh
ec2:describeTags()
```

Time to put this theory to the test. Setting aim at the sandbox Cognito resources within our WithSecure lab account, we proceed to issue the commands:

1. Retrieving a Cognito Identity using the Identity Pool ID within the context parameter (field "I")

![](/assets/img/1a2fddd9bfa850c302746ba83722479937b348c3.png)

2. Obtaining an OpenID Token for this identity

![](/assets/img/79376b2b9de0578f61716fa2462f111f57f7942f.png)

3. Exchanging this token for AWS credentials, specifying the IAM role worked out from the "O" field of the context parameter

![](/assets/img/d24b31bbcf86862c67ceb09e7a1b519f1b6f81e7.png)

4. Loading these credentials for the final PoC command... et voilà!

![](/assets/img/6f444d808e8cc2c48a9394e035d1359970c71158.png)

And that confirms the exploitation scenario end-to-end: We went from dashboard viewers, to listing EC2 tags. All by just knowing a URL, and URLs like this are not hard to find...

## Hunting for Exposed Dashboards

But how common is this Dashboard Sharing functionality really? And which sharing method was the most prevalent? Answers were easily given using Google dorks. For example, the following query filtered out dashboards with SSO authentication:

```bash
inurl:https://cloudwatch.amazonaws.com/dashboard.html -corporate
```

This revealed no more than 20 or so results:

![](/assets/img/1390714c7626803ecdbb7dc40192ab5931bb10ee.png)

A few more unique dashboards were found in [URLscan](https://urlscan.io/search/#page.domain%3Acloudwatch.amazonaws.com):

![](/assets/img/e643029057e434465431209db95a0bb9437c17ac.png)

Clicking through a few of them indeed revealed dashboards of all sorts, with real time data flowing throughout the widgets:

![](/assets/img/9d066f832fb2dcffb66a78d4940999fc0ace16f2.png)

Without, of course going all the way to list EC2 tags in those third party accounts, a few observations can be made simply by looking at those results:
- The naming format of all resources mentioned within the `context` parameter was consistent, including the IAM role ARN (field "O"): `"CWDBSharing-PublicReadOnlyAccess-[0-9A-Z](8)"`
- Alarmingly, the number of dashboards shared publicly were far more than the sum of those shared with SSO authentication and those requiring username + password... Once again, users are shown to choose the easy way instead of the secure alternative. Not so surprising.

## Is This Really That Bad?

Listing EC2 tags is arguably not the end of the world. Amazon's guidance [clearly advises](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Using_Tags.html) against putting sensitive data in tags. However, in our assessments we have found in there sensitive information of various forms, such as PII, owner contact details and sometimes even credentials.

Looking for ways this could cause more serious problems, we first explored potential scenarios where the IAM role of the dashboard would be more privileged than these use cases we've already seen. As previously established, these IAM permissions were fixed and configured by Amazon code, so it was unlikely that users would manually modify "canned resources" that "just work". Unless they are after more features!

Further reading in the [docs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-dashboard-sharing.html#share-cloudwatch-dashboard-logwidget) indeed revealed two scenarios where the dashboard's IAM role would need to be manually granted extra permissions by the user for certain functionality to work:

1. **Logs Table Widgets** - resulting in the addition of a few more, (hopefully) resource-scoped, CloudWatch permissions

```sh
logs:FilterLogEvents
logs:StartQuery
logs:StopQuery
logs:GetLogRecord
logs:DescribeLogGroups
```

2. **Custom Widgets** – leading to the juicier, `lambda:InvokeFunction` permissions

Regarding the former scenario, we read in [this blog post](https://web.archive.org/web/20200916042808/https%3A//dev.classmethod.jp/articles/amazon-cloudwatch-dashboards-supports-sharing/) from 2020, that these `logs:` permissions used to be included in the dashboard's IAM role policy by default, and were only later made optional. Another note here is that performing `logs:` actions on behalf of the dashboard role does not require the Basic authflow sequence we demonstrated above. The "scope-down" [Session Policy](https://docs.aws.amazon.com/cognito/latest/developerguide/iam-roles.html#access-policies-inline-policy) includes these additional permissions, therefore the credentials obtained by the browser itself through the Simplified authflow would also do the trick.

As for the latter scenario, we finally had a truly impactful outcome in the cards: conditional, but unauthenticated Lambda function execution. In other words, viewers of a dashboard with custom widgets can *programmatically* invoke the associated function.  An attacker could use this to run up the victim’s AWS bill, also known as a [Denial of Wallet](https://summitroute.com/blog/2020/06/08/denial_of_wallet_attacks_on_aws/) attack. Through this Lambda backend, a custom widget can also allow execution of technically any AWS API, through the [cwdb-action](https://github.com/aws-samples/cloudwatch-custom-widgets-samples?tab=readme-ov-file#is-interactivity-possible-in-the-returned-html), however once again, this would require further configuration.

Finally, we tried a few different ways to take this attack further, by attempting IAM attacks such as:

- specifying a more privileged policy on the `sts:AssumeRoleWithWebIdentity` step of the programmatic Basic authflow
- assuming a different IAM role altogether,
- going cross-account, and trying to assume the dashboard's role from a foreign Cognito Identity in the Simplified authflow's `CognitoIdentity:getCredentialsForIdentity` - inspired by Nick Frichette's [similar research](https://securitylabs.datadoghq.com/articles/amplified-exposure-how-aws-flaws-made-amplify-iam-roles-vulnerable-to-takeover/) against AWS Amplify

…All without success. This last one in particular, was prevented here by the additional `Condition` added automatically to the dashboard role's Trust Policy, restricting assumption to a single Cognito Identity universally:

```json
{
  "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Federated": "cognito-identity.amazonaws.com"
        },
	"Action": "sts:AssumeRoleWithWebIdentity",
	"Condition": {
	  "StringEquals": {
	    "cognito-identity.amazonaws.com:aud": "us-east-1:7d60b318-a9ed-4043-a426-fdc0e126ef63"
	  },
	  "ForAnyValue:StringLike": {
	    "cognito-identity.amazonaws.com:amr": "authenticated"
	  }
        }
      }
    ]
}
```

## Root Cause Analysis

While reading up on typical attacks against Cognito resources in those previous attempts to escalate further, we came to a realisation: **this vector could have been prevented.** It is possible to [block Basic authflow](https://docs.aws.amazon.com/cognitoidentity/latest/APIReference/API_UpdateIdentityPool.html#CognitoIdentity-UpdateIdentityPool-request-AllowClassicFlow) on a Cognito Identity Pool altogether, and indeed our dashboard sharing one had it blocked, as seen in the Console:

![](/assets/img/304ac1dca4b8209df81bdaa3257ef8a9e3a21dd4.png)

Or had it? If we look closely at the image above, we see the value of the relevant field is in fact a hyphen! Not a clear-cut "No" or "Inactive" - more like a "N/A". Could a Boolean flag have three possible values instead of two? Querying the resource programmatically confirms our hypothesis:

![](/assets/img/ab14f5dd1cb8c48d035f8fb9a70ae5f839a0062e.png)

Notice how the `AllowClassicFlow` field is missing from the JSON object…

What we're seeing is a vulnerability known as "Fail Open" behaviour. In effect, a system insecurely skipping its controls once faced with undefined conditions. In our case, if users would omit to explicitly set the `AllowClassicFlow` flag, it would be left in an uninitialised state, which effectively translated to `True` , subsequently allowing OpenID tokens to be retrieved via the Basic (classic) authflow. Through communications with AWS, it was revealed that this behaviour was historically chosen for backwards compatibility, when the Enhanced flow was introduced, to avoid breaking customers' exisiting solutions when creating new identity pools.

Note that the same "undefined" setting was seen in Identity pools created when dashboards were shared with username & password authentication. In fact, closer inspection of the sharing process (with both authentication options - username & password or public sharing) revealed that the API calls to create the Cognito resources were taking place client-side, in the browser. And this included the vulnerable Cognito Identity Pool. Opening the inspector allowed us to observe the `CreateIdentityPool` operation carried out by the Console's JavaScript SDKs, and confirmed our finding - the `AllowClassicFlow` field was not set:

![](/assets/img/01462dc990a56721f2c169d53aaba38709df5efd.png)

At that point, we had finally solved the riddle of the EC2 tag exposure: The combined effect of a misconfigured Cognito Identity pool created by the Console SDK, and the undefined behaviour of a Boolean attribute when creating such resources, resulted in Basic authflow being unintentionally allowed, and thereby providing viewers of a CloudWatch dashboard with access to EC2 tags within the affected account.

For completeness, before raising the findings with Amazon, we cross-checked for this undefined behaviour across the different methods users deploy cloud resources with. The most common ones, at least:

- the Web Console
- the SDKs
- the AWS CLI
- Terraform

Note that this aimed at understanding how these methods instantiated Cognito Identity Pools, and particularly, whether they set the `AllowClassicAuth` flag or not. Not how they shared dashboards.

Sharing a dashboard was - and remains - only possible via the Console, despite [a Reddit reply](https://www.reddit.com/r/aws/comments/jfwiwl/comment/g9n3mhc/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button) by a CloudWatch developer around the feature release date, which suggested that programmatic dashboard sharing would be implemented in the future:

![](/assets/img/44f170d83f5e671b89b0af83acc1deb97ac24708.png)

### Affected Clients

Starting from the Console, we went ahead and created a new identity pool from the web UI, walking through the multi-step process. As seen below in the second to last step, we are asked explicitly whether to enable Basic (Classic) auth or not:

![](/assets/img/07918e91ac8405b9a53c82fa9e2b15284f14a64f.png)

Interestingly, leaving this field empty here, securely defaults to a `False` setting:

![](/assets/img/4d6c043007f734cb1de7450e11c64b77d2bff305.png)

And this indeed results in the expected error upon attempting to retrieve OpenID tokens:

![](/assets/img/9f8424e036160df3fa220eb768f44d59b9246a4f.png)

Moving over to the CLI. Behind the scenes, the AWS CLI uses the Python boto3 library which is the official Python SDK, and both these technologies generate class definitions on the fly, according to the latest upstream API version. The version at the time of testing also demonstrated the behaviour observed within the dashboard sharing process: Creating a Cognito Identity Pool from the CLI did not require the `--allow-classic-auth-flow` flag to be set, and resulted in the field to be uninitialised in the resulting object:

```sh
$ aws --version
aws-cli/2.17.17 Python/3.11.9 Linux/5.10.102.1-microsoft-standard-WSL2
exe/x86_64.ubuntu.20

$ aws cognito-identity create-identity-pool --identity-pool-name idpool-CLIcreated
--allow-unauthenticated-identities
{
  "IdentityPoolId": "eu-west-1:8f...-a25cf3bd5720",
  "IdentityPoolName": "idpool-CLIcreated",
  "AllowUnauthenticatedIdentities": true,
  "IdentityPoolTags": {}
}

$ aws cognito-identity describe-identity-pool --identity-pool-id "eu-west-1:8f...-a25cf3bd5720"
{
  "IdentityPoolId": "eu-west-1:8f...-a25cf3bd5720",
  "IdentityPoolName": "idpool-CLIcreated",
  "AllowUnauthenticatedIdentities": true,
  "IdentityPoolTags": {}
}
```

This issue had been [raised before](https://github.com/aws/aws-cli/issues/7652) - more than a year earlier, in February 2023 – where the last human comment mentions the expected behaviour. After no response or resolution however, the ticket was closed due to inactivity.

And finally, Terraform. As per the [documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cognito_identity_pool#allow_classic_flow), the `allow_classic_flow` argument although optional, safely defaults to False when defining an `aws_cognito_identity_pool` resource:

![](/assets/img/682f630e29304db4ec594b1c93ebd96ffe5ff0f4.png)

In summary, the default behaviour for each client is provided below:

| | "Allow Classic Auth" default |
| --- | --- |
| from Console | False unless explicitly enabled |
| from CLI/SDKs | Uninitialised, effectively True |
| from Terraform | set by TF to False by default |

## Mitigating Controls

So far we have focused only on the least secure but most common option of public sharing. What about the other forms available? Are dashboards shared with username and password, or SSO authentication, similarly allowing EC2 tag exposure (and potentially `logs` actions, or Lambda function invocations)?

### Username & Password Sharing

Looking at username and password sharing first, we see a similar Warning dialog when sharing a dashboard to an email address:

![](/assets/img/11d6d1e2b46e82e6ea626d201b0013d75eaa095a.png)

The generated dashboard URL and encoded `context` parameter looks no different, except the “M” field:

![](/assets/img/9c9378c46d34364a03b0cd097e8706e0bc20a9cb.png)

Trying to reproduce the attack by initiating a Basic authflow via the AWS CLI, using just the Identity Pool ID expectedly fails, given the Cognito Identity Pool no longer supports Unauthenticated access:

```bash
$ aws cognito-identity get-id --identity-pool-id "us-east-1:b7aa...-4f04a4b6a6ff" --region us-east-1
An error occurred (NotAuthorizedException) when calling the GetId operation:
Unauthenticated access is not supported for this identity pool.
```

But even if we skip that step by somehow obtaining an Identity ID, then we *still* can't retrieve a token via Basic authflow, that would allow us to set a permissive session policy and list tags:

```bash
$ aws cognito-identity get-open-id-token --identity-id "us-east-1:3b5...-b707-06ce515fb598" --region us-east-1
An error occurred (InvalidParameterException) when calling the GetOpenIdToken operation: Basic (classic) flow is not supported with RoleMappings, please use enhanced flow.
```

Awesome, unauthenticated third parties can no longer list EC2 tags. But what about intended viewers?

While Basic authflow is disabled, it is no longer needed, as the Console's frontend code this time explicitly specifies the service-role's IAM Policy in the `CognitoIdentity:GetCredentialsForIdentity` API call, avoiding the "scope-down" policy of the Simplified authflow. That means that the temporary credentials retrieved by the browser are enough to list the EC2 tags.

![](/assets/img/40c65b8bda9c71d52204d279edfa3027701a781b.png)

### SSO Sharing

And what about SSO sharing? In our brief hunt, we found this method to be the next most common after that of public sharing. This scenario is applicable to multi-account AWS Organizations, and leverages the IAM Identity Center service to expose the shared dashboard as an “Application” to assigned users or groups:

![](/assets/img/60f8edfe48a61ccd0c8e9542bc5a48df4b50f9db.png)

Following [the guides](https://aws.amazon.com/blogs/security/how-to-create-and-manage-users-within-aws-sso/) we eventually managed to set up a lab environment to trial this out. Starting from the dashboard URL as always, we notice little difference to all previous sharing methods:

![](/assets/img/3d086a76902ec9bb39d04bd0bc978bf440f118c7.png)

Once again, initiating a Basic authflow against the Identity Pool is not possible and neither is obtaining an Identity from an unauthenticated perspective via `get-id`. However, using the temporary credentials retrieved by the browser as above, we can still list EC2 tags and conditionally invoke Lambda functions or retrieve CloudWatch Logs. Investigating why this is possible here, points us to the `id_token` retrieved in a previous step, by the SAML provider the administrator configured for SSO.

![](/assets/img/149d2a490b5228372d79e278379f418d9709a266.png)

The role ARN specified in this token is passed in the `CognitoIdentity:GetCredentialsForIdentity` API call within the `Logins` parameter, and is effectively the provider’s “suggestion” to Cognito of the preferred IAM role that the requesting user must be granted. This communication between the different components is depicted across steps 1 and 2 in the flow diagram attached to the guide linked above.

![](/assets/img/b42b9913b40445981d8b9808bf39bad037df2d8b.png)

And how was this role set in first place? [Documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/saml-identity-provider.html) states that "Amazon Cognito identifies the group with the lowest precedence value in a user’s set of groups, and writes the IAM role ARN associated with that group to the `preferred_role` claim". And indeed, this @gmail.com user that the dashboard was shared to, was automatically added to a group, with that particular role configured:

![](/assets/img/ada1ba7f03a93d38b49e0ce7a7c6edb7e192ee97.png)

It is worth noting that all of this configuration was done automatically by the Console, when the dashboard was shared. The API calls made by the JavaScript code proceeded to create, or modify the existing Cognito resources, and set this role mapping - all without any administrator interaction.

So summarising the exposure for all sharing methods:

| Dashboard shared with | EC2 tags (/Lambda/CW Logs) exposure | |
| --- | --- | --- |
|  | 3rd Parties | Intended Viewers |
| Public Sharing | Yes | Yes |
| Username and Password Sharing | No | Yes |
| SSO Sharing | No | Yes |

It is worth noting that we tested for other typical Cognito attacks applicable for this type of sharing namely Self-Registration, and User Attribute modifications. However none resulted in meaningful exploitation scenarios.

### Unsharing Dashboards

Before Amazon fixed the issue by blocking Basic authflow (detailed in the next section), we took a moment to investigate whether “unsharing” a dashboard fully mitigated the exposure.

Our testing indicated that unsharing a dashboard will remove some, but not necessarily all of the relevant Cognito resources. The lifecycle of each resource type differs according to the state of the environment. For example, every time a new dashboard is shared, a new Identity Pool is created. However, a single User Pool persists any dashboard shared in any way. The progression of all related resources between the different states is depicted in a Gantt chart below, also covering the dashboard IAM role – the only non-Cognito resource involved:

![](/assets/img/116e7046a698ed760e3de3145f26be0b228c95bf.png)

## Remediation

The issues in the AWS Console and the Cognito service, along with their resulting impact on user resources, were confidentially disclosed to AWS in late July 2024, through their vulnerability reporting [program](https://aws.amazon.com/security/vulnerability-reporting/). The security team was swift in their responses and in early September, we were notified that a fix had been rolled out on August 28, 2024. This fix disabled basic flow and ensured that calls to `ec2:DescribeTags` were no longer permitted in a shared dashboard. This resulted in EC2 instance names not being shown in dashboard widgets.

Our retesting confirmed that the Console API calls creating the new Cognito Identity Pools upon sharing of a dashboard, were now indeed setting the `AllowClassicFlow` flag explicitly to `False`.

![](/assets/img/c55c150e7528cdd3e044c4f8d0c714d66e5cb25d.png)

This was the case for all sharing methods, including Username & Password and SSO.

## Disclosure Timeline

The detailed timeline of the disclosure is provided below:

| **Date** | **Detail** |
| --- | --- |
| 26/07/2024 | Initial contact with AWS Security Team |
| 31/07/2024 | AWS Security Team responds stating they are actively working on a fix. Publication of the research is also discussed |
| 01/08/2024 | WithSecure informs AWS Security Team of intentions to publicise research |
| 10/09/2024 | AWS Security Team notifies WithSecure of the fix deployed, requests to collaborate on publication to ensure technical accuracy |
| 30/09/2024 | WithSecure shares initial version of the blog post with AWS Security Team |
| 25/10/2024 | Feedback received and additional clarifications proposed |
| 16/01/2025 | Blog post published on WithSecure Labs |

## Summary

Overall, we saw that sharing CloudWatch dashboards provides viewers with some permissions over the source account. In the most common sharing method observed on the internet - that of no authentication - a now-fixed bug allowed viewers to enumerate the EC2 tags within the source account. Should additional functionality be enabled, and customers have manually modified IAM policies to grant the additional permissions, then viewers could also invoke Lambda functions and retrieve CloudWatch logs. Although unauthenticated viewers can no longer perform these actions without further knowledge of identifiers within the account, intended viewers are still granted the respective permissions within the account. Carrying out these actions is therefore trivial, simply by extracting the credentials obtained on their behalf by the web browser.

To protect their environments from external threats, customers using shared dashboards should review whether this functionality - and any additional features such as custom widgets - are required for their operations, given the risk posed.

Administrators should also review which entities are granted the following permission "bundle", as it can allow them to share Dashboards and introduce the risks described in this post.

- `iam:CreateRole`
- `iam:CreatePolicy`
- `iam:AttachRolePolicy`
- `iam:PassRole`
- `cognito-idp:*`
- `cognito-identity:*`

Finally, when modifying manually the IAM policy created by the Console to use these additional dashboard features, care should be taken to limit the resources in scope for these additional permissions – avoiding wildcard operators ("\*").
