---
layout: post
title:  "Solving iOS UnCrackable 1 Crackme Without Using an iOS Device"
date:   2019-10-28 16:17:22 +0800
---
**TL;DR:** iOS UnCrackable Level 1 crackme application can be solved without using an iOS device using Angr's dynamic execution engine.  

**Time To Read:** 5 min

OWASP MSTG [UnCrackable Level 1 Crackme App](https://github.com/OWASP/owasp-mstg/tree/master/Crackmes/iOS/Level_01) is an iOS crackme application. The goal of the crackme is to find a secret string hidden somewhere inside the application. There are multiple ways to solve the crackme using Frida, or a debugger, or even by manually reversing the native code. In this post I will solve the challenge using Angr's dynamic execution engine. With the current approach, by using static analysis assisted with dynamic execution, the crackme can be **solved without needing an iOS device.** 

To perform static analysis, multiple tools are available - Ghidra, R2 or IDA Pro. The application code is not obfuscated and can be easily followed in any of these tools. I am using Ghidra in this post.   

The application gives a message "Verification Failed" when a wrong input is provided to it. 

![Verification Failed Message](/assets/images/owasp_mstg_angr_ios_app_wrong_input.png "Verification Failed Message")

On searching the binary for this string, it is found in `buttonClick` function. The message is displayed after a comparison operation using `isEqualToString`. The comparison is being performed between the input string and the value of a label marked as *hidden*. 

![Decompilation of buttonClick function](/assets/images/owasp_mstg_angr_ghidra_buttonclick_decompiled.png "Decompilation of buttonClick function")

To find the secret string, we need to find the value of this label. Further, on analysing the function `viewDidLoad`, the value of the label is being set using the return value of the function at offset `0x1000080d4`.

![Decompilation of viewDidLoad function](/assets/images/owasp_mstg_angr_ghidra_viewdidload_decompile.png "Decompilation of viewDidLoad function")

Shifting our attention to function at `0x1000080d4`, there are multiple sub-function calls, and return values from each of these sub-functions is stored at an index of an array at address `0x10000dbf0`. This array is the hidden string we are looking for! Each of the above sub-functions are not too complicated, but nevertheless require some manual effort to reverse them and obtain the eventual hidden string.

![Decompilation of function at 0x1000080d4](/assets/images/owasp_mstg_angr_ghidra_native_disassembly.png "Decompilation of function at 0x1000080d4")

The function at `0x1000080d4` or any of its sub-functions are self contained, i.e, there are no library calls or system calls; we can easily run this code in any emulator like Unicorn or its likes. Angr is a reverse engineering framework with multiple features and one of them being to dynamically execute code. For using dynamic execution feature of Angr, we need to identify and pass the points in the program from where we want to emulate the code and the addresses where the eventual secret string generated. In this case, the function at `0x1000080d4` is the function we want to emulate and the return value is the information we are interested in. The return value is a pointer to the hidden string.

The eventual Angr script looks as following:

```python
import angr
import claripy

def solve():

    # Load the binary by creating angr project.
    project = angr.Project('uncrackable.arm64')

    # Pass the address of the function to the callable
    func = project.factory.callable(0x1000080d4)

    # Get the return value of the function
    ptr_secret_string = claripy.backends.concrete.convert(func()).value
    print("Address of the pointer to the secret string: " + hex(ptr_secret_string))

    # Extract the value from the pointer to the secret string
    secret_string = func.result_state.mem[ptr_secret_string].string.concrete
    print(f"Secret String: {secret_string}")

solve()
```

Angr `Project` is the basic building block, and must needed to access any functionality of Angr. Using `callable` API we inform the Angr engine to execute the code from the offset `0x1000080d4`, and the return value post *concrete* execution (dynamic execution is also called concrete execution) is captured in `ptr_secret_string`. The value stored in the pointer is accessed using `result_state`.

This solution is also contributed to the [OWASP MSTG](https://github.com/OWASP/owasp-mstg) and with some more details on reverse engineering part. 
