---
layout: post
title:  "Windows Component Object Model"
date:   2021-01-16
categories: reversing
---

Windows COM is an interesting area to explore and I decided to put this blog post together as a method of "self-study" to understand it a bit better. Since I am mostly interested in malware reversing and offensive tradecraft it is nice to have at least a general idea of how it works and how it can be abused.

In the first section I try to explain briefly what is COM, in the second section I show an example of a malware sample that uses a COM object to connect to an URL, the third section is about a technique known as COM hijacking and the final section is about using DCOM (Distributed Component Object Model) to achieve remote code execution.

# COM

COM refers to "Component Object Model" and it is a mechanism used by the Windows operating system to allow different applications to use the components of each other without knowing their internals. This way, two different applications, written in two different languages are able to communicate with each other.

COM is implemented as a client/server framework. The clients are the applications that are using the COM objects and the servers are the applications that are exposing their functionality through COM (the COM objects themselves).

COM defines interfaces and applications that want to expose some of their functionalities through COM must implement them accordingly.  A COM object is defined by a CLSID (Class ID) which is nothing more than a GUID (Global Unique Identifier).

Each object exposes functionality by implementing one or more interfaces, which are defined via IIDs (Interface IDs) that are also GUIDs.

The classes are defined in the registry under the **HKEY_CLASSES_ROOT\CLSID** and the interfaces are under **HKEY_CLASSES_ROOT\Interface**.

Classes:

![classes](/assets/images/windows_com/classes.png)

Interfaces:

![interfaces](/assets/images/windows_com/interfaces.png)

(There is also registration free COM that uses XML files to register COM objects instead of the windows registry.)

If you expand a random CLSID key you will see something similar to the following:

![inprocserver](/assets/images/windows_com/inprocserver.png)

You can see a key named InprocServer32 under the CLSID. This means that this COM object is implemented by a DLL.

![inprocserver2](/assets/images/windows_com/inprocserver2.png)

You may also find some CLSIDs that have a sub-key named LocalServer32 instead of InprocServer32, which means that the COM object is implemented by an EXE file.

In the values inside of InprocServer32 key you will find a path to the DLL:

![inprocserver3](/assets/images/windows_com/inprocserver3.png)

And a value named ThreadingModel:

![inprocserver4](/assets/images/windows_com/inprocserver4.png)

The value ThreadingModel can contain the following data:

| Data      | Meaning                |
| --------- | ---------------------- |
| Apartment | Single Thread          |
| Free      | Multi Thread           |
| Both      | Single or Multi Thread |
| Neutral   | Thread Neutral         |

# Malware using COM

It is common to find malware that uses COM objects to somehow masquerade their true intentions and make analysis harder.

