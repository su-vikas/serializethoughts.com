---
layout: post
title:  "Bypassing SSL Pinning in Android Applications"
date:   2016-08-18 16:17:22 +0800
---

It is a common practice for Android and iOS applications' developers to implement SSL Pinning in order to make reverse engineering of the apps difficult. As per [OWASP](https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning), SSL Pinning can be defined as the process of associating a host (in this case the app), with their expected X509 certificate or public key. Applications communicating over HTTPS and using SSL Pinning makes it non-trivial to perform Man-In-The-Middle attack and grab the network traffic in clear text using the proxy tools. For further reading about SSL Pinning, I would recommend [OWASP article](https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning) to get started with.

In this post, I will be looking into the steps involved in bypassing SSL Pinning checks for an Android application and show how we can patch the application binaries in order to intercept the traffic. For this post I am using Facebook Messenger (FBM) application as an example.

# SSL Pinning In Android
SSL Pinning in case of Android can be performed either in the Java layer, using the Android API, or in the native C/C++ layer. Lets look into each of the cases one at a time:

## Java Layer

To implement SSL Pinning, Android API exposes multiple functions to do so.  Android developer [website](https://developer.android.com/training/articles/security-ssl.html) provides a good overview about the topic.  In order to bypass the SSL Pinning in Java layer one can use existing tools or can patch the APK file manually.

- **Xposed Framework**: If the device is rooted, one can install Xposed Framework and use one of the multiple modules available to disable SSL Pinning. One such module is SSL Unpinning. Using the module is straight forward and I would leave the details of usage to the readers to figure out.
- **Manual Patching**:  Xposed Framework requires the device to be rooted. In such a case we cannot use the tools discussed above to bypass the SSL checks. In such a situation we can patch the APK file manually. Patching the file manually requires some extra effort, but given the abundance of exiting tools, this can be done with ease. The steps involved are following:

1. Decompile the application using Apktool or any other similar tool. Apktool gives Smali code for the application.
2. Patch the relevant functions in the Smali code.
3. Compile the application back using apktool, sign it using jarsign and run zipalign over it.
4. Installed the patched APK generated above.

The above approach is discussed in detail in this [blog post](http://blog.dewhurstsecurity.com/2015/11/10/mobile-security-certificate-pining.htm). Hence, I will refrain from duplicating the information and recommend you to read it there.

## Native Layer

Now enters a bit more challenging part. If the above approaches fail, you can fairly be confident that the SSL Pinning checks are being performed in the native layer. FBM is doing exactly same. To make things a bit obscure, the FBM application do have SSL Pinning logic in Java layer as well, but patching it does not work.

To get started, simply run APKTool and get the decompiled/unzipped version of the APK. If you look in the lib folder, there are 6 native libraries.

![FB Libraries](/assets/images/fb_libs.png)

After exploring these libraries using IDA Pro, to my surprise, none was having network related code. Which entailed more research is required to find the relevant code. If you look into these existing native libraries, you will find there is logic for xz decoding and logic for loading more native libraries. Also, if you go on and install the application and after the first usage, you will find that the */data/data/com.facebook.orca* directory have a folder *lib-xzs*, which contains another 30+ libraries.

![libliger](/assets/images/libliger.png)

So now the work is cut out clearly, we have to look into these libraries to find the network code. After evaluating each library using IDAPro, the code for SSL Pinning was found in libliger-native.so. There are many functions corresponding to SSL Pinning and network in this library. On seeing the various functions,  proxygen::SSLVerification::verifyWithMetrics() seems to be an easy target to fix. The function performs a comparison of the certificate received against the desired certificate embedded/pinned in the application and returns true or false. The pseudocode is shown below:

![VerifywithMetrics](/assets/images/verifywithmetrics1.jpg)

- **Dynamic Patching**: Dynamic library can be patched by performing dynamic injection and patching the function during the runtime. [ADBI](https://github.com/crmulliner/adbi) and [FRIDA](https://frida.re) are the two frameworks that provide such functionality. After some fiddling around with the above mentioned tools,  in both the cases FBM was crashing whenever an injection is made. Whether it was a security defense mechanism or instability of the process post injection, I left this unexplored for the time being. A note to the readers, to use these framework you need a rooted device.

- **Native Library Patching**: Given the dynamic injection is causing frequent crashes, patching the binary manually and reloading it again seems to be the only other option to go with. I used IDAPro to patch the binary. At the offset  0x0006732E patch code from 0xB948 (CBNZ R0, loc_6744) to 0xB9B8 (CBNZ R0, loc_6760). Where the block at loc_6760 is responsible for setting the return value, variable 'result', to always true. To get more understanding how the ARMv7 instructions represented at the bit level, read this [answer](http://stackoverflow.com/questions/9279451/armv7-word-patch-cbnz). After patching, replace the libliger-native.so file in /data/data/com.facebook.orca/lib-xzs with the new one, restart the application and now you can intercept the traffic.

Before patching library looks like following: 
![before_patching](/assets/images/before_patching.jpg)

After patching: 
![After Patching](/assets/images/patched.jpg)

Recently, there was another [blog post](https://eaton-works.com/2016/07/31/reverse-engineering-and-removing-pokemon-gos-certificate-pinning/) discussing patching of Android application (Pokemon Go app) to bypass SSL Pinning and I would recommend to read that one as well. The approach taken in that post is slightly different, as each application has different implementation.

Keep hacking :)
