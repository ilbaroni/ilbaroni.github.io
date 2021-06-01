---
layout: post
title:  "XFILES stealer"
date:   2021-03-29
categories: reversing
---
# Introduction

In this blog post I will go through a X-Files stealer sample and will also share some indicators that can be used for detection purposes.

This stealer goes by the name of X-Files and it's being sold on underground forums.

![ ](/assets/images/xfiles_stealer/advertise1.jpeg)

![ ](/assets/images/xfiles_stealer/advertise2.jpeg)

![ ](/assets/images/xfiles_stealer/advertise3.jpeg)

(Credit to [3xp0rt](https://twitter.com/3xp0rtblog) for the tweet and images about this stealer.)

The stealer business model is based on a subscription with the following options.

```
250 RUB for 7 days
500 RUB for 1 month
1500 RUB for lifetime access
```

The panel is on Telegram and the builds are made on the developer server. The following telegram handles are shared on the advertised post.

```
Bot - @xFilesStealerBot
Support - @XFilesStealer
```

The stealer targets the following browsers.

```
Google Chrome, Chromium, Slimjet, Vivaldi and Opera
```

# Quick look into X-Files Stealer

The main function.

![ ](/assets/images/xfiles_stealer/image-20210329125411439.png)

First, it opens the registry key HKEY_CURRENT_USER and creates a BrowserInfoReader() object which has all the functions needed to get the data from the browsers. 

![ ](/assets/images/xfiles_stealer/image-20210329125916185.png)

Next, it creates five new directories if they don't exist.

![ ](/assets/images/xfiles_stealer/image-20210329130024172.png)

List of created directories.

```
%APPDATA%\Stealer
%APPDATA%\Stealer\Logins
%APPDATA%\Stealer\Cards
%APPDATA%\Stealer\Cookies
%APPDATA%\Stealer\Desktop
```

Steals the information from the browsers (logins, cards, and cookies), grabs files from the user desktop, takes a screenshot, and saves all the information inside a zip file.

![ ](/assets/images/xfiles_stealer/image-20210329131214995.png)

**GetLogins() function**

This function will steal any saved logins from the browsers.

![ ](/assets/images/xfiles_stealer/image-20210329150239745.png)

The following files will be created to save the saved logins.

```
%APPDATA%\Stealer\Logins\Google Chrome.txt
%APPDATA%\Stealer\Logins\Chromium.txt
%APPDATA%\Stealer\Logins\Slimjet.txt
%APPDATA%\Stealer\Logins\Vivaldi.txt
%APPDATA%\Stealer\Logins\Opera GX.txt
```

**GetCards() function**

This function will steal any saved credit card data from the browsers.

![ ](/assets/images/xfiles_stealer/image-20210329145640140.png)

The following files will be created to save the credit card data.

```
%APPDATA%\Stealer\Cards\Google Chrome.txt
%APPDATA%\Stealer\Cards\Chromium.txt
%APPDATA%\Stealer\Cards\Slimjet.txt
%APPDATA%\Stealer\Cards\Vivaldi.txt
%APPDATA%\Stealer\Cards\Opera GX.txt
```

**GetCookies() function**

This function will grab any available cookies from the browsers.

![ ](/assets/images/xfiles_stealer/image-20210329134124530.png)

The following files will be created to save the cookies.

```
%APPDATA%\Stealer\Cookies\Google Chrome.txt
%APPDATA%\Stealer\Cookies\Chromium.txt
%APPDATA%\Stealer\Cookies\Slimjet.txt
%APPDATA%\Stealer\Cookies\Vivaldi.txt
%APPDATA%\Stealer\Cookies\Opera GX.txt
```

**Grab() function**

This function will search for files within the user desktop folder with the following extensions: .txt, .png, .jpg, .doc.

![ ](/assets/images/xfiles_stealer/image-20210329133515058.png)

The files that match the extensions are copied to the following directories.

```
%APPDATA%\Stealer\Desktop\Images
%APPDATA%\Stealer\Desktop\Txt
%APPDATA%\Stealer\Desktop\Docs
```

**Screenshot() function**

This function will take a screenshot and save it to a file.

![ ](/assets/images/xfiles_stealer/image-20210329150702136.png)

Saved screenshot file.

```
%APPDATA%\Stealer\Screenshot.png
```

# Exfiltration

After stealing all the information and take the screenshot, the directory `%APPDATA%\Stealer` is compressed and saved as a zip file to the following path.

```
%APPDATA%\<Public IP>.zip
```

The zip file is then sent to the hardcoded C2 server via HTTP.

![ ](/assets/images/xfiles_stealer/image-20210329151105913.png)

Sample hardcoded C2.

```
178.154.251.1
```

C2 exfiltration endpoint.

```
/log.php
```

Stolen data being sent to the C2 server.

![ ](/assets/images/xfiles_stealer/image-20210329154240195.png)

C2 server response.

![ ](/assets/images/xfiles_stealer/image-20210329154351674.png)

# Cleaning the stolen data

After sending the stolen data to the remote C2, the malware deletes the zip file from the disk, deletes the directory containing all the stolen information, and sets a value in the HKCU registry key.

![ ](/assets/images/xfiles_stealer/image-20210329131907255.png)

Registry value added.

```
Registry key: HKCU
Name: "started"
Value: "true"
```

# Conclusion

This stealer is not very sophisticated at all and there are lots of detection opportunities for its behavior. The next section contains the consolidated list of IOCs that can be used for the general detection of this stealer.

# IOCs

```
Created directories:

%APPDATA%\Stealer
%APPDATA%\Stealer\Logins
%APPDATA%\Stealer\Cards
%APPDATA%\Stealer\Cookies
%APPDATA%\Stealer\Desktop
%APPDATA%\Stealer\Desktop\Images
%APPDATA%\Stealer\Desktop\Txt
%APPDATA%\Stealer\Desktop\Docs

Created files:

%APPDATA%\Stealer\Logins\Google Chrome.txt
%APPDATA%\Stealer\Logins\Chromium.txt
%APPDATA%\Stealer\Logins\Slimjet.txt
%APPDATA%\Stealer\Logins\Vivaldi.txt
%APPDATA%\Stealer\Logins\Opera GX.txt
%APPDATA%\Stealer\Cards\Google Chrome.txt
%APPDATA%\Stealer\Cards\Chromium.txt
%APPDATA%\Stealer\Cards\Slimjet.txt
%APPDATA%\Stealer\Cards\Vivaldi.txt
%APPDATA%\Stealer\Cards\Opera GX.txt
%APPDATA%\Stealer\Cookies\Google Chrome.txt
%APPDATA%\Stealer\Cookies\Chromium.txt
%APPDATA%\Stealer\Cookies\Slimjet.txt
%APPDATA%\Stealer\Cookies\Vivaldi.txt
%APPDATA%\Stealer\Cookies\Opera GX.txt
%APPDATA%\Stealer\Screenshot.png
%APPDATA%\<Public IP>.zip

Registry key values set:

Registry key - HKCU
Name - "started"
Value - "true"

C2 exfiltration:

http://178.154.251.1/log.php

C2 exfiltration (generic):

http://<IP>/log.php
```

yara

```
rule WIN_XFILES_STEALER
{
    meta:
        author="!j"
        description="X-Files stealer payload"
        date="20210329"
    
    strings:
        $pdb = "C:\\Users\\Administrator\\Desktop\\Svc_host\\Svc_host\\obj\\Release\\Svc_host.pdb" wide ascii

        $str1 = "started" wide ascii
        $str2 = "true" wide ascii
        $str3 = "/Windows.txt" wide ascii
        $str4 = "/Screenshot.png" wide ascii
        $str5 = ".zip" wide ascii
        
        $path1 = "\\Stealer" wide ascii
        $path2 = "\\Logins" wide ascii
        $path3 = "\\Cards" wide ascii
        $path4 = "\\Cookies" wide ascii
        $path5 = "\\Desktop" wide ascii
        $path6 = "/Images" wide ascii
        $path7 = "/Txt" wide ascii
        $path8 = "/Docs" wide ascii
        $path9 = "/Txt/" wide ascii
        $path10 = "/assets/images/" wide ascii
        $path11 = "/Docs/" wide ascii

        $url1 = "http://checkip.dyndns.org" wide ascii
        $url2 = "http://ipinfo.io/" wide ascii
        
        $c2exfil = "/log.php" wide ascii

        $sql1 = "SELECT origin_url,  username_value, password_value FROM logins" wide ascii
        $sql2 = "SELECT name_on_card,  expiration_month, expiration_year, card_number_encrypted FROM credit_cards" wide ascii
        $sql3 = "SELECT host_key, name, path, is_secure, expires_utc, encrypted_value, is_httponly FROM cookies" wide ascii

    condition:
        $pdb or ( all of ($str*) and all of ($path*) and all of ($url*) and $c2exfil and all of ($sql*) )
}
```

