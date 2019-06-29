---
layout: post
title:  "Differentiate an ELF executable from a shared library"
date:   2019-06-29 16:17:22 +0800
---
**Time To Read:** 5 min

**TL;DR:** A position independent ELF executable is compiled as a shared library to achieve position independent code. The OS is able to differentiate between the two by checking for PT_INTERP segment, which is present in executable and not in a shared library. 


All modern operating systems support Address Space Layout Randomization, ASLR, as part of defense in depth approach, to make it harder for an attacker to reliably jump to a memory location while exploiting a vulnerability, like bufferoverflows. To effectively use ASLR, the code should be compiled as position independent, i.e code can be loaded at any address in the memory. Shared libraries are always compiled as position independent, while in the past (before ASLR) an ELF executable was always loaded at a fixed address.

![ELF library](/assets/images/pic-pie-readelf-lib.png)

![ELF executable](/assets/images/pic-pie-readelf-ssh.png)

For an executable to utilize ASLR, OS should be able to load executable at any address in the memory. Such executable is often referred as PIE (position independent executable). To make an executable PIE, the solution lies in reusing the existing implementation of how a shared library is compiled. In a nutshell, it turns out to be quite simple to create a PIE: a PIE is simply an executable shared library. This can be verified by checking the ELF filetype using *readelf* command. Both shared library and PIE are: _DYN (Shared object file)_.   

Since both files have same ELF filetype, it raises the question how to determine if a given ELF binary is a shared library or an executable. The answer unsurprisingly lies in the fact how the OS differentiates between the two. As it is evident from above, ELF filetype cannot be used anymore to differentiate between the two, the OS need to look for some other hints. 

A quick read through on how an ELF file is [loaded by the kernel and executed](https://lwn.net/Articles/631631/) gives us the hint. If you are interested in learning the gory details, the article is highly recommended, while for the others, the hint is in "The code now loops over the program header entries, checking for an interpreter (PT_INTERP) ...". As it turns out **PT_INTERP** is an ELF segment which contains the path name of the interpreter used to execute a file. And this **PT_INTERP** segment is the answer to our problem. An ELF executable file has this segment while a shared library will not. This can be verified using readelf as shown in the screenshots below.  

![ELF segments in library](/assets/images/pic-pie-readelf-segment-lib.png)

![ELF segments in executable](/assets/images/pic-pie-readelf-segment-ssh.png)

To sumamrize, an ELF executable is made to be position independent by compiling it like a shared library, and to make this shared library executable you just need to give it a *PT_INTERP* segment and appropriate startup code.

Some other information I discovered and might be interesting for further experimentation is, an ELF executable and shared library are so closely similar that with some hacking you can make an executable work as a shared library. This is discussed by [Romain Thomas](https://lief.quarkslab.com/doc/latest/tutorials/08_elf_bin2lib.html) on how to achieve this using [LIEF](https://github.com/lief-project/LIEF). Also, Ian Lance in [Piece of PIE](https://www.airs.com/blog/archives/549) discusses about how similar ELF position independent executable and shared library are. 
