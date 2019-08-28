---
layout: post
title:  "iOS Solid State NAND Storage"
date:   2015-11-21 16:17:22 +0800
---

There is not much literature available on how does NAND storage of Apple's iDevices is like. While reading "Hacking and Securing iOS Applications" by Jonathan Zdziarski, I came across how does the NAND storage looks like until iOS 5. As a note on caution, it is very possible that this structure has been changed in the subsequent iOS versions.

Knowing the structure of storage is an important step in order to understand how encryption works in iOS. The text below is verbatim from the book.
The NAND is divided into six separate slices:

- **BOOT**: Block zero is referred to as the BOOT block of the NAND, and contains a copy of Apple’s low level boot loader.

- **PLOG**: Block 1 is referred to as effaceable storage, and is designed as a storage locker for encryption keys and other data that needs to be quickly wiped or updated. The PLOG is where three very important keys are stored, which you’ll learn about in this chapter: the BAGI, Dkey, and EMF! keys. This is also where a security epoch is stored, which caused iOS 4 firmware to seemingly brick devices if the owner attempted a downgrade of the firmware.

- **NVM**: Blocks 2–7 are used to store the NVRAM parameters set for the device.

- **FIRM**: Blocks 8–15 store the device’s firmware, including iBoot (Apple’s second stage boot loader), the device tree, and logos.

- **FSYS**: Blocks 16–4084 (and higher, depending on the capacity of the device) are used for the filesystem itself. This is the filesystem portion of NAND, where the operating system and data are stored. The filesystem for both partitions is stored here.

- **RSRV**: The last 15 blocks of the NAND are reserved.

If there is any recent material on this topic, please let me know by commenting below.

Keep Hacking :D
