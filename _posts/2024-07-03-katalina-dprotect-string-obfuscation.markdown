---
layout: post
title:  "Recovering dProtect's Obfuscated Strings Using Katalina in an Android App"
date:   2024-07-03 18:32:22 +0800
tags:
    - Obfuscation
    - Android
    - Dex
    - Java
    - Emulation
---

**TL;DR:** *In an Android app, strings obfuscated using dProtect can be recovered by Dalvik code emulation using Katalina.*

**Time To Read:** 5 min

## Katalina
[Katalina](https://github.com/huuck/Katalina) is a Dalvik bytecode emulation tool. Katalina implements a sandbox in Python and executes one dalvik instructions at a time for emulation. As highlighted by the author, such approach is useful in dealing with obfuscated code, especially in recovering obfuscated strings.

String obfuscation is a highly effective technique used by both, malwares and genuine applications to slow down reverse engineering attempts. For instance, lack of visible strings can deter many automated static analysis tools and slows down manual attempts, forcing the attacker to perform dynamic analysis. The app authors can further implement various anti-dynamic analysis techniques to prolong the time to reverse engineer the app. 

To put Katalina to test, I created an obfuscated application using dProtect. [dProtect](https://github.com/open-obfuscator/dprotect) is an open source obfuscation tool for Android. It is an extension of Proguard and offers code obfuscation for Java and Kotlin, along with symbol obfuscation feature from Proguard. 


## dProtect String Obfuscation

[DetectMagiskHide](https://github.com/darvincisec/DetectMagiskHide) is used. Following dProtect configuration is used to enable string obfuscation, along with other techniques: 

```
-keep,allowobfuscation class com.** { *; }

-obfuscations *
-obfuscate-strings      class com.darvin.security.** { *; }
-obfuscate-arithmetic   class com.darvin.security.** { *; }
-obfuscate-constants    class com.darvin.security.** { *; }
-obfuscate-control-flow class com.darvin.security.** { *; }

-repackageclasses
-allowaccessmodification
-flattenpackagehierarchy
-useuniqueclassmembernames
```

Post obfuscation, the code is transformed to following: 

```java
public class AppZygotePreload implements ZygotePreload {
    private static int a;
    private static long[] b;
 
    static {
        b = r0;
        long[] jArr = {1959339100, 1532247175, 496329810, 84648423, 1290985939, 139648832, 1345075629};
        a = ((int) jArr[6]) ^ 618995181;
    }
 
    public static String a(String str) {
        long[] jArr;
        StringBuilder sb = new StringBuilder();
        int i = ((int) b[1]) ^ 1532247175;
        while (i < str.length()) {
            char charAt = str.charAt(i);
            long[] jArr2 = b;
            int i2 = (((int) jArr2[2]) ^ 496366473) + charAt + (((int) jArr2[3]) ^ 84648422);
            int i3 = ~charAt;
            sb.append((char) ((i % (((int) jArr2[4]) ^ 1290935852)) ^ ((i2 + ((~(((int) jArr2[2]) ^ 496366473)) | i3)) - (((((int) jArr2[2]) ^ 496366473) + charAt) - (((charAt + (((int) jArr2[2]) ^ 496366473)) + (((int) jArr2[3]) ^ 84648422)) + ((~(((int) jArr2[2]) ^ 496366473)) | i3))))));
            do {
                jArr = b;
                int i4 = (((int) jArr[3]) ^ 84648422) + i + (((int) jArr[3]) ^ 84648422);
                int i5 = ~i;
                i = i + (((int) jArr[3]) ^ 84648422) + (((int) jArr[3]) ^ 84648422) + ((~(((int) jArr[3]) ^ 84648422)) | i5) + (((((int) jArr[3]) ^ 84648422) + i) - (i4 + ((~(((int) jArr[3]) ^ 84648422)) | i5)));
            } while ((a + (((int) jArr[3]) ^ 84648422)) % (((int) jArr[5]) ^ 139648834) == 0);
        }
        return sb.toString();
    }
 
    @Override // android.app.ZygotePreload
    public void doPreload(ApplicationInfo applicationInfo) {
        if (applicationInfo == null || Build.VERSION.SDK_INT < (((int) b[0]) ^ 1959339073)) {
            return;
        }
        System.loadLibrary(a("鞵鞻鞭鞱鞩鞻韰鞰鞺鞰"));
    }
}
```

Call to `System.loadLibrary()` expects a string input and dProtect has obfuscated the original string passed to this function. The original string is recovered during the runtime by calling `a("鞵鞻鞭鞱鞩鞻韰鞰鞺鞰")`. 

If we closely analyze the function `a`, the code is self-contained in the class, which means all the dependencies and logic required to execute this function is present in the same class. **There is no dependency on any other class. Such scenarious are perfect to handle using emulation.**

On using Katalina, the original string `native-lib` can be easily recovered:  

```
python3 Katalina/main.py -v classes.dex -x 'Lcom/darvin/security/AppZygotePreload;->a(Ljava/lang/String;)Ljava/lang/String;' '鞵鞻鞭鞱鞩鞻韰鞰鞺鞰'     
 
// --- output ----
...
INFO     String created: Lcom/darvin/security/AppZygotePreload;.a(LL) -> native-lib
INFO     String created: native-lib
...
```

## Patching Katalina

The existing implementation of Katalina does not handle all array use-cases properly (and mentioned in the Readme). dProtect's code is using `aget-wide` and `aput-wide` instructions, and to handle them it required patching Katalina. The patched code is available [here](https://github.com/su-vikas/Katalina).