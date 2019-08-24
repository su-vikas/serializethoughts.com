---
layout: post
title:  "Android Zygote"
date:   2016-04-15 16:17:22 +0800
---

In this post I will discuss about a very interesting piece of Android Operating System. If you have worked with Android, you might have run the *ps* command and might have observed that all applications have same parent PID (PPID). Android takes an unconventional approach to spawn processes, which ensure application startup is snappy. The process from which all the Android applications are derived is called Zygote. So in the screenshot below, all the applications have PPID of 1914, which is the PID of Zygote. In the rest of the post, I will talk about what is the need of Zygote, how does it come into existence and some discussion about Zygote in general.

![android_zygote](/assets/images/android_zygote.png)

# Need of Zygote?

In a typical Linux process, when a process is started - by forking parent process, it goes through various setup steps, including loading of libraries and resources. The details are out of scope for this post. This process setup consumes time and on our beefy desktops, it is hardly noticeable. But in case of Android, not all devices are high spec and this process setup is noticeable to end user. As a workaround to normalize the process startup times on various devices, Android coldstarts a process during OS startup, from which applications can be forked when required. This process is called **Zygote**.

# Zygote Startup

After the Android device is turned on, and following all booting up steps, then init system starts, and run */init.rc* file to setup various environment variables, mount points, start native daemons etc. There are many resources available on the internet discussing about Android bootup process and init system as well, and thus bypassed in this post. While executing init.rc, that Zygote is started. There is no binary directly corresponding to Zygote, instead it is started by a binarycalled app_process. Corresponding line in init.rc can be found here.

```
    service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
```

*app_process* firstly starts Android Runtime, which in turn starts Dalvik VM of the system, and finally Dalvik VM invokes Zygote's [main()](https://github.com/android/platform_frameworks_base/blob/master/core/java/com/android/internal/os/ZygoteInit.java#575).

Zygote initilization can be simplified into following steps:
1. Register Zygote socket (listens for connections on /dev/socket/zygote) for requests to start new apps
2. preload all java classes
3. preload resources
4. start system server (not covered in this post)
5. open socket
6. listen to connections

# Zygote Socket

As mentioned above, zygote opens up a socket, */dev/socket/zygote*, and listens on it for requests to start new applicationss. On receiving a request, it simply forks itself and launches the new requested application. So now the new application have Dalvik VM already loaded, including other necessary libraries and resources, and can start with execution straight away.

One feature of Linux is important to understand over here, Copy-on-Write (COW) policy for forks. Forking in Linux involves creating a new process, which is an exact copy of the parent process. With COW, Linux doesn't actually copy anything. On the contrary, it just maps the pages of the new process over to those of the parent process and makes copies only when the new process writes to a page. Thus, saving significant amount of memory and setup time as well. To add, in case of Android, these pages are never written, as the libraries are mostly immutable and hardly changed over the process lifetime.

If you want to learn more details, with corresponding code, [this is an excellent resource.](http://allenlsy.com/android-kernel-3/)

In upcoming post I will discuss about the security implications of having Zygote model and various existing alternative workarounds.
