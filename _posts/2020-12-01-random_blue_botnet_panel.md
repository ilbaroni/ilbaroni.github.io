---
layout: post
title:  "Looking into a random blue botnet panel"
date:   2020-12-01
categories: reversing
---

##  Panel Overview

When we visit the page we are presented with a directory listing:

![image-20201129192311489](/assets/images/random_blue_botnet/image-20201129192311489.png)

We can see a zip with the panel, some samples, some php files and a txt file. There are two panels in the server and If we go to the `newbot` directory or to `BOTNET_HOST/newbot/` we are presented with the same panel login page:

![image-20201129192548260](/assets/images/random_blue_botnet/image-20201129192548260.png)

I found that this was a blue botnet panel after looking into one of the samples. The next section of this blog post will be about the blue bot sample.

Comparing the source code of this panel with a leaked version of the blue botnet original panel we can see that only the titles and labels were changed:

![image-20201129192745455](/assets/images/random_blue_botnet/image-20201129192745455.png)

The languange of the new titles and labels is Vietnamese, thus this can be a good lead on the origin of the actor behind the panels.

The panel is not very sophisticated as it doesn't even use a sql database in the backend, it's all based on files present in the root of the panel.

The bots are registered in a file named `visitors.txt`:

![image-20201129193328583](/assets/images/random_blue_botnet/image-20201129193328583.png)

Only the first panel `hxxp://158.247.208.239/newbot/visitors.txt` have bots registered:

![image-20201129193635904](/assets/images/random_blue_botnet/image-20201129193635904.png)

The bots are registered based on who visit the `botlogger.php` file:

![image-20201129194934849](/assets/images/random_blue_botnet/image-20201129194934849.png)

The panel will register everyone who visits that page as a bot and the registration is simply based on the remote IP and the timestamp. Based on this, we cannot be 100% sure if every IP address registered on this panel is a bot, since any crawler, scanner, sandbox or curious person who visits that php file will get registered as a bot :(

This is a DoS botnet and the panel maintains a list of targets written to the following files:

```
target
target.ip
target.method
target.port
```

The `target` file as it's basically a formatted string based on the `target.ip`, `target.method` and `target.port` files:

![image-20201129194213108](/assets/images/random_blue_botnet/image-20201129194213108.png)

**Inside the panel**

The panel if really simple as it's only possible to define new targets and see the registered bots.

Define new targets:

![image-20201201132330068](/assets/images/random_blue_botnet/image-20201201132330068.png)

Bots:

![image-20201201132350165](/assets/images/random_blue_botnet/image-20201201132350165.png)

## Analysis of a blue bot sample

The blue bot sample found on the open directory is a .net binary:

```
vlauto.exe: PE32 executable (GUI) Intel 80386 Mono/.Net assembly, for MS Windows
```

After opening it in a PE viewer an interesting pdb can be found:

```
c:\Users\huggye\Documents\Visual Studio 2013\Projects\Blue Botnet\Blue Botnet\obj\Debug\file.pdb
```

This pdb is similar to the pdb of the blue bot builder that is published on the Internet:

```
c:\Users\huggye\Documents\Visual Studio 2013\Projects\Blue Botnet Bot Builder\Blue Botnet Bot Builder\obj\Debug\Blue Botnet Bot Builder.pdb
```

The structure of this .net PE file already gives a good overview of what this samples does:

![bluebot](/assets/images/random_blue_botnet/Screenshot_2020-11-28_17-47-34.png)

This bot starts by parsing the hardcoded c2 config and sets persistence by adding itself to the startup programs or by adding a run key in the windows registry:

![image-20201130234456922](/assets/images/random_blue_botnet/image-20201130234456922.png)

Then it parses the following two lists from the webroot of the c2 server, proxy and blog, respectively:

![image-20201130234849891](/assets/images/random_blue_botnet/image-20201130234849891.png)

Next, it parses the targets by reading the target files from the c2 webroot:

![image-20201130235142694](/assets/images/random_blue_botnet/image-20201130235142694.png)

Finally, it launches an attack to the target and it creates two threads, one for updating the targets and other to update the lists:

![image-20201130235316085](/assets/images/random_blue_botnet/image-20201130235316085.png)

Here's a yara rule to detect a blue bot sample:

```yaml
rule malware_ddos_bluebot
{
    meta:
        author = "_j"
        date = "2020-11"

    strings:
        $pdb_bot = "c:\\Users\\huggye\\Documents\\Visual Studio 2013\\Projects\\Blue Botnet\\Blue Botnet\\obj\\Debug\\file.pdb" wide ascii
        $pdb_builder = "c:\\Users\\huggye\\Documents\\Visual Studio 2013\\Projects\\Blue Botnet Bot Builder\\Blue Botnet Bot Builder\\obj\\Debug\\Blue Botnet Bot Builder.pdb" wide ascii

        $str1 = "DoSAttack" wide ascii
        $str2 = "Blue_Botnet" wide ascii
        $str3 = "botlogger.php" wide ascii

        $persistence1 = "C:\\ProgramData\\Microsoft\\Windows\\Start Menu\\Programs\\Startup\\drvhandler.exe" wide ascii
        $persistence2 = "C:\\ProgramData\\Microsoft\\Windows\\Menu Start\\Programmi\\Esecuzione Automatica\\drvhandler.exe" wide ascii

    condition:
        uint16(0) == 0x5a4d
		and (any of ($pdb*))
		or (all of ($str*))
		or (all of ($persistence*) and 1 of ($str*))
}
```

## Hunting more samples

| Domain    | jx2-bavuong.com                     |
| --------- | ----------------------------------- |
| IP        | 158.247.208.239                     |
| Registrar | GMO Internet, Inc. d/b/a Onamae.com |

On VirusTotal it was possible to find more malware samples related to this domain:

![image-20201129213528274](/assets/images/random_blue_botnet/image-20201129213528274.png)

From these samples, only two are the blue bot payload:

![image-20201129215819455](/assets/images/random_blue_botnet/image-20201129215819455.png)

The other samples are mostly downloaders and after running them the [VTI](https://www.virustotal.com/graph/embed/g1b8b3022ef6b4fc8b06361dbb6de4e5672f0b47d16f34bf3b9c4650102d3f6a3) to extract new indicators, graph escalated quickly:

![image-20201130224902607](/assets/images/random_blue_botnet/image-20201130224902607.png)

The following domains are related with `jx2-bavuong.com`:

```
volamthanthoai.net
up2.vlhoiucvn.com
volam2.club
```

## To wrap-up

I decided to take a look at this panel because I never saw it before and it was a good opportunity to do some hunting and try to find more context about the panel. It seems that the actor behind this botnet speaks Vietnamese but I'm not sure if this botnet is still alive since the way that the panel registers bots is not reliable...
