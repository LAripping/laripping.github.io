---
layout: post
title: 'Xiaomi Redmi 5 Plus: Second Space Password Bypass'
excerpt_separator: "<!--more-->"
categories:
  - "cve-advisory"
tags:
  - android
  - system
  - xiaomi
  - bounty
  - "second space"
last_modified_at: 2021-12-22T15:35:00
---

{% include note.html content="This advisory was originally published on [F-Secure LABS](https://labs.f-secure.com/advisories/xiaomi-second-space/)" %}

<!-- {% include toc.html %} -->
<!-- Again, TOC messes things up -->

Xiaomi Second Space replaces Android User Profiles on MIUI devices. It allows for a Primary (admin) and a Second user to switch profiles via an icon on the homescreen or from the lock screen. Both user spaces can be protected by a PIN or password.

A method was discovered in the Xiaomi MIUI System, that allows a user to switch between spaces without providing a password or PIN. This requires Second Space and Password / PIN screen lock to be enabled along with USB debugging.

The vulnerability is triggered by an ADB command that sends an intent to start the `SwitchUserService` service with a flag that bypasses the password prompt. As a result, the user can immediately switch space without knowing the password.

The issue was discovered in a Redmi 5 Plus device, but it has been confirmed to affect several of the latest MIUI versions.

<!--more-->

## Proof of Concept

From a brand new Redmi 5 plus device:

1. Set up "Second Space" feature, and set a Lock Screen Password or PIN if prompted for one, from **Settings** > **Second Space** 
2. Activate Developer Options by navigating to **Settings** > **About Phone** and tapping 7 times on **MIUI Version**
3. Enable USB Debugging from **Settings** > **Additional Settings** > **Developer Options** > **USB Debugging**
4. Connect USB cable and authorise Computer once prompted
5. Type the following commands to get Second Space user's ID and then switch to it, bypassing the Password / PIN prompt

```bash
$ SECOND_USER=`adb shell pm list users | grep -o "{[0-9]*" | tr -d '{' | tail -n 1`
$ adb shell am start-service --ez params_check_password False --ei params_target_user_id $SECOND_USER -a com.miui.xspace.TO_CHANGE_USER
```

to switch from Second Space to Primary without providing the password just change to

```bash
$ adb shell am start-service -e params_check_password False -e params_target_user_id 0 -a com.miui.xspace.TO_CHANGE_USER 
```

The process is demonstrated in the video attached below (full screen viewing is recommended).

<!-- <div class="video-container">
  <iframe class="embed-responsive-item" src="/assets/video/xiaomi-demo.mp4" allowfullscreen frameborder="0">
  </iframe>
</div> -->

<video controls="controls" style="width:100%">
  <source src="/assets/video/xiaomi-demo.mp4" type="video/mp4">
</video>


The following actions are recorded:

1. (00:02) Switch to the second space, PIN entry is requested and provided when the screen is blacked, visible through the "Settings" activity briefly shown
2. (00:13) Switch back to the primary space, PIN entry is requested and provided. Again, the "Settings" activity is briefly shown
3. (00:29) Switch to the second space without PIN, no "Settings" popup
4. (00:43) Switch back to the primary space altering the `params_target_user_id`, without PIN no "Settings" popup
5. (01:23) Switch again to the second space to show the PIN entry requirement in the relevant Settings

## Technical Details

The system application / package responsible for most Second Space functionality is `com.miui.securitycore`. Static analysis of the "Switching between spaces" option in the settings seen below, points us to the `SecondSpaceSettingsFragment` class which implements this switch with a call to the `switchToOwnerSpace()` method. This in turn starts the `com.miui.securityspace.service.SwitchUserService` service with an intent.

<table>
  <tr>
    <td>
      <img src="/assets/img/2nd-space-settings-hl.png" style="height:auto; width:auto; max-height:500px" />
    </td>
    <td>
      <img src="/assets/img/xiaomi-code.png"  style="height:auto; width:auto; max-height:500px"  /> 
    </td>
  </tr>
</table>


Inside this service's implementation, the `startCommand()` override calls `checkPasswordBeforeSwitch()` that processes the intent and then calls `SpaceManager.switchUser()` from where the switch process is kicked-off.

The vulnerability lies in the way the password requirement is enforced inside `checkPasswordBeforeSwitch()`, as the existence of a boolean extra is sufficient to skip the prompt. This means that one can bypass password entry when sending the service start intent with the `params_check_password` extra set to False. This can be achieved using the ADB shell, without need for root access, by issuing the following command:

```bash
$ am start-service --ez params_check_password False --ei params_target_user_id 10 -a com.miui.xspace.TO_CHANGE_USER
```

The above command also works in reverse, i.e. switching from second to primary space by specifying the `params_target_user_id` integer extra to 0.

Closer examination of the application's `AndroidManifest.xml` reveals the Service is not declared `private` nor `exported=false`. However, if not explicitly set, the Service is exported by default due to the existence of intent-filters, as seen in the following extract from the manifest:

```xml
<service android:name="com.miui.securityspace.service.SwitchUserService"
         android:permission="android.permission.INTERACT_ACROSS_USERS"
         android:process="com.miui.securitycore.remote">
    <intent-filter>
        <action android:name="com.miui.xspace.TO_CHANGE_USER"/>
    </intent-filter>
</service>
```

Therefore, the security of the Service relies on the permission required to start it, as most applications holding the `INTERACT_ACROSS_USERS` permission are system ones. This permission however is also held by the shell user, which is the access level a normal, non-root ADB shell provides.

## Credit

The issue was discovered by Leonidas Tsaousis ([@laripping](https://twitter.com/laripping)) of F-Secure Consulting.


## Timeline


| Date | Summary |
| ---- | ------- |
| 25/02/2020 | Issue reported to Xiaomi through HackerOne  |
| 26/02/2020 | Bug triaged and bounty was rewarded. |
| 04/05/2020 | Contacted Xiaomi via HackerOne, requesting status update |
| 06/05/2020 | Xiaomi replied stating the issue has been fixed and would be tested in development version released by the end of the week  |
| 09/05/2020 | Xiaomi states that the issue is fixed in development version 5.09 |
| 28/05/2020 | F-Secure release advisory |