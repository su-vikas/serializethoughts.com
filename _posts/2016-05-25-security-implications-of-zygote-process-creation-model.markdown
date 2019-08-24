---
layout: post
title:  "Security Implications of Zygote Process Creation Model in Android"
date:   2016-05-25 16:17:22 +0800
---

In the [previous post](https://serializethoughts.com/2016/04/15/android-zygote/) I discussed about the Zygote process creation model in Android OS and importance of having different process creation model than a linux process in a mobile device. Before getting into the technical specifics, it is advisable to freshen the concept pertaining to Linux process creation and ASLR.

In  Zygote process creation model, a process template is created on the system startup, having the Dalvik VM (DVM) instance initialized and also other essential libraries loaded. When an application launch request is received, this template process is forked and application is loaded into it, saving significant time by avoiding loading of libraries and in instantiating DVM on every launch. Also, Linux COW (copy-on-write) helps in reducing global memory usage. But this trade-off have a major security implication.

Zygote process creation model causes two types of  memory  layout  sharing  on  Android,  which  undermine  the effectiveness of ASLR. Firstly, the code of an application is always loaded at the exact same memory location across different runs even when  ASLR  is  present;  and  secondly,  all  running apps  inherit  the  commonly  used  libraries  from  the  Zygote process (including the libc library) and thus share the same virtual memory mappings of these libraries. If an attacker is able to get the memory mapping information for one process, he can easily predict for the target process (as both share same mappings for Zygote loaded libraries). Thus developing ROP attacks becomes very easy, in spite of having ASLR. For more details, read this excellent article by Copperhead team.

After understanding the pros and cons of the approach, it must raise the curiosity that how can it be fixed without breaking the existing applications. To start with, one approach is to switch to Linux style process creation model, i.e, fork and exec, rather than creating a template and keep forking it subsequently. This approach fixes the security issue, but raises the very issue for which Zygote model was created. We will revisit this approach again at the end and re-evaluate its applicability in light of new performance enhancements measures introduced in Android.

Another approach to fix the security issue with Zygote process creation model was proposed by [Lee et. al. in this paper](http://wenke.gtisc.gatech.edu/papers/morula.pdf). They name their approach as Morula process creation model. Why it is called Morula is left as an exercise for the readers.

# Morula Process Creation Model:

In this approach, a template process performs the common and time-consuming initialisation tasks beforehand.  The whole process is divided into 2 steps:

1. **Preparation Phase**: initiated by the Activity Manager. A preparation request is made to the Zygote process, when the system is idle or lightly occupied. The Zygote process forks a child, which immediately call a exec() to establish a new memory image with a fresh randomized layout. Then, the new process constructs DVM instance and loads all shared libraries and common Android classes, which would tremendously prolong the app launch time if not done in advance. Now the Morula process is fully created, waiting for the request to start an app. Multiple Morula process can be created in order to accommodate a flurry of requests to start several apps. If these newly created process is not used immediately, the  process enters the sleep mode and awakened only when needed.

2. **Transition Phase**: When the Activity Manager requests an app launch, the request is routed through Zygote process first, where a decision is made regarding if the app should be started in a Morula process or in a fork of the Zygote process. Having this option allows the Morula model to be backward compatible with the Zygote model, in order to carry out an optimization strategy. Depending on the configuration, using either of the process the application is launched.

From Android 4.4.4 onward, Google was previewing a new runtime environment instead of Dalvik, called ART. From Android 5.0 it was fully deployed and considered to be faster than previous Dalvik VM. Factoring in the advances made by ART, Copperhead experimented with fork and exec approach in their Copperhead OS. As per their [findings](https://copperhead.co/blog/2015/05/11/aslr-android-zygote), "The Morula proof of concept code has some issues like file descriptor leaks and needs to be ported to Lollipop. It’s much less important now that ART has drastically improved start-up time without the zygote." To conclude, with ART, fork and exec approach is not as slow as compared with Dalvik runtime. For security paranoid users, a small performance glitch should not be a big barrier.

It is highly recommended to read both, the Morula paper and blog by Copperhead to understand the nitty-gritty of the topic. And for the hackers out their, patch for implementing Morula is available [here](https://github.com/lifeasageek/morula).

