---
layout: post
title:  "Unpacking an ICEDID loader"
date:   2020-11-16
categories: reversing
---

In this blog post I will show how to unpack a recent IcedId loader. The file that I'm going to unpack is available [here](https://bazaar.abuse.ch/sample/4734648ab6553ba7bfb89776ac59a6ce1d71828ce69ef2e39e985a14e60f1372/).

# Introduction

Before starting let's understand what is a packer. Basically, to malware developers the goal of using a packer is to protect the final payload/executable file not only from anti virus but also from malware analysts by increasing as much as possible the time required for analysis.

After packing an executable, the new executable will contain 2 important things: the unpacking stub and the encrypted/compressed executable.

Now it comes the question, what is an unpacking stub? The unpacking stub is basically a piece of shellcode that will decrypt/decompress the packed code, allocate memory for it and transfer the execution to the original entry point (aka OEP). Although, some packers will try to inject the unpacked code to a remote process, so it's not always going to be equal when it comes to unpack code!

# Unpacking IcedId

So, IcedId loaders normally do self injection which means that they will decrypt and decompress the packed code and somehow transfer execution to it inside of it's own process memory. Note, that I will not be deep diving into the packing routine here and just on how to get the final unpacked executable.

So for this one, let's start by placing a breakpoint on VirtualAlloc and VirtualProtect:

![image-20201116220607458](/assets/images/unpacking_icedid_loader/image-20201116220607458.png)

![image-20201116220645144](/assets/images/unpacking_icedid_loader/image-20201116220645144.png)

Next, let's execute until we hit the breakpoint at VirtualAlloc for the second time and execute until return (**CTRL** + **F9**) to check the EAX value on the dump window:

![image-20201116220912006](/assets/images/unpacking_icedid_loader/image-20201116220912006.png)

It's empty as expected, as it's a new memory region. Let's continue by pressing **F9** and we should break again at VirtualAlloc. This time if we look close EDI is pointing to the string "PE":

![image-20201116221203583](/assets/images/unpacking_icedid_loader/image-20201116221203583.png)

If we follow the address on the dump window we should see a new PE file in memory:

![image-20201116221527131](/assets/images/unpacking_icedid_loader/image-20201116221527131.png)

So now all we have left to do is to select all the bytes since the beginning of the MZ header until the end of the section,copy the bytes (**right click on the selected bytes** > **Binary** > **Binary** **Copy**) and paste it on an hex editor like HxD:

![image-20201116221901826](/assets/images/unpacking_icedid_loader/image-20201116221901826.png)

Cool, so now let's save it to disk and check if it's all good on a PE viewer:

![image-20201116222000504](/assets/images/unpacking_icedid_loader/image-20201116222000504.png)

The unpacked executable looks like it's all correct. All the imports are there and there are no problems on the section headers since the executable wasn't mapped to memory.

Imports:

![image-20201116222115231](/assets/images/unpacking_icedid_loader/image-20201116222115231.png)

Section Headers:

![image-20201116222207007](/assets/images/unpacking_icedid_loader/image-20201116222207007.png)

That's all, we successfully unpacked an IcedId loader. IDA should open the file with no problems!

Note that this is not the final payload of IcedId, this one will be actually downloading the final payload from the C2.
