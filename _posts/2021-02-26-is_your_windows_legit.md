---
layout: post
title:  "Is your windows legit? Anti sandbox"
date:   2021-02-26
categories: posts
---

## Introduction

While thinking about anti-VM and anti-sandbox tricks I had the idea of implementing a simple check for a valid windows license. The goal is basically check if we are running on a legit Windows operating system and if not, abort the execution.

Google-fu pointed me to this api: [SLIsGenuineLocal()](https://docs.microsoft.com/en-us/windows/win32/api/slpublic/nf-slpublic-slisgenuinelocal?redirectedfrom=MSDN).

![ ](/assets/images/is_your_windows_legit/image-20210226201255018.png)



The first argument is a pointer to an SLID structure that specifies the application to check, the second argument is a pointer to an [SL_GENUINE_STATE](https://docs.microsoft.com/en-us/windows/win32/api/slpublic/ne-slpublic-sl_genuine_state) enum that specifies the state of the installation, and the third can be NULL.

![ ](/assets/images/is_your_windows_legit/image-20210226203644475.png)



The proof-of-concept code is really simple as it only checks if the current status is SL_GEN_STATE_IS_GENUINE and opens a message box with a message.



## Licensed Windows

![ ](/assets/images/is_your_windows_legit/image-20210226211714915.png)



## Unlicensed Windows

![ ](/assets/images/is_your_windows_legit/image-20210226205131390.png)
