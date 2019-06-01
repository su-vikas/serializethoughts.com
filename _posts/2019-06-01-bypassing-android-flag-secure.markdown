---
layout: post
title:  "Bypassing Android FLAG_SECURE using FRIDA"
date:   2019-06-01 16:17:22 +0800
---

Since Android 5 via MediaProjection API, Android allows screen capturing and screen sharing using third party applications. I won't be going in detail of how this API work and what are its various security implications.  This article by [Nightwatch Cybersecurity](https://wwws.nightwatchcybersecurity.com/2016/04/13/research-securing-android-applications-from-screen-capture/) summarizes it very succinctly.

The important point to keep in mind is, to protect sensitive applications from screen capturing and sharing is to set [FLAG_SECURE](https://developer.android.com/reference/android/view/WindowManager.LayoutParams#FLAG_SECURE) flag for that respective screen.

Recently I came across an Android application using this very  FLAG_SECURE flag to prevent from screen capturing or sharing. I wrote a simple FRIDA script to bypass this check and it is very straightforward.  The script I used is below:

{% gist 36410f67c9e0127961ae344010c4c0ef %} 


There is an XPosed module as well which performs exactly same thing.



