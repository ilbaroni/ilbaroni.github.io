---
layout: post
title:  "Is your windows legit? Anti sandbox"
image_alt: {
    url: "/assets/images/is_your_windows_legit_preview.png",
    source: "self"
}
excerpt: "While thinking about anti-VM and anti-sandbox tricks, I came up with the idea of ​​implementing a simple check to confirm whether the operating system has a valid license or not."
date:    2021-02-26 00:00:00 -0000
categories: []
---

While thinking about anti-VM and anti-sandbox tricks, I came up with the idea of ​​implementing a simple check to confirm whether the operating system has a valid license or not.

After some google-fu I found this Windows API: [SLIsGenuineLocal()](https://docs.microsoft.com/en-us/windows/win32/api/slpublic/nf-slpublic-slisgenuinelocal?redirectedfrom=MSDN).

![](/assets/images/is_your_windows_legit1.png)

The first argument is a pointer to an SLID structure that specifies the application to check, the second argument is a pointer to an [SL_GENUINE_STATE](https://docs.microsoft.com/en-us/windows/win32/api/slpublic/ne-slpublic-sl_genuine_state) enum that specifies the state of the installation, and the third can be NULL.

![](/assets/images/is_your_windows_legit2.png)

The proof-of-concept code is really simple as it only checks if the current status is SL_GEN_STATE_IS_GENUINE and opens a message box with the result.

Licensed Windows:

![](/assets/images/is_your_windows_legit3.png)

Unlicensed Windows:

![](/assets/images/is_your_windows_legit4.png)
