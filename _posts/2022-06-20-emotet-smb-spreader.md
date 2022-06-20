---
layout: post
title:  "Emotet SMB spreader overview"
date:   2022-01-30
categories: posts
---


Emotet is back in business and it's revealing some new tricks. Not long ago, Emotet introduced a new module, the Google Chrome's credit card grabber. More recently, the SMB spreader module has been brought back and is now, once again, part of the infection chain.

#### Loading the necessary DLLs

The very first step of the spreader module is to load and save pointers to the following DLLs:

```
shlwapi.dll
userenv.dll
crypt32.dll
advapi32.dll
shell32.dll
urlmon.dll
netapi32.dll
bcrypt.dll
wininet.dll
mpr.dll
wtsapi32.dll
```

These DLLs are used to resolve the Windows APIs as needed.

#### Getting the path of Emotet loader

The path of the Emotet loader is extracted directly from the command line of the current process. (This is possible since the modules run in the same process as the loader).

![image-20220620164744778](/assets/images/emotet_smb_spreader/image-20220620164744778.png)

The path will be used when the spreader tries to copy the loader to a remote share.

#### Hardcoded usernames and passwords

The spreader contains a list of hardcoded usernames and a list of passwords and uses them to bruteforce the IPC$ share from servers on the same network as the infected machine.

Decrypting and building a linked list of usernames:

![image-20220620170538430](/assets/images/emotet_smb_spreader/image-20220620170538430.png)

Decrypting and building a linked list of passwords:

![image-20220620171033412](/assets/images/emotet_smb_spreader/image-20220620171033412.png)

Full list of hardcoded usernames:

```
user,owner,operator,HP_Owner,HP_Administrator,administrator,admin
```

Full list of hardcoded passwords:

```
letmein,tigger,jennifer,999999,lovely,qazwsxedc,hunter,Password,147258369,q1w2e3r4t5,222222,andrew,123456789a,joshua,secret,samsung,starwars,11111111,nicole,1111,123abc,michelle,lol123,thomas,liverpool,jordan,soccer,Status,jessica,naruto,a123456,qwer1234,charlie,123654,0123456789,baseball,asd123,asdfgh,555555,aaaaaa,fuckyou,computer,1234561,abcd1234,1q2w3e,sunshine,7777777,master,azerty,qwe123,123456a,superman,1234qwer,qazwsx,asdasd,daniel,121212,shadow,michael,killer,football,112233,pokemon,asdfghjkl,123123123,q1w2e3r4,monkey,zxcvbnm,159753,123qwe,987654321,princess,ashley,dragon,666666,1qaz2wsx,password1,1qaz2wsx3edc,qwerty123,654321,qwertyuiop,1q2w3e4r,123321,000000,123,iloveyou,q1w2e3r4t5y6,1q2w3e4r5t,abc123,1234567,1234567890,111111,1234,123123,12345,12345678,qwerty,password,123456789,123456
```

#### Impersonating the logged on user

The token from the logged-on user account is duplicated using the `SecurityImpersonation` level, which, according to MSDN, allows a process to impersonate the client's security context on the local system:

![image-20220620171912056](/assets/images/emotet_smb_spreader/image-20220620171912056.png)

After duplicating the user token, the spreader calls `ImpersonateLoggerOnUser` to finally let its thread impersonate the security context of the logged-on user:

![image-20220620172155069](/assets/images/emotet_smb_spreader/image-20220620172155069.png)

#### Finding remote servers

To find remote servers the spreader uses two WinAPIs, `WnetOpenEnumW` and `WnetEnumResourceW`. If the network resource is a server, the name gets saved to a linked list of remote server names:

![image-20220620173448227](/assets/images/emotet_smb_spreader/image-20220620173448227.png)

#### Moving laterally

The spreader iterates over the list of servers and bruteforces the IPC$ share with the hardcoded users and passwords:

![image-20220620190132580](/assets/images/emotet_smb_spreader/image-20220620190132580.png)

> The function fn_connect_2_network_resource uses the WinAPI WNetAddConnection2W with the hardcoded credentials.

If no valid credentials are found, it tries to fetch usernames from the target server using the WinAPI `NetUserEnum`:

![image-20220620175820397](/assets/images/emotet_smb_spreader/image-20220620175820397.png)

All fetched usernames not present in the hardcoded list will be bruteforced with the hardcoded password list.

If the spreader finds valid credentials for the IPC$ share, it tries to connect to ADMIN$ and C$ shares:

![image-20220620183236111](/assets/images/emotet_smb_spreader/image-20220620183236111.png)

If the spreader connects successfully to one of these shares, the Emotet loader gets copied to the share with a random filename (derived from the machine CPU counter) and launched as a service.

Paths to where the loader can be copied:

```
C:\<random>.dll
%SystemRoot%\<random>.dll
```

The newly created service will execute one of the following commands:

```
regsvr32.exe "C:\<random>.dll"
regsvr32.exe "%SystemRoot%\<random>.dll"
```

#### IOCs

* [3D8F8F406A04A740B8ABB1D92490AFEF2A9ADCD9BEECB13AECF91F53AAC736B4](https://www.virustotal.com/gui/file/3d8f8f406a04a740b8abb1d92490afef2a9adcd9beecb13aecf91f53aac736b4) - SMB spreader module