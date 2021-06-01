---
layout: post
title:  "COBRALOCKER ransomware"
date:   2021-01-24
categories: reversing
---
# Introduction

This is a .NET ransomware that encrypts the files using AES 256 in CBC mode using as key the SHA256 hash of a random password generated during runtime.

# Overview of the ransomware

This ransomware targets all files under the following locations:

```
C:\\Users\\<USERNAME>\\Desktop
C:\\Users\\<USERNAME>\\Downloads
C:\\Users\\<USERNAME>\\Documents
C:\\Users\\<USERNAME>\\Pictures
```
Password geration function:

![passgen](/assets/images/cobralocker/passgen.PNG)
As seen, the ransomware creates a password with a length of 999:

![passcreation](/assets/images/cobralocker/passcreation.PNG)

File encryption using the SHA256 of the generated password as the AES key:

![crypt](/assets/images/cobralocker/crypt.PNG)

After encrypting the files the following ransomnote is written to **C:\\Users\\<USERNAME>\\Desktop\\readme.txt**:

```
Ooops! All your important files are encrypted!

All you important files are encrypted with AES 256 algoritm. No one can help you to restore files without our special decryptor.
All repair tools are useless.
If you want to restore some your files for free write to email and attach 2-3 encrypted files 
(non-archived and your files should not contain valuable information like databases, backups, large excel sheets etc.)
You have to pay $300 in bitcoin to decrypt other files.
As soon as we get bitcoins you'll get all your decrypted data back.

P.S.
Remember we are not scammers

Contact:
1.Download tor browser (https://www.torproject.org/)
2.Create account on mail2tor (http://mail2tor2zyjdctd.onion/)
3.Write e-mail to us (CobraLocker@mail2tor.com)

That's all
Good luck and have fun
```
The ransomware will check the running processes and look for the following:
```
cmd
regedit
Processhacker
sdclt
powershell
processhacker
Processhacker2
gpedit
gpedit.msc
```
If it finds any of the processes it will hide its window.

The ransomware executes the following command which deletes **C:\\bootmgr** and **C:\\Windows\\System32\\LogonUI.exe**:

```
cmd.exe /k takeown /f C:\\bootmgr && icacls C:\\bootmgr /grant %username%:F && del C:\\bootmgr && takeown /f C:\\Windows\\System32\\LogonUI.exe && icacls C:\\Windows\\System32\\LogonUI.exe && del C:\\Windows\\System32\\LogonUI.exe && exit
```
I made a quick PoC of the password generation function and I wonder how they will decrypt the files with the "special decryptor" lol.
![poc](/assets/images/cobralocker/poc.PNG)

# IOCs
file hash:
```
6a85ff1bbaf1db330fb4e737074ca8f3dbd7ff4a4897a0af26e56751ad8c9cff
```