Calls to [OleInitialize](https://docs.microsoft.com/en-us/windows/win32/api/ole2/nf-ole2-oleinitialize), [CoInitializeEx](https://docs.microsoft.com/en-us/windows/win32/api/combaseapi/nf-combaseapi-coinitializeex) are good indicators that a malware will use COM objects.

![oleinitialize](/assets/images/windows_com/oleinitialize.png)

A call to [CoCreateInstance](https://docs.microsoft.com/en-us/windows/win32/api/combaseapi/nf-combaseapi-cocreateinstance) with a CLSID and an IID being passed as arguments is used to initiate an instance of a COM object:

![cocreateinstance](/assets/images/windows_com/cocreateinstance.png)

If you take a look at the registry key and look for the CLSID you will find that it belongs to Internet Explorer:

![clsid](/assets/images/windows_com/clsid.png)

If you do the same but for the IID you will see that it refers to the IWebBrowser2 interface:

![iid](/assets/images/windows_com/iid.png)

This means that this malware is requesting access to a COM object (from Internet Explorer) that implements the IWebBrowser2 interface.

Later the malware calls the IWebBrowser2.Navigate() function: 

![navigate](/assets/images/windows_com/navigate.png)

If you check the MSDN documentation for the [IWebBrowser2](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/platform-apis/aa752127(v=vs.85)) interface, you will see the full list of methods provided by the interface and the meaning of the Navigate method:

![navigatemsdn](/assets/images/windows_com/navigatemsdn.png)

As seen, in this example the Navigate function allows the malware to use Internet Explorer to access an URL.

# COM Hijacking

When it comes to abuse COM, there is a technique known as COM Hijacking. The goal of this method is to replace the path inside the registry entry so that it points to a different DLL, one that is controlled by yourself. This can be used for purposes such as persistence and even privilege escalation.

However, this technique can be dangerous depending on the COM object that you try to hijack, because this method can break the functionality of an application.

One thing that is important to know in order to understand this technique is that COM objects are defined in two locations inside the registry key. 

| Description              | Registry Location           |
| ------------------------ | --------------------------- |
| Machine wide COM objects | HKLM\SOFTWARE\CLASSES\CLSID |
| User COM objects         | HKCU\SOFTWARE\CLASSES\CLSID |

These to locations are then merged and build the already known location: **HKEY_CLASSES_ROOT\CLSID**.

Why does this matter? Well, a regular user **can define** COM objects on the HKCU and even **duplicate** the ones in HKLM. This is interesting because when it comes to execution the HKCU definitions **take precedence** over HKLM, thus giving the opportunity to hijack what will be executed.

Like I mentioned before, COM hijacking is dangerous as it can break functionality of applications... So a common practice is to look for applications trying to access COM objects that **don't exist** and hijack those.

![notfound](/assets/images/windows_com/notfound.png)

Also you better choose one CLSID that is not accessed to often, otherwise your payload will execute to many times...

For this example I created a simple DLL that when loaded into a process memory creates a message box with some text.

![dll](/assets/images/windows_com/dll.png)

Now all you need to do is to target a CLSID and define it in the registry. The following PowerShell commands registers the targeted COM object inside the HKCU:

```
New-Item -Path "HKCU:Software\Classes\CLSID" -Name "{660b90c8-73a9-4b58-8cae-355b7f55341b}"

New-Item -Path "HKCU:Software\Classes\CLSID\{660b90c8-73a9-4b58-8cae-355b7f55341b}" -Name "InprocServer32" -Value "C:\Users\IEUser\Desktop\comhijack.dll"

New-ItemProperty -Path "HKCU:Software\Classes\CLSID\{660b90c8-73a9-4b58-8cae-355b7f55341b}\InprocServer32" -Name "ThreadingModel" -Value "Both"
```

After some random time, the following message box appears:

![hijack](/assets/images/windows_com/hijack.png)

This means that the previously inexistent COM object was successfully hijacked with my custom DLL.

# DCOM - Remote Code Execution

DCOM refers to "Distributed Component Object Model" which extends Microsoft's COM and allows the communication between software components over a network through COM.

There are many applications that export some of their functionality through DCOM. To get a list of applications that use DCOM we can run the following PowerShell command:

```
Get-CimInstance Win32_DCOMApplication
```

You should see something like this:

![dcomapps](/assets/images/windows_com/dcomapps.png)

In this example I use the MMC Application Class (MMC20.Application) since it provides a way of executing remote commands on a computer. (Note: The usage of the MMC Application Class for remote execution was not discovered by myself. In [**this**](https://enigma0x3.net/2017/01/05/lateral-movement-using-the-mmc20-application-com-object/) blogpost you can find the research of enigma0x3 about this topic.)

If you open an instance of the MMC Application Class object and search for the methods under Document.ActiveView you will see a method named [**ExecuteShellCommand**](https://docs.microsoft.com/en-us/previous-versions/windows/desktop/mmc/view-executeshellcommand?redirectedfrom=MSDN):

![mmc](/assets/images/windows_com/mmc.png)

This method will allow a user with administrator privileges to execute a command through DCOM on a remote computer.

As seen below my user which is logged at DC1 is admin on WINSRV16: 

![admin](/assets/images/windows_com/admin.png)

All I need to do now is to open an instance of the MMC20.Application object while specifying the hostname of the target:

![activator](/assets/images/windows_com/activator.png)

And finally call ExecuteShellCommand method to trigger the remote code execution: 

![remoteexec](/assets/images/windows_com/remoteexec.png)

On WINSRV16 where I am logged in as the user jamess there is now running a win32calc.exe process that belongs to the user Administrator:

![tasklist](/assets/images/windows_com/tasklist.png)



![processes](/assets/images/windows_com/processes.png)

As seen, DCOM can be used for remote code execution, thus it provides another way of moving laterally through a network.

To detect this specific way of using MMC Application Class through DCOM to trigger remote code execution defenders can look for child processes of mmc.exe.

![childprocess](/assets/images/windows_com/childprocess2.png)



If you read this I hope you enjoyed it and if I made any mistake let me know :)

Cheers!

