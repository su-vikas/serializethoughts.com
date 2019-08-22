---
layout: post
title:  "Linux Thread is a Standard Process"
date:   2016-78-16 16:17:22 +0800
---

As per [Wikipedia](https://en.wikipedia.org/wiki/Thread_(computing)), a computing thread is defined as "the smallest sequence of programmed instructions that can be managed independently by a scheduler". And further goes on to say, the implementation of threads and processes differs between operating systems, but in most cases a thread is a component of a process. A process can have multiple threads within a shared memory address space of the process.  In this article, I won't be going into the nitty-gritty of threads. There are many good resources available on the Internet discussing various details of threads. Rather I would like to focus on an important aspect of threads specifically in Linux kernel.

While discussing Linux internals with a senior developer, he mentioned that in case of Linux kernel, there is no difference between a process and a thread. Intrigued by the fact, I started looking into kernel's source, aided by the excellent book on Linux kernel, [Linux Kernel Development](https://www.amazon.com/Linux-Kernel-Development-Robert-Love/dp/0672329468), by Robert Love.

Coming on to the topic, in Linux, there is no separate data structure defining a thread. A thread and a process have the same data structure. Also, a thread is created exactly in the same way as a normal process is created, i.e by calling fork or vfork() (which eventually calls clone() system call). Although, in case of a thread, while making clone() system call some extra flags are passed. These extra flags specify the resources which are to be shared. A typical call looks like:
```
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND, 0);
```

So the address space, filesystem resources, file descriptors, and signal handlers are shared.

For the readers who want to delve more into the internal working, *struct task_struct* contains information about a process, defined in *<linux/sched.h>*. The field *unsigned int* flags stores all the flag related information. These flags, help specify the behaviour of the new process and detail what resources the parent and child will share. List of other clone flags can also be found in linux/sched.h.

For completeness, in case of Microsoft Windows, kernel have explicit support for kernels and also referred as lighweight process. For Windows, a thread is an abstraction which provides  a lighter, quicker execution than the heavy process. The pros and cons of the two approaches requires more deep understanding of the two, and will be a much bigger post than this one.
