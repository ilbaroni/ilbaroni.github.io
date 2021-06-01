---
layout: post
title:  "Is your windows legit? A possible anti sandbox trick"
date:   2021-02-26
categories: reversing
---

## Introduction

While thinking about anti-VM and anti-sandbox tricks I had the idea of implementing a simple check for a valid windows license. The goal is basically check if we are running on a legit Windows operating system and if not, abort the execution.

Use case for this? Well, evade automated analysis systems that use unlicensed images of Windows.

Google-fu pointed me to this api: [SLIsGenuineLocal()](https://docs.microsoft.com/en-us/windows/win32/api/slpublic/nf-slpublic-slisgenuinelocal?redirectedfrom=MSDN).

![image-20210226201255018](/assets/images/is_your_windows_legit/image-20210226201255018.png)

So it takes three arguments... The first argument is a pointer to an SLID structure that specifies the application to check.

The second argument is a pointer to an [SL_GENUINE_STATE](https://docs.microsoft.com/en-us/windows/win32/api/slpublic/ne-slpublic-sl_genuine_state) enum that specifies the state of the installation.

![image-20210226203644475](/assets/images/is_your_windows_legit/image-20210226203644475.png)

The third argument is used to show a dialog box so I'll leave it NULL.

The Proof-of-Concept program is really simple as it only checks if the current status is SL_GEN_STATE_IS_GENUINE and opens a message box with a message.

## Licensed Windows

![image-20210226211714915](/assets/images/is_your_windows_legit/image-20210226211714915.png)

## Unlicensed Windows

![image-20210226205131390](/assets/images/is_your_windows_legit/image-20210226205131390.png)
