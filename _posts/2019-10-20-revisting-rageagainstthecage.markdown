---
layout: post
title:  "RageAgainstTheCage - Revisting Android adb setuid Exhaustion Attack"
date:   2019-10-28 16:17:22 +0800
---

**Time To Read:** 5 min

**TL;DR:** *adb setuid exhaustion attack* (aka RageAgainstTheCage) was present in Android 1.6 to 2.2. During *adb* initialisation, while dropping its privileges there is no check for *setuid* syscall's return value. This can be exploited by causing a race condition by creating RLIMIT_NPROC processes and killing *adb*, on *adb* restart *setuid* syscall might fail and can lead to *adb* continue running with root privileges. 

In 2010 Sebastian Krahmer discovered a vulnerability in the implementation of *adb* for Android 1.6 to 2.2 versions. The vulnerability is - *adb* fails to check *setuid* return code and this can be caused to fail by a shell user already having RLIMIT_NPROC processes. The vulnerability in itself is very simple, unlike many other vulnerabilities which require various memory gymnastics to exploit. Apart from the simplicity, the vulnerability gives us a good insight into some of the Linux kernel working and also a good lesson for the developers.  

# RLIMIT_NPROC

To understand the working of this exploit, we need to understand how resource allocation works in Linux. Linux uses *getrlimit()* and *setrlimit()* system calls to get and set resource limits for a process. Some of the process resources are: virtual memory size, CPU time a process can consume etc, a list is available at [*getrlimit* man page](http://man7.org/linux/man-pages/man2/setrlimit.2.html). In this case we are interested in RLIMIT_NPROC resource. This resources was introduced to protect against a [fork bomb](https://en.wikipedia.org/wiki/Fork_bomb "fork bomb"). As per the man page:

> This is a limit on the number of extant process (or, more precisely on Linux, threads) for the real user ID of the calling process.  So long as the current number of processes belonging to this process's real user ID is greater than or equal to this limit, fork(2) fails with the error EAGAIN. 

# Vulnerability

When Android Debug Bridge (*adb*) is started, there are multiple tasks need to be performed before *adb* is available to the user. To accomplish these tasks, *adb* is launched with root privileges and once all the initialisation is complete, it drops its privileges. Linux system call *setuid* is used to lower the privileges. The main cause of the vulnerability is, in Android 1.6 to 2.2, *adb* does not check the return value of this *setuid* syscall while lowering its privileges. 

```c
// setuid call in adb 

setgid(AID_SHELL);
setuid(AID_SHELL);
```

When *setuid* is called, the kernel checks if the number of processes for that non-root user set by administrator is not crossed (i.e RLIMIT_NPROC). If this non-root user already has RLIMIT_NPROC processes running, then *setuid* call will fail. 

If we can somehow reach this RLIMIT_NPROC processes and then make *adb* restart, the kernel will prevent the *adb* to lower its privileges. Since, *adb* does not check the *setuid* return value, it will continue to execute with root privileges. To accomplish this, we can launch multiple dummy processes and reach RLIMIT_NPROC process limit for the user. Then kill *adb*, which will cause *adbd* to restart *adb*. While relaunching *adb*, it causes a race condition between *adb* and the processes we launched to spawn the last RLIMIT_NPROC process. If our dummy process is launched, *setuid* for *adb* will fail, and in this case it will continue running with root privileges.   

Multiple dummy processes can be launched using the following code:

```c
if (fork() == 0) {
    fork();
    for (;;)
        sleep(0x743C);
}
```

# Fix

One obvious fix for this vulnerability is to check the return value of *setuid*. But it seems non-checking of *setuid* return value is a common mistake with grave consequences. In the past, *sendmail capabilities bug* was caused by the same programming malpractice. The common occurrence of this programming pattern has lead to discussion within Linux Kernel community to fix it in the Kernel itself, discussed in this excellent [LWN article](https://lwn.net/Articles/451985/). 

# References

1. https://thesnkchrmr.wordpress.com/2011/03/24/rageagainstthecage/
2. https://lwn.net/Articles/451985/
