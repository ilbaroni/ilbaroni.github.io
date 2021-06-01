---
layout: post
title:  "RAGNARLOCKER ransomware analysis"
date:   2020-11-28
categories: reversing
---

## Checking Operating System Language

Ragnar starts by the checking the language of the operating system and if it matches any of the following blacklisted languages it will terminate the process:

![image-20201126150804102](/assets/images/ragnarlocker/image-20201126150804102.png)

Blacklisted Languages:

```
Azerbaijan
Armenian
Belorussia
Kazakh
Kyrgyz
Moldavian
Tajik
Russian
Turkmen
Uzbek
Ukrainian
```

## Disabling Windows System Restore

Ragnar will try to disable the capability of windows to restore itself to a previous saved state. Ragnar does this by creating five new processes to run five different command as seen below:

![image-20201126164039109](/assets/images/ragnarlocker/image-20201126164039109.png)

The following commands will delete the windows shadow copies and disable windows automatic repair:

```
vssadmin delete shadows /all / quiet
wmic.exe shadowcopy delete
bcdedit /set {default} recoveryenabled No
bcdedit /set {default} bootstatuspolicy IgnoreAllFailures
bcdedit /set {globalsettings} advancedoptions false
```

## Killing Processes And Services

Ragnar will create two threads to search for specific processes and services to kill them:

![image-20201126170256229](/assets/images/ragnarlocker/image-20201126170256229.png) 

The strings are stored encrypted.

List of services to kill:

```
vss,sql,memtas,mepocs,sophos,veeam,backup,pulseway,logme,logmein,connectwise,splashtop,mysql,Dfs,vmms,vmcompute,Hyper-v
```

List of processes to kill:

```
sql,mysql,veeam,oracle,ocssd,dbsnmp,synctime,agntsvc,isqlplussvc,xfssvccon,mydesktopservice,ocautoupds,encsvc,firefox,tbirdconfig,mydesktopqos,ocomm,dbeng50,sqbcoreservice,excel,infopath,msaccess,mspub,onenote,outlook,powerpnt,steam,thebat,thunderbird,visio,winword,wordpad,EduLink2SIMS,bengine,benetns,beserver,pvlsvr,beremote,VxLockdownServer,postgres,fdhost,WSSADMIN,wsstracing,OWSTIMER,dfssvc.exe,dfsrs.exe,swc_service.exe,sophos,SAVAdminService,SavService.exe,Hyper-v
```

## File Encryption

Ragnar first forces all hidden volumes to be mounted before starting encrypting any files:

![image-20201126163206180](/assets/images/ragnarlocker/image-20201126163206180.png)

Then it starts iterating through the local system files and encrypt them using the Microsoft CryptoAPI to encrypt the files using RSA crypto.

Ragnar imports a RSA public key (which is stored encrypted and decrypted on runtime) to encrypt the files.

Extracted RSA key:

![image-20201126175354113](/assets/images/ragnarlocker/image-20201126175354113.png)

Importing the RSA key:

![image-20201126175736575](/assets/images/ragnarlocker/image-20201126175736575.png)

Finally, Ragnar encrypts all the files that were found on the mounted volumes however the following file names, folders and extensions are whitelisted:

```
autorun.inf
boot.ini
bootfont.bin
bootsect.bak
bootmgr
bootmgr.efi
bootmgfw.efi
desktop.ini
iconcache.db
ntldr
ntuser.dat
ntuser.dat.log
ntuser.ini
thumbs.db
.db
.sys
.dll
.lnk
.msi
.drv
.exe
Windows
Windows.old
Tor browser
Internet Explorer
Google
Opera
Opera Software
Mozilla
Mozilla Firefox
$Recycle.Bin
ProgramData
All Users
```

Ragnar also adds some metadata to the end of every encrypted file:

![image-20201127104730835](/assets/images/ragnarlocker/image-20201127104730835.png)

In the following image we can see the encrypted files and the ransomnote which was written to a txt file:

![image-20201127105752208](/assets/images/ragnarlocker/image-20201127105752208.png)

## Ransomnote

As seen in the previous section Ragnar writes a txt file with the ransomnote inside of it. The filename follows the following schema: `!$R4GN4R_XXX$!.txt` where XXX is a custom id based on the computer name:

![image-20201127110750623](/assets/images/ragnarlocker/image-20201127110750623.png)

The ransomnote is encrypted inside the binary and it is decrypted on runtime.

In the following image it's possible to see the decrypted ransomnote:

![image-20201127111410839](/assets/images/ragnarlocker/image-20201127111410839.png)

## Command Line Options

Ragnar locker payload support a few command line arguments such as:

```
-backup
-list
-force
-vm
-vmback
```

The difference on the behavior of this sample when we run it with command line arguments are minor. However, there were some intrusions where the samples of Ragnar Locker came with a VM image to encrypt the host files from inside the VM to avoid detections.

## To wrap-up

This sample of Ragnar Locker was first seen on VirusTotal on July 2020 as seen below:

![image-20201127112344952](/assets/images/ragnarlocker/image-20201127112344952.png)

SHA256:

```
04C9CC0D1577D5EE54A4E2D4DD12F17011D13703CDD0E6EFD46718D14FD9AA87
```

As seen on the ransomnote this payload was targeting CWT company which is a travel agency. Since no data of this company was leaked by the Ragnar Locker gang it looks that this company actually paid the ransom which was $4.5M.
