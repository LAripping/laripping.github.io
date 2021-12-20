---
layout: post
title: 'Native Enrich: Scripting Ghidra and Frida to discover "hidden" JNI functions'
excerpt_separator: "<!--more-->"
categories:
  - "blog-posts"
tags:
  - reversing
  - native
  - android
  - ghidra
  - frida
last_modified_at: 2021-12-20T17:46:00
---

Recently, while doing some Android reverse engineering, I bumped into the following problem: After digging into the decompiled Java code, my analysis for the functionality of interest led me to a native function... which was nowhere to be found in the disassembly of the respective .so library. What sorcery is this? And where on god's green earth is my native function in that massive ELF blob?

In this blog post I will therefore:

* Explain why sometimes JNI functions don't pop up in the Function Listing, and most importantly
* Present a novel technique to "enrich" the disassembly with these functions' signatures ...wait for it... using Frida!  

<!--more-->

{% include toc.html %}


## The problem

> To avoid confusion, in the text that follows we'll be using the term "native methods" when talking about the Java code, and we'll be referring to them as "native functions" when talking about their implementation in the C code.   

Let's take it from the beginning.   

Let's assume that we're reversing an Android application, hunting down certain functionality e.g. the sending of a message. Our analysis has led us to a native method in the decompiled Java code, looking something like this: 

```java
package com.app.jni;

public class MessageWrite{
    ...
}
   
public class PhoneControllerHelper {
	...  
    @Override 
    public native boolean handleSendMessage(MessageWrite messageWrite);
```

...and we can somehow guess that this function is implemented in the `libEngineNative.so` library, from the few libs that the app packs in it's `lib/` directories. How? Using the power of `grep` of course.

