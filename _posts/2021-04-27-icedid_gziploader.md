---
layout: post
title:  "Let's take a look at the new ICEDID GZIPLOADER"
date:   2021-04-27
categories: reversing
---

# About ICEDID

ICEDID is one of the most sophisticated malware families currently out there. It is a banking trojan that steals banking data and even personal information from the victims. 

One of the interesting aspects of ICEDID is that it uses in-memory payloads, only the loaders touch the disk of the infected machines.

The old loader was called PHOTOLOADER, but as discovered by the Binary Defence threat intelligence team, there is a new ICEDID loader that has been delivered by a QAKBOT affiliate using malicious EXCEL documents.

**Example of the malicious EXCEL doc:**

![ ](/assets/images/icedid_gziploader/image-20210427085533856.png)

These EXCEL files make usage of Excel 4.0 macros, which is a legacy way for writing macros. 

Several adversaries use this type of macros because of the low detection rates observed against several AV engines.

Once the macros are enabled the document will download a malicious DLL and execute it using "rundll32.exe" with the parameter "DllRegisterServer".

Although, there are some cases where these EXCEL files use "regsvr32.exe" to execute the DLL.

Below the process tree of a recent ICEDID infection starting with the malicious EXCEL doc:

![ ](/assets/images/icedid_gziploader/image-20210427085616203.png)

**Detection opportunity:**

Watch for "rundll32.exe" and "regsvr32.exe" processes being executed under office programs such as "winword.exe" and "excel.exe".

# GZIPLOADER Stage 1

This first loader is responsible for doing host reconnaissance and downloads the next stage from a remote C2 server that looks like a GZIP file, hence the name GZIPLOADER.

## Technical details

The first step of the loader is to decrypt its config and make an HTTPS GET request to a legit domain (aws.amazon.com, in this case):

![ ](/assets/images/icedid_gziploader/image-20210427103003644.png)

Next, it performs some host reconnaissance to build the HTTP cookie headers and tries to connect to the decrypted C2:

![ ](/assets/images/icedid_gziploader/image-20210427110831638.png)

Looking at the HTTP response from the C2 it looks like a GZIP file because of the initial bytes (`\x1f\x8B`) .

Rather than a legit GZIP file, it's encrypted data containing another loader and the encrypted main ICEDID bot.

**HTTP GET and response masqueraded as a GZIP file:**

![ ](/assets/images/icedid_gziploader/image-20210427091907226.png)

Summary of the ICEDID loader cookies (good for Suricata rules):

```
__gads
_gat
_ga
_u
__io
_gid
```

Next, the loader decodes the fake GZIP data:

![ ](/assets/images/icedid_gziploader/image-20210427110918610.png)

And writes the two decoded files to disk and transfer execution to the second loader:

![ ](/assets/images/icedid_gziploader/image-20210427111043403.png)

Example of the file paths on disk:

```
C:\Users\admin\AppData\Local\Temp\vessel-64.dat (2nd loader)
C:\Users\admin\AppData\Roaming\VisitLeader\license.dat (ICEDID encrypted main bot)
```

Example of the command line arguments used to launch the second loader:

 ![ ](/assets/images/icedid_gziploader/image-20210427115239680.png)

**Detection opportunity:** 

Watch for the creation of files named "license.dat" within the Appdata\Roaming directory, and "rundll32.exe" processes using that file as part of the command line arguments.

# GZIPLOADER Stage 2 - Main Loader

This second stage loader is a big executable that will decrypt the main ICEDID bot (license.dat file) and load it to memory for execution.

With the script provided by Binary Defence (check the references section for the link), it's possible to decrypt both the "license.dat" payload and extract the config from the main loader.

Example of the output provided by the tool when we try to pull the config from the second stage loader:

```
$ python3 ../IcedDecrypt/IcedDecrypt.py -f vessel-64.dat
[+] Warning: Not Guaranteed to work
Size of data: 604
Size: 604
[+] Finished Decrypting And Extracting Data
{'BuildID': 1820688957, 'uri': '/news/', 'c2_1': 'timerework.fun', 'c2_2': 'pexxota.space', 'UnknownData': 'iReKyXISE0cOwX2V2hpN7StXQtI='}
```

For more information about the GZIP algorithm used by ICEDID, check the Binary Defence Analysis blog post in the references section.

# Indicators

**File hashes:**

| SHA1                                     | Description                 |
| ---------------------------------------- | --------------------------- |
| b18ff26e82916eba8a7a69e5f5ccf6ba0f87ccf9 | document-2073999746.xls     |
| 1717402591b663767b37b3ce0635d991ace08432 | stage 1 loader (gziploader) |
| 52286ca71ac4239c5e2faad25e569f83ca4b35ee | stage 2 loader              |
| 0febc376cc066bb668f1a80b969ed112da8e871c | license.dat                 |

**URLs:**

```
hxxp://rlvq27rmjej02sfvb.com/fedara.gif
hxxp://berxion9.online/
hxxps://timerework.fun/users/1820688957/3327032839
```

**ICEDID C2 servers:**

```
berxion9.online
timerework.fun
pexxota.space
```

# References

* [**IcedID GZIPLOADER Analysis - Binary Defence**](https://www.binarydefense.com/icedid-gziploader-analysis/)
* **[IcedID Decryption Tool - Binary Defence](https://github.com/BinaryDefense/IcedDecrypt)**
* [**IcedID PhotoLoader evolution - @sysopfb**](https://sysopfb.github.io/malware,/icedid/2020/04/28/IcedIDs-updated-photoloader.html)

