---
layout: post
title:  "Working of LD_PRELOAD for Android Applications and its anti-RE Technique"
date:   2017-04-01 16:17:22 +0800
---

*Have you ever checked the parent process of an Android application launched using “wrap” system property set with LD_PRELOAD value? It's not Zygote!!!*

**Expected Reading Time**: 10 mins

There are multiple tools and methodology available for reverse engineering an Android application. In this post, I am not exploring various RE techniques available, instead look into how one such technique works in case of Android applications. **LD_PRELOAD** is a functionality of dynamic linker present across all Linux distributions, giving precedence to library passed to be searched first for symbols. It is not a RE technique per se, but I have used it often for such purposes and hence I classify it as a RE technique which one should know.

Many Linux users will be aware of how LD_PRELOAD works in case of native binaries. For the starters, there are many good blogs available and I will leave it as an assignment. As a side note, knowledge of how dynamic linkers work is always handy for reverse engineering assignments.

In case of a simple native binary compiled to run on Android, LD_PRELOAD works as for other Linux systems. Things get interesting once you intend to use LD_PRELOAD for an Android application. If you simply set the LD_PRELOAD env variable and then try to launch the application, it will not work. Rather you need to perform some Android trickery to make it work for an application. For example, we want to load libpreload.so using LD_PRELOAD for application with package name com.foo.bar. We need to perform following steps:

- Push the library to the device.  *adb push libpreload.so /data/local/tmp/libpreload.so*
- Set system property:  *setprop wrap.com.foo.bar LD_PRELOAD=/data/local/tmp/libpreload.so*

Note, in last step the property name is  "wrap" prefixed to the application's package name.

Now if you launch the application, the library *libpreload.so* will be searched first by the dynamic linker for symbols. (In recent Android versions, one need to disable SELinux to make this work, as *libpreload.so* does not have SELinux context assigned, because of which the library is not loaded and whole process exits on such an error.) The above work around is neither a hidden feature nor a newly added feature. Many of us might have used it several times in the past.

After knowing how to make it work practically, lets try to understand what is actually happening internally, and why we need to perform these extra steps. Some might have already guessed, it is because of Zygote process creation model used in Android. As covered previously on this blog as well, an Android application is launched by forking a Zygote process. Argument being,  in order to save time while launching an application, Android coldstarts a process during OS startup, from which applications can be forked when required. This process is initialized with all necessary libraries required to run an Android application, and remains in sleep state. When a request to launch an application is received, this Zygote process is simply forked and used there on.

Since, Zygote process is initialized well before launching the application, setting LD_PRELOAD env variable before launching an application will cause no change in symbol search order for linker. Thus, to enable LD_PRELOAD functionality, Android needs a work around.

Lets revisit how an application launch request is passed to Zygote. When user makes a request to launch an application,  a request is sent to /dev/socket/zygote. On receiving request, Zygote forks itself and launches the application in the child process. As per this mechanism all the application’s, as expected, will have **Zygote as the parent process**. But in case of LD_PRELOAD scenario, a slight different code path is taken.

Code listening at */dev/socket/zygote* runs in a loop, referred as 'select loop' in code. On receiving request, *runOnce()* in [ZygoteConnection](http://androidxref.com/5.1.1_r6/xref/frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java#142)  is called. In this function, many checks and operations are performed, one such check is to check if “wrap” system property set for the given application’s package name.

```java
boolean runOnce() throws ZygoteInit.MethodAndArgsCaller {
...
applyUidSecurityPolicy(parsedArgs, peer, peerSecurityContext);
applyRlimitSecurityPolicy(parsedArgs, peer, peerSecurityContext);
applyInvokeWithSecurityPolicy(parsedArgs, peer, peerSecurityContext);
applyseInfoSecurityPolicy(parsedArgs, peer, peerSecurityContext);
checkTime(startTime, "zygoteConnection.runOnce: apply security policies");
applyDebuggerSystemProperty(parsedArgs);
applyInvokeWithSystemProperty(parsedArgs);
checkTime(startTime, "zygoteConnection.runOnce: apply security policies");
...
}
```

If it is set, it extracts the value of the property. It is important that, a system property name cannot be more than 31 chars, and if package name is more than 31 chars, it need to be truncated to 31 chars.

After extracting the value, execApplication() in [WrapperInit](http://androidxref.com/5.1.1_r6/xref/frameworks/base/core/java/com/android/internal/os/WrapperInit.java#97) is called. In this function, a command string is constructed with required parameters, to launch application via **/system/bin/app_process**  (Zygote) directly. The shell arguments, LD_PRELOAD in this case, is appended in this command string. This command is executed using **/system/bin/sh**. Thus, making **/system/bin/sh as the parent process to the finally launched application**.

```java
public static void execApplication(String invokeWith, String niceName,
int targetSdkVersion, FileDescriptor pipeFd, String[] args) {

StringBuilder command = new StringBuilder(invokeWith);
command.append(" /system/bin/app_process /system/bin --application");
if (niceName != null) {
command.append(" '--nice-name=").append(niceName).append("'");
}
command.append(" com.android.internal.os.WrapperInit ");
command.append(pipeFd != null ? pipeFd.getInt$() : 0);
command.append(' ');
command.append(targetSdkVersion);
Zygote.appendQuotedShellArgs(command, args);
Zygote.execShell(command.toString());
}
```

This very fact, that parent process for such an application is not Zygote, can be used as an anti-RE technique. If application's parent is not Zygote, then application can change its behavior or exit, preventing application analysis.

Keep hacking :)
