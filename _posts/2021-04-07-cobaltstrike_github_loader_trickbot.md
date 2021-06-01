---
layout: post
title:  "Analysis of a COBALTSTRIKE Loader possibly related with TRICKBOT crew"
date:   2021-04-07
categories: reversing
---
## Introduction

As pointed out by security researchers from Walmart in [this](https://medium.com/walmartglobaltech/trickbot-crews-new-cobaltstrike-loader-32c72b78e81c) blog post, the Trickbot group seems to have a Cobalt Strike loader that uses GitHub to host encoded shellcode in the form of fake image files.

According to the blog post, the IP address 74.118.138[.]113 belongs to an actor that was involved in Trickbot/CS intrusions and ransomware operations.

![ ](/assets/images/cobaltstrike_github_loader_trickbot/image-20210406153604544.png)

In the image above, it's possible to see two communicating files, the CS loader that uses GitHub to host shellcode and a cobalt strike shellcode loader.

Extracting the beacon from the first file which is the cobaltstrike_shellcode.exe:

```
$ cobaltstrike_shellcode_exe_decrypt.py -f cobaltstrike_shellcode.exe_ -o beacon.bin

[+] yara match!
[+] initial xor key: 0xa47bde75
[+] encrypted data size: 206336 bytes (0x32600)
[*] starting to decrypt data...
[*] finished decrypting the data!
[+] MZ header was found!
[+] successfully wrote 206336 bytes to: beacon.bin
```

Beacon config:

```
{"BeaconType": ["HTTPS"], "Port": 443, "SleepTime": 60000, "MaxGetSize": 1048576, "Jitter": 0, "MaxDNS": 255, "C2Server": "bird.allsafelink.com,/cx,masks.allsafelink.com,/visit.js", "UserAgent": "Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; WOW64; Trident/5.0; NP09; NP09; MAAU)", "HttpPostUri": "/submit.php", "Malleable_C2_Instructions": [], "HttpGet_Metadata": ["Cookie"], "HttpPost_Metadata": ["Content-Type: application/octet-stream", "id"], "SpawnTo": "AAAAAAAAAAAAAAAAAAAAAA==", "PipeName": "", "DNS_Idle": "0.0.0.0", "DNS_Sleep": 0, "SSH_Host": "Not Found", "SSH_Port": "Not Found", "SSH_Username": "Not Found", "SSH_Password_Plaintext": "Not Found", "SSH_Password_Pubkey": "Not Found", "HttpGet_Verb": "GET", "HttpPost_Verb": "POST", "HttpPostChunk": 0, "Spawnto_x86": "%windir%\\syswow64\\rundll32.exe", "Spawnto_x64": "%windir%\\sysnative\\rundll32.exe", "CryptoScheme": 0, "Proxy_Config": "Not Found", "Proxy_User": "Not Found", "Proxy_Password": "Not Found", "Proxy_Behavior": "Use IE settings", "Watermark": 0, "bStageCleanup": "False", "bCFGCaution": "False", "KillDate": 0, "bProcInject_StartRWX": "True", "bProcInject_UseRWX": "True", "bProcInject_MinAllocSize": 0, "ProcInject_PrependAppend_x86": "Empty", "ProcInject_PrependAppend_x64": "Empty", "ProcInject_Execute": ["CreateThread", "SetThreadContext", "CreateRemoteThread", "RtlCreateUserThread"], "ProcInject_AllocationMethod": "VirtualAllocEx", "bUsesCookies": "True", "HostHeader": ""}
```

## Cobalt Strike GitHub Loader

The hash of the loader is the following:

````
0234f80c6fd3768f9619d6fcd50d775ec686719fcc665007bfd1606bbe787744
````

Few imports means that the necessary DLLs and APIs are loaded and resolved during execution.

![ ](/assets/images/cobaltstrike_github_loader_trickbot/image-20210406160542303.png) 

All the strings are encoded, so, simply checking the strings will not give any useful hints about what's going on. Example of an encoded string being loaded to the stack:

![ ](/assets/images/cobaltstrike_github_loader_trickbot/image-20210406161033498.png)

The loader starts by decoding the paths to KERNEL32.dll and NTDLL.dll.

```
C:\Windows\System32\kernel32.dll
C:\Windows\System32\ntdll.dll
```

After decoding them, the loader will "manually" read the contents of the two DLLs to memory.

![ ](/assets/images/cobaltstrike_github_loader_trickbot/image-20210406164133377.png)

To read the contents of the DLLs to memory, the loader uses the combination of VirtualAllocExNuma(), CreateFileW(), and ReadFile().

![ ](/assets/images/cobaltstrike_github_loader_trickbot/image-20210406164046171.png)

After reading the contents of the DLLs to memory, the loader resolves LoadLibraryW() from Kernel32.dll and NtAllocateVirtualMemory() from Ntdll.dll.

![ ](/assets/images/cobaltstrike_github_loader_trickbot/image-20210406165327499.png)



![ ](/assets/images/cobaltstrike_github_loader_trickbot/image-20210406165421176.png)

Finally, it decodes the string "wininet.dll" and uses the previously resolved API LoadLibraryW to load it to memory.

![ ](/assets/images/cobaltstrike_github_loader_trickbot/image-20210406165636093.png)

After loading the DLL, the loader decodes the following WinINet API names:

````
InternetOpenW
InternetConnectW
HttpOpenRequestW
HttpSendRequestW
InternetReadFile
InternetCloseHandle
````

And resolves them by parsing the exports directory of WININET.dll.

![ ](/assets/images/cobaltstrike_github_loader_trickbot/image-20210406172015364.png)

At this point, and with all the necessary APIs resolved the loader decodes the GitHub repository and uses the WinINet functions to download a fake JPG file.

Stack string with encoded GitHub repository:

![ ](/assets/images/cobaltstrike_github_loader_trickbot/image-20210406172722623.png)

Requesting the fake image:

![ ](/assets/images/cobaltstrike_github_loader_trickbot/image-20210406173021679.png)

After getting the contents of the fake image file, the loader decodes and executes it within memory.

At this point, the GitHub accounts are not active so I wasn't able to get the fake JPG file. According to the Wallmart blog post, the file contained a shellcode to download **[this](https://bazaar.abuse.ch/sample/210f032fe0af6d25bf6b74ce5393fdd3b90a741ca8ab716866765371dfe05327/)** cobalt strike beacon from:

```
cakes.rainbowmango.info/gifs20210122.dat
```

Beacon config:

```
{"BeaconType": ["HTTPS"], "Port": 443, "SleepTime": 6145, "MaxGetSize": 2097468, "Jitter": 55, "MaxDNS": "Not Found", "C2Server": "subs.rainbowmango.info ,/share/pink,food.rainbowmango.info,/share/pink", "UserAgent": "Not Found", "HttpPostUri": "/us", "Malleable_C2_Instructions": ["Remove 36 bytes from the end", "NetBIOS decode 'a'", "Remove 140 bytes from the end"], "HttpGet_Metadata": "Not Found", "HttpPost_Metadata": "Not Found", "SpawnTo": "AAAAAAAAAAAAAAAAAAAAAA==", "PipeName": "Not Found", "DNS_Idle": "Not Found", "DNS_Sleep": "Not Found", "SSH_Host": "Not Found", "SSH_Port": "Not Found", "SSH_Username": "Not Found", "SSH_Password_Plaintext": "Not Found", "SSH_Password_Pubkey": "Not Found", "HttpGet_Verb": "GET", "HttpPost_Verb": "POST", "HttpPostChunk": 0, "Spawnto_x86": "%windir%\\syswow64\\gpupdate.exe", "Spawnto_x64": "%windir%\\sysnative\\gpupdate.exe", "CryptoScheme": 0, "Proxy_Config": "Not Found", "Proxy_User": "Not Found", "Proxy_Password": "Not Found", "Proxy_Behavior": "Use IE settings", "Watermark": 1359593325, "bStageCleanup": "False", "bCFGCaution": "False", "KillDate": 0, "bProcInject_StartRWX": "False", "bProcInject_UseRWX": "False", "bProcInject_MinAllocSize": 16535, "ProcInject_PrependAppend_x86": "Empty", "ProcInject_PrependAppend_x64": "Empty", "ProcInject_Execute": ["CreateThread", "CreateRemoteThread", "ntdll.dll:RtlUserThreadStart", "SetThreadContext", "NtQueueApcThread-s", "kernel32.dll:LoadLibraryA", "RtlCreateUserThread"], "ProcInject_AllocationMethod": "VirtualAllocEx", "bUsesCookies": "True", "HostHeader": ""}
```

## IOCs

```
IP Addresses

74.118.138[.]113

Domains

bird.allsafelink[.]com
masks.allsafelink[.]com
lists.allsafelink[.]com
cakes.rainbowmango[.]info
subs.rainbowmango[.]info
food.rainbowmango[.]info

SHA256

3444d16e03eb092369a4c58fc2534b4ddbdb0000dedc8f735ef996aa08daa33d (cobaltstrike_shellcode.exe)
0234f80c6fd3768f9619d6fcd50d775ec686719fcc665007bfd1606bbe787744 (Cobalt Strike GitHub Loader)
210f032fe0af6d25bf6b74ce5393fdd3b90a741ca8ab716866765371dfe05327 (beacon downloaded via shellcode)
```
