---
layout: post
title:  "RANSOMEXX linux ransomware"
date:   2021-01-05
categories: reversing
---

## Introduction

In this blog point I will go through a Linux sample of the ransomware family RansomEXX (aka Defray777). This ransomware group did some big attacks during 2020 and only Windows samples were known until [Kaspersky](https://securelist.com/ransomexx-trojan-attacks-linux-systems/99279/) revealed that the group also had a Linux build.

## Reversing

This ransomware uses AES in ECB mode with a 256 bit key for file encryption and RSA to encrypt the AES key used for file encryption. The encrypted key is then added as a blob of metadata to every encrypted file. This way the group guarantees that only them can recover the AES key used to encrypt the each file.

For encryption the ransomware uses a library named [mbedtls](https://tls.mbed.org) and it also implements a nice logic for speeding the encryption as it uses threads with in memory "queues".

So let's go through the main functionality of this sample and start with main function:

![main](/assets/images/ransomexx_linux/image1.png)

The function named GeneratePreData()  generates the 256 bit AES key and also encrypts it using RSA.

Generating a random AES key:

![AES key generation](/assets/images/ransomexx_linux/image2.png)

Importing the RSA key and encrypting the AES key:

![RSA key import and encryption](/assets/images/ransomexx_linux/image3.png)

Next, this function saves both the AES key and encrypted AES key to global variables. This way the workers responsible for the encryption can access this data.

![Global Variables](/assets/images/ransomexx_linux/image4.png)

The ransomware uses a mutex while accessing or writing to the global variables. This way the ransomware guarantees that no workers are accessing the global variables while the key is being changed and that the key is not changed when the workers are encrypting files.

![Mutex](/assets/images/ransomexx_linux/image5.png)

Right after generating the keys the ransomware creates a new thread:

![New Thread](/assets/images/ransomexx_linux/image6.png)

This new thread will generate a new AES key every 0.18 seconds. This way the ransomware ensures that all the files aren't encrypted with the same AES key.

![Updating the keys](/assets/images/ransomexx_linux/image7.png)

It is important to notice that the ransomware expects the paths for file encryption to be passed as arguments:

![Parsing Argv](/assets/images/ransomexx_linux/image8.png)

The function EnumFiles() takes a path as an argument and it will:

- Initialize the workers (threads) for file encryption.
- Recursively search the directory for files by calling list_dir().

The function list_dir() creates a ransomnote inside the current directory (where the search starts) by calling ReadMeStoreForDir() and it starts searching for files.

If a new file is found and it's not either the ransom note or an already encrypted file it will then be added to the workers "queue" in memory by calling add_task_to_worker(). 

If a directory is found and it's not either "." or ".." then the function calls itself again but this time with a new directory to search, and so on...

The following image shows exactly how this logic is implemented:

![Traversing directories](/assets/images/ransomexx_linux/image9.png)

The workers use AES in ECB mode to encrypt the files:

![AES encryption](/assets/images/ransomexx_linux/image10.png)

## Ransomnote

The ransomnote is written to a file with following name structure `!NEWS_FOR_<COMPANY>!.txt`. As seen, this sample was designed for [EIGS](https://www.eigsi.fr/):

![Ransomnote](/assets/images/ransomexx_linux/image11.png)

## Conclusion

Ransomware is nasty but it can be profitable for the threat actors and not even Linux is safe from this kind of threats. The best way to prevent this kind of threat is to always be prepared for it.

File:

```
SHA256: CB408D45762A628872FA782109E8FCFC3A5BF456074B007DE21E9331BB3C5849
```