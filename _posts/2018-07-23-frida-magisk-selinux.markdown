---
layout: post
title:  "Frida, Magisk and SELinux"
date:   2018-07-23 16:17:22 +0800
---
While using Frida 12.x with Magisk v16.3+, I came across the problem that Frida is not able to spawn applications on Android. In logcat one can see the SELinux error:

```
avc: denied { sigchld } for scontext=u:r:zygote:s0 tcontext=u:r:magisk:s0 tclass=process permissive=0
```


I am not sure from the problem has started, is it Magisk, which does perform some tinkering with SELinux or is it new version of Frida. I have not open a bug with either projects yet, as I am not sure.

But for those stuck with this problem, the solution is to use magiskpolicy, which is installed along with magisk. Just go to the terminal and add this policy:

```
magiskpolicy --live "allow zygote magisk process *"
magiskpolicy --live "allow system_server magisk process *"
magiskpolicy --live "allow radio magisk process *"
```

In case someone knows the root cause I would be more than happy to know.

UPDATE:

It seems others also have come across problems related to SELinux and FRIDA, like https://github.com/frida/frida-core/issues/123. The recommended solution is to set SELinux to Permissive mode. In case you don't want to do that, as some applications do check for SELinux status and do not work if set to Permissive mode, then use the commands above, as a workaround.