The next move is to drop the lib into Ghidra, our dissassembler of choice, make ourselves a [Frappé](https://en.wikipedia.org/wiki/Frapp%C3%A9_coffee) while the auto-analysis runs, and eagerly filter for "handleSendMessage" on the Function Listing once it's finished and we've had a sip.  But we're in for a suprise: 

![](/assets/img/nofun.png)

Why didn't Ghidra find the function? And if a disassembler couldn't, how can the Android JVM during runtime? I was always a bit weak on native code RE, so I decided it was a good chance to get good, and after reading through [Maddie Stone's excellent tutorial](ttps://www.ragingrock.com/AndroidAppRE/reversing_native_libs.html) on the subject, I realized this is known as "Dynamic Linking" (or maybe this is Static Linking, the text gets kinda fuzzy at this point, I'm still not sure :sweat_smile:) where basically the native code declares *itself* the native functions that should be exported, *during runtime*, using the `RegisterNatives` JNI API. This is contrary to (what makes more sense as) "Static Linking", where the symbols are declared beforehand, by following naming rules which would lead to a familiar "Java_com_app_jni_..." function shown in the Listing. 

Here's how example C code to perform this Dynamic Linking would look like, as per the [Android documentation](https://developer.android.com/training/articles/perf-jni#native-libraries):

```c
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    JNIEnv* env;
    // A common version check
    if (vm->GetEnv((&env), JNI_VERSION_1_6) != JNI_OK) {
      return JNI_ERR;
    }

    // Find the classname 
    jclass c = env->FindClass("com/app/jni/PhoneControllerHelper");
    if (c == nullptr) return JNI_ERR;

    // Specify it's signature and fnPointer
    static const JNINativeMethod methods[] = {
      {"handleSendMessage", "(Lcom/app/jni/im2/MessageWrite;)Z", <void*>(handleSendMessage)}
    };
    
    // Register the class's native method
    int rc = env->RegisterNatives(c, methods, sizeof(methods)/sizeof(JNINativeMethod));
    if (rc != JNI_OK) return rc;

    return JNI_VERSION_1_6;
}
```

But although the strings "handleSendMessage" and "(Lcom/app/jni/im2/MessageWrite;)Z" are indeed found in the dissassembly (Search > Memory > As String), none of them are cross-referenced to the location of the registration, where we could look for the valuable third argument to the API call, the function pointer which would lead us to the actual body of the function.... 

So it looks like we need 2 things here to proceed with our analysis.

1. A runtime dump of arguments passed to the `RegisterNatives`  API, on all possible invocations
2. A way to save this information back into the disassembly, including the JNI type definitions of the newly found signatures 

Essentially, this translates to a Frida hook for the first step and Ghidra script for the second. Let's roll :metal:



## The Frida part 

For a start, we need a way to identify the `RegisterNatives` API itself before we're able to hook it. After playing in the Frida interpreter with methods like `Process.enumerateModules` and `Module.enumerateSymbols`, I came up with the final programmatic way to locate the APIs runtime address

```js
let addrRegisterNatives = null

Process.enumerateModules().forEach(function (m) { 
    Module.enumerateSymbolsSync(m.name).forEach(function (s) { 
        if (s.name.includes("RegisterNatives") && (!s.name.includes("CheckJNI"))) { 
            console.log(m.name, s.name)
            addrRegisterNatives = s.address
        } 
    }) 
})
```

Note that the second  "CheckJNI" condition filters out a similarly named API alongside our target one, which is irrelevant. From the simple code above we find that our API's mangled name is `_ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi` and it's located within the system  `libart.so` library, which kinda makes sense. 

Now we can proceed to `Interceptor.attach` this location, "translating" and dumping arguments on each call. We add the following to our Frida JS code, just to print the info for this first time:

```js
Interceptor.attach(addrRegisterNatives, {
    // jint RegisterNatives(JNIEnv *env, jclass clazz, const JNINativeMethod *methods, jint nMethods);
    onEnter: function (args) {
        var calledFromLibnOffset = String(DebugSymbol.fromAddress(this.returnAddress))
        var nMethods = parseInt(args[3]);
        console.log("\nenv->RegisterNatives()")
        console.log("\tnMethods="+nMethods);
        
        var class_name = Java.vm.tryGetEnv().getClassName(args[1]);
        console.log("\tclazz.name="+class_name)
        
        console.log("\tmethods[]:");
        var methods_ptr = ptr(args[2]);
        
        for (var i = 0; i < nMethods; i++) {
            var name_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize*3));
            var methodName = Memory.readCString(name_ptr);
            var sig_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize*3 + Process.pointerSize));
            var sig = Memory.readCString(sig_ptr);
            console.log("\t\t"+methodName+"(), sig:", sig)
            var fnPtr_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize*3 + Process.pointerSize*2));
            var find_module = Process.findModuleByAddress(fnPtr_ptr);
            var fnPtr_ptr_ghidra = ptr(fnPtr_ptr).sub(find_module.base).add(0x00100000)
            console.log("\t\t\tfnPtr:", fnPtr_ptr,  " ghidraOffset:", fnPtr_ptr_ghidra);
        }

    }
})
```

Running this we get a lot of output, as was expected, from the trove of which we isolate the message of interest:

```
env->RegisterNatives()
	nMethods=1
	clazz.name=com.app.jni.PhoneControllerHelper
	methods[]:
		handleSendIM2Message(), sig: (Lcom/app/jni/MessageWrite;)Z
			fnPtr: 0x733a924280  ghidraOffset: 0x1d7280
```

As can be seen from the data type fiddling in the JS code, we also print the function's offset within Ghidra, starting from the function's pointer passed to the "methods" argument. Navigating to this address in the Ghidra disassembly for verification, it looks like our assumptions are correct, there's a function start at this offset:

![](/assets/img/identified.png)



Right, so let's tweak this JS code slightly, to populate a full array of JSON objects, one for each native method registered during runtime, with its name as the Key, and its Ghidra address as the Value. The final version of the script can be found [on Frida Codeshare](https://codeshare.frida.re/@LAripping/trace-registernatives/):  

Then, after spawning the app with Frida hooked and following a few seconds of wait, we type `nativeMethods`  on the Interpreter to get the full array:

```bash
[Device::com.app]-> nativeMethods
{
    "methods": [
        {
            "ghidraOffset": "0x1d7280",
            "methodName": "com.app.jni.PhoneControllerHelper.handleSendMessage"
        },
        {
            "ghidraOffset": "0x1df3ac",
            "methodName": "com.app.jni.PhoneControllerHelper.handleResult"
        },
        ...
```

We copy this output and place it in a file named e.g. [nativeMethods-frida.json](nativeMethods-frida.json). Time to switch context :hourglass:



## Interlude - an ode to Ayrx

Earlier, while studying Maddie's tutorials for native Android reversing, i bumped into a project which appeared pretty useful.  Ayrx's [JNIAnalyzer](https://github.com/Ayrx/JNIAnalyzer/tree/master) not only seemed quite relevant to our challenge (yet, not exactly the solution) but also looked like a good Ghidra script template due to this similarity in purpose. I therefore ~~used this code as inspiration~~ shamelessly copy-pasted its Java code to form the basis of my Ghidra script. But not its current version!... 

Apparently the latest version of Ayrx's script asks for a whole APK , decompiles it on the fly with jadx and parses it to identify native functions. Then it tries to locate them in the disassembly ("Code Listing" in Ghidra speak) by looking for  "Java_..." names. Obviously, this wouldn't work here as we're interested in the elusive Dynamically registered functions which have no particular name in Ghidra since they're not exported. The script however is also valuable in that it then "enriches" the dissassembly by creating function signatures, and retyping both the arguments and return values appropriately, in the form of standard [JNI types](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/types.html) (e.g. `jint`, `jobject`, etc...). Ghidra can "learn" these types straight-away, by importing the [jni_all.gdt](https://github.com/Ayrx/JNIAnalyzer/blob/master/JNIAnalyzer/data/jni_all.gdt) file into the "Data Type Manager", although the script has code that does this programmatically. 

But to import a whole APK, decompile it, and parse it within a Ghidra script is a rather cumbersome solution, and -as I found out- unrealistic for production apps, which will easily break Ghidra's JVM with OOM errors after lengthy waiting times. Luckily, [a previous version](https://github.com/Ayrx/JNIAnalyzer/commit/102116182a602bab8f5f901f1ef168dcd6aa955b) existed which just asked for the JSON output of a previous step, involving the execution of a [JAR file](https://github.com/Ayrx/FindNativeJNIMethods/releases/download/0.4/FindNativeJNIMethods.jar) against the APK. This first step was an "offline" stage so to speak, involving no Ghidra whatsoever.  

The script turned out to do a wonderful job of listing all native functions of even large APKs, but most importantly *converting them to JNI format*, before exporting them to the JSON file as can be seen below:

 ```bash
$ java -jar FindNativeJNIMethods.jar base.apk nativeMethods-jar.json
$ less nativeMethods-jar.json
{
    "methods": [
        ...
        {
            "methodName": "com.app.jni.PhoneControllerHelper.handleSendMessage",
            "argumentSignature": "Lcom/app/jni/MessageWrite;",
            "argumentTypes": ["jobject"],
            "returnType": "jboolean",
            "isStatic": false
        }, {
            "methodName": "com.adjust.sdk.sig.NativeLibHelper.nSign",
            "argumentSignature": "Landroid/content/Context;Ljava/lang/Object;[BI",
            "argumentTypes": ["jobject", "jobject", "jbyteArray", "jint"],
            "returnType": "jbyteArray",
            "isStatic": false
        }, {
        ...
 ```

Now we need a Ghidra script to merge the two JSON files and define all these functions in the disassembly. 



## The Ghidra part

To get started with Ghidra scripting I read a few resources, both official and third party, the most helpful of which turned out to be a set of in-browser slides by the Ghidra team themselves - which unfortunately I can no longer seem to find! Nearly *all* resources however suggested using Eclipse, which Ghidra also ships a plugin ("GhidraDev") for, to supposedly aid Extension development. Of course, I ditched Eclipse altogether and opened the code in Intellij (IDEA). Then I found [this blog post](https://reversing.technology/2019/11/18/ghidra-dev-pt1.html) describing how to add a one-click-build button in Intellij, basically boiling down to a new "Facet" which would spawn, but eventually I found the proposed flow not so optimal and therefore defaulted to using the raw `gradle`  command from the CLI, just like Ayrx. A proper build is only needed for the first run anyway, as for minor subsequent edits Ghidra's Live Edit (depicted below) was enough.  So all-in-all I'd suggest starting the script development in Intellij purely for the syntactic sugar, then building from the command line.   

The final code for the script can be found on [Github](https://github.com/laripping/NativeEnrich). Following the installation instructions included, we import this to Ghidra. This is how it looks like in the Script Manager:

![](/assets/img/script_manager.png)

Executing it on the target Code Listing and providing  it the JSON files when prompted leads to the following output on the Scripting window:

![](/assets/img/script_out_final.png)

We let it run briefly and boom! Hundreds of new, properly typed functions in both the listing, the disassembly, the decompiler. Xrefs and everything. Just looks at this bad boy:

![](/assets/img/script_result_codelisting.png)





## Putting it all together

To conclude, the method we demonstrated in the previous sections allows one to enrich the disassembly of a native library, with functions dynamically registered during runtime. Functions which the disassembler would otherwise have no way of identifying. In total, this process boils down to the following steps: 

1. Download [FindNativeJNIMethods.jar](https://github.com/Ayrx/FindNativeJNIMethods/releases/download/0.4/FindNativeJNIMethods.jar) and give it your APK

   ```bash
   $ java -jar FindNativeJNIMethods.jar base.apk nativeMethods-jar.json
   ```

2. Setup a device/emulator with Frida and run the Frida counterpart against your app, then print the JSON object in the interpreter

   ```bash
   $ frida --codeshare LAripping/trace-registernatives -f com.app
    ...
   [Device::com.app]-> nativeMethods
   ```

   Copy the output and save it as `nativeMethods-frida.json` 

3. Download the [NativeEnrich](https://github.com/laripping/NativeEnrich) Ghidra script, and compile it following the instructions

4. Import the ZIP file to Ghidra and run it - providing the JSON files when prompted

5. Dig into your newly identified functions! :hammer:



## Closing notes

* **One Python to rule them all?**

  Yes, Ghidra also offers a Python interpreter, and the (Java) Dev APIs are all translated to Python counterparts so in theory, yes, I could port everything to a one-stop-shop PyGhidra script, using the Frida python library. In practice though I found this both a cumbersome solution -as Frida would have to attach to the device from within Ghidra- and a potentially unnecessary binding of the method's steps, which might not be as trivial for your apps as they were for mine. Essentially, it's highly possible that running Frida in first place might present challenges such as Anti-debugging controls, Root detection routines etc. So I opted to keep things loose and maybe less fool-proof.       

* **Skip Frida altogether and use Ghidra Debugger?** 

  Another idea would be to attach to the process with Ghidra Debugger (*if* there is such thing, I'm honestly not sure and just presume based on IDA experience) in order to identify invocations of the `RegisterNatives` API. Then I quickly realised that `RegisterNatives` isn't even in the library of our choosing, since -as we found- it lies in the system `libart.so` instead. Primarily though, should there be another way to replace Frida with the debugger to once again bundle it all in one single script, this would be subject to challenges similar to the previous bullet, as attaching a debugger to a production application is rarely trivial if not impossible.



Aand that's about it! Thanks for reading and happy Ghidra-staring!  