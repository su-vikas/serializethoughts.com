---
layout: post
title:  "Poking around with FRIDA's Java enumerateClassLoaders"
date:   2021-05-07 16:17:22 +0800
tags:
    - Android
    - Frida
---

**TL;DR:** *In Android, if classes are loaded by using non-default classloader, FRIDA will not be able to hook it. By using enumerateClassLoaders() API one can get the other classloaders used by the program and use them to hook Java classes.*

**Time To Read:** 5 min

## Java Classloaders

In Java, a [Classloader](https://en.wikipedia.org/wiki/Java_Classloader) is a part of Java Runtime Environment that dynamically loads Java classes into the Java Virtual Machine. By doing so, the Java run time does not need to know about files and file systems as this is delegated to the classloader. This concept is very much applicable with Dalvik bytecode for Android as well.

Often malwares and obfuscation tools change the default behaviour of class loading in Android. By default all the classes to be loaded are present in the classes.dex file and are loaded using the default `dalvik.system.PathClassLoader` classloader. But Android offers other classloaders too, like `dalvik.system.InMemoryDexClassLoader`, using which one can load a dex (containing Java classes) file from the memory.

## FRIDA

FRIDA will throw error while trying to hook a class not loaded using the default classloader. To this problem, FIRDA provides a easy solution via [`Java.enumerateClassLoaders()`](https://frida.re/docs/javascript-api/#java) API. You can iterate over the classloaders and use them by using `Java.ClassFactory.get(loader)` API. Once the new loader is set, the `.use()` API can be used to get the desired class and hook it.


```JS
Java.enumerateClassLoaders({
        onMatch: function(loader){
            Java.classFactory.loader = loader;
            var TestClass;

            // Hook the class if found, else try next classloader.
            try{
                TestClass = Java.use("com.example.TestClass");
                TestClass.testMethod.implementation = function(){
                    console.log("[+] Inside test method");
                }
            }catch(error){
                if(error.message.includes("ClassNotFoundException")){
                    console.log("\n You are trying to load encrypted class, trying next loader");
                }
            }
        },
        onComplete: function(){

        }
    });
```
