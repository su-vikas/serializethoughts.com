---
layout: post
title:  "Bypassing anti-debugger check in iOS Applications"
date:   2018-01-23 16:17:22 +0800
---

**Expected Reading Time**: 5 Mins

While pentesting mobile applications, very often you will come across applications implementing myriads of anti-reversing techniques. Specially while performing dynamic analysis, it is imperative to disable these checks. To make the bypassing these checks more daunting, some applications heavily obfuscate the binaries. In this post, we will look into bypassing one of the anti-debugging technique for iOS applications and using IDAPython script to automate patching of the binary. The idea presented in this post is really simple, and many readers might have been already using it, for others here is another idea to add to your armory.

# Ptrace

iOS under the hood runs a XNU kernel. The XNU kernel do implement ptrace() system call, but it is not potent enough when compared to Unix and Linux implementations. On the other hand, XNU kernel exposes another interface via Mach IPC to perform debugging. In this post we won't be comparing between the two mechanisms, in fact we will only talk about one feature of ptrace syscall, PT_DENY_ATTACH. It is a fairly well known among the iOS application hackers, and among the most frequently encountered anti-debugging technique in iOS applications.

Before getting into the details of bypassing the check, lets first try to understand briefly what exactly does PT_DENY_ATTACH do. Although, there is ample literature available on the Internet discussing PT_DENY_ATTACH in various depths, but [The Mac Hacker's Handbook](https://www.amazon.com/Mac-Hackers-Handbook-Charlie-Miller/dp/0470395362) defines it most succinctly.

> PT_DENY_ATTACH
> This request is the other operation used by the traced process; it allows a process that is not currently being traced to deny future traces by its parent. All other arguments are ignored. If the process is currently being traced, it will exit with the exit status of ENOTSUP; otherwise, it sets a flag that denies future traces. An attempt by the parent to trace a process which has set this flag will result in the segmentation violation in the parent.

To rephrase it, using ptrace with PT_DENY_ATTACH, it ensures that no other debugger can attach to the calling process; even if an attempt is made to attach a debugger, the process exits.

It is important to know that ptrace() is not part of public API on iOS. As per the [AppStore publishing policy](http://(https//developer.apple.com/documentation/), use of non-public API is prohibited and use of them may lead to rejection of the app from the AppStore ). Because of this, developers don't call ptrace()   directly in the code, instead its called via obtaining ptrace() function pointer using dlsym. Programmatically it looks like:

```objective-c
#import %3Csys/types.h>
#import typedef int (*ptrace_ptr_t)(int _request, pid_t _pid, caddr_t _addr, int _data);
void anti_debug() {
ptrace_ptr_t ptrace_ptr = (ptrace_ptr_t)dlsym(RTLD_SELF, "ptrace"); ptrace_ptr(31, 0, 0, 0); // PTRACE_DENY_ATTACH = 31
}
```

The dis-assembly of the binary implementing this approach looks like following:

![ptrace original](/assets/images/ptrace-original.png)

To break down whats happening in the binary, at 0x19086 dlsym() is called with "ptrace" as the 2nd argument (register R1, offset 0x19084) to the function. The return value, in register R0 is moved to register R6 at offset 0x1908A. Eventually at offset 0x19098, the pointer value in register R6 is called using BLX R6 instruction. In this case, to disable ptrace() call, all we need to do is to replace the instruction BLX R6 (0xB0 0x47 in Little Endian) with NOP (0x00 0xBF in Little Endian) instruction. [Armconverter.com](http://armconverter.com/) is a handy tool for conversion between bytecode and instruction mnemonics. The code after patching looks like following:

![call_ptrace](/assets/images/call-ptrace.png)

This can be done manually easily if there is only one or a few calls to ptrace, but this turns tedious once such calls are made multiple times across the binary.

# IDAPython Script
IDAPython script can be leveraged to perform this task automatically. I wrote one such script to automate the task for the binary dis-assembly shown above. 

{% gist 06b6837a0f8b66a7a0a33f4ed719510a %}

The script is quite self explanatory, but if something is not clear, feel free to drop a comment below for further clarification.

To conclude, there are many other anti-debugging approaches available for iOS, the one discussed here is the most commonly found in the iOS applications. Using such techniques to slow down the attackers is fine, but solely depending on such techniques for the security of an application is not at all advisable. Such techniques can be part of the defense in depth approach, but should not be the only defense.

Keep hacking :)
