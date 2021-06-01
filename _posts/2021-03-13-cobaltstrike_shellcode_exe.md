---
layout: post
title:  "Extracting the beacon from cobaltstrike_shellcode.exe files"
date:   2021-03-13
categories: reversing
---

# Introduction

In this blog post, I will show what was my approach to statically extract indicators from these `cobaltstrike_shellcode.exe` executables.

As seen below, there are a lot of these executables in VirusTotal.

![](/images/cobaltstrike_shellcode_exe/vt.png)

# Reversing engineering the executable

The behavior of these files is very simple to understand as they just decrypt a large blob of data (which is the cobalt strike beacon) and transfer execution to it.

These executables start with the popular **jmp/call/pop** technique to start executing the decryption stub.

![ ](/images/cobaltstrike_shellcode_exe/jmp_call_pop.png)

The call instruction seen at the bottom of the previous image will place the address of the next instruction on the stack, which in this case will be a pointer to data that is crucial to the decryption routine.

As seen below, the first DWORD is the initial XOR key used in the decryption routine, the second DWORD is a value that when it is XOR'd with the first DWORD will give out the size of the encrypted data and the third DWORD is here the encrypted data starts.

![ ](/images/cobaltstrike_shellcode_exe/data.png)

The decryption routine will basically loop through the encrypted data and XOR every 4 bytes with the key.

The first 4 encrypted bytes are decrypted using the hardcoded XOR key while the next will be decrypted with the previous 4 decrypted bytes. Hope that makes sense :)

To test the decryption routine I made a quick IDA script.

![ ](/images/cobaltstrike_shellcode_exe/ida_python.png)

So now at this point, I only needed to automate the extraction of the encrypted beacon payload and try to extract the config from it. 

But how could I achieve this statically? Well, the solution that I found to be more reliable was to develop a Yara signature that was good enough to catch the main shellcode body in a data set of more or less 100 samples and automate the extraction of the embedded beacon.

Below it's possible to see my Yara rule matching the whole data set.

![ ](/images/cobaltstrike_shellcode_exe/yara.png)

Now it was just a matter of using it in a python script to get the offset to the shellcode, extract the data required to emulate the decryption routine, and dump the decrypted cobalt strike beacon.

![ ](/images/cobaltstrike_shellcode_exe/decryption.png)

Now let's try to parse the beacon config with the [Sentinel One extractor](https://github.com/Sentinel-One/CobaltStrikeParser).

![ ](/images/cobaltstrike_shellcode_exe/beacon_config.png)

As seen, it worked!

**Note:** Sometimes the beacons may have the "MZ" magic bytes stripped but that will be ok as long as the Yara rule keeps matching on the shellcode.

![ ](/images/cobaltstrike_shellcode_exe/no_mz.png)

Now with a working script to decrypt the beacon and a working beacon config extractor, I can combine them to extract indicators automatically.

That's it.

**Full script** can be found in [**this gist**](https://gist.github.com/jnzer0/54a7a153d49b4e9b9e31ecc654f9b80d)

