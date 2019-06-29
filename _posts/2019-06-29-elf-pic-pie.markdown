---
layout: post
title:  "Differentiate ELF executable from shared library"
date:   2019-06-29 16:17:22 +0800
---

All modern operating systems support Address Space Layout Randomization, ASLR, as part of defense in depth approach, to make it harder for an attacker to reliably jump to a memory location while exploiting a vulnerability, like bufferoverflows. To effectively use ASLR, the code should be compiled as position independent, i.e code can be loaded at any address in the memory. Shared libraries are always compiled as position independent, while in the past (before ASLR) an ELF executable was always loaded at a fixed address.
![ELF library](/assets/images/pic-pie-readelf-lib.png)
![ELF executable](/assets/images/pic-pie-readelf-ssh.png)

For an executable to utilize ASLR, it needs to be loaded at any address in the memory, called as PIE (position independent executable). The solution is quite simple and logical by using existing implementations, by compiling executable as a shared library. This can be verified by checking the ELF File type using readelf. Both shared library and position independent executable show as _DYN (Shared object file)_.   

Since both files have same ELF filetype, it raises the question how to determine if a given ELF binary is a shared library or an executable. The answer unsurprisingly lies in the fact how the OS differentiates between the two, given it cannot use ELF filetype anymore to differentiate between the two. A quick read through on how an ELF file is [loaded by the kernel and executed](https://lwn.net/Articles/631631/) holds the answer. If you are interested in learning the gory details, the article is highly recommended, and while for the others, the crux lies
in "The code now loops over the program header entries, checking for an interpreter (PT_INTERP) ...". *PT_INTERP* is an ELF segment which contains the path name of the interpreter to be used to execute the file. And this *PT_INTERP* segment is the answer to our problem. An ELF executable file has this segment while a shared library will not. This can be verified using readelf as shown in the screenshots below.  

![ELF segments in library](/assets/images/pic-pie-readelf-segment-lib.png)
![ELF segments in executable](/assets/images/pic-pie-readelf-segment-ssh.png)

An ELF executable and shared library are so closely similar that with some hacking you can make an executable work as a shared library. This is discussed by [Romain Thomas](https://lief.quarkslab.com/doc/latest/tutorials/08_elf_bin2lib.html) on how to achieve this using [LIEF](https://github.com/lief-project/LIEF). Also, Ian Lance in [Piece of PIE](https://www.airs.com/blog/archives/549) discusses about how similar ELF position independent executable and shared library are. 
