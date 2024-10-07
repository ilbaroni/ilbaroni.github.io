---
layout: post
title:  "How to get a DLL base address from the Process Environment Block (PEB)"
image_alt: {
    url: "/assets/images/get_dll_base_from_peb_preview.png",
    source: ""
}
excerpt: "In this post I show how to get dll base addresses using the Process Environment Block (PEB) structure."
date:    2021-08-03 00:00:00 -0000
categories: []
---

We often see malware families resolving Windows APIs dynamically and abusing the built-in Windows OS structures to so. The Process Environment Block (PEB) is a structure that contains information about the process itself and the loaded modules.

PEB structure according to [MSDN](https://docs.microsoft.com/en-us/windows/win32/api/winternl/ns-winternl-peb):

```c
typedef struct _PEB {
    BYTE Reserved1[2];
    BYTE BeingDebugged;
    BYTE Reserved2[1];
    PVOID Reserved3[2];
    PPEB_LDR_DATA Ldr;
    PRTL_USER_PROCESS_PARAMETERS ProcessParameters;
    PVOID Reserved4[3];
    PVOID AtlThunkSListPtr;
    PVOID Reserved5;
    ULONG Reserved6;
    PVOID Reserved7;
    ULONG Reserved8;
    ULONG AtlThunkSListPtr32;
    PVOID Reserved9[45];
    BYTE Reserved10[96];
    PPS_POST_PROCESS_INIT_ROUTINE PostProcessInitRoutine;
    BYTE Reserved11[128];
    PVOID Reserved12[1];
    ULONG SessionId;
} PEB, *PPEB;
```

One way of getting the pointer to PEB is, for example, by accessing a specific offset using the FS or GS register depending on the architecture. 
- FS[0x30] - FS offset to PEB for 32 bit Windows
- GS[0x60] - GS offset to PEB for 64 bit Windows

Here's a C code snippet to get a pointer to PEB:
```c
#ifndef _WIN64
	    PEB* peb = (PEB*) __readfsdword(0x30);
#else
	    PEB* peb = (PEB*) __readgsqword(0x60);
#endif
```

Having the PEB pointer it is now possible to access the PEB.Ldr field which points to a PEB_LDR_DATA structure.

```
0:000> dt ntdll!_PEB_LDR_DATA
   +0x000 Length           : Uint4B
   +0x004 Initialized      : UChar
   +0x008 SsHandle         : Ptr64 Void
   +0x010 InLoadOrderModuleList : _LIST_ENTRY
   +0x020 InMemoryOrderModuleList : _LIST_ENTRY
   +0x030 InInitializationOrderModuleList : _LIST_ENTRY
   +0x040 EntryInProgress  : Ptr64 Void
   +0x048 ShutdownInProgress : UChar
   +0x050 ShutdownThreadId : Ptr64 Void
```

The PEB_LDR_DATA.InLoadOrderModuleList field points to a LIST_ENTRY structure.

As seen below, a LIST_ENTRY structure is a double-linked list that contains two pointers. One pointer is for the next LIST_ENTRY structure, while the other is for the previous one.

```
0:000> dt ntdll!_LIST_ENTRY
   +0x000 Flink            : Ptr64 _LIST_ENTRY
   +0x008 Blink            : Ptr64 _LIST_ENTRY
```

The **trick** is that these LIST_ENTRY pointers are part of a another structure (a bigger one), which contains more data. This structure is named LDR_DATA_TABLE_ENTRY and below we can see its definition:
```
0:000> dt ntdll!_LDR_DATA_TABLE_ENTRY
ntdll!_LDR_DATA_TABLE_ENTRY
   +0x000 InLoadOrderLinks : _LIST_ENTRY
   +0x010 InMemoryOrderLinks : _LIST_ENTRY
   +0x020 InInitializationOrderLinks : _LIST_ENTRY
   +0x030 DllBase          : Ptr64 Void
   +0x038 EntryPoint       : Ptr64 Void
   +0x040 SizeOfImage      : Uint4B
   +0x048 FullDllName      : _UNICODE_STRING
   +0x058 BaseDllName      : _UNICODE_STRING
   +0x068 Flags            : Uint4B
   +0x06c LoadCount        : Uint2B
   +0x06e TlsIndex         : Uint2B
   +0x070 HashLinks        : _LIST_ENTRY
   +0x070 SectionPointer   : Ptr64 Void
   +0x078 CheckSum         : Uint4B
   +0x080 TimeDateStamp    : Uint4B
   +0x080 LoadedImports    : Ptr64 Void
   +0x088 EntryPointActivationContext : Ptr64 _ACTIVATION_CONTEXT
   +0x090 PatchInformation : Ptr64 Void
   +0x098 ForwarderLinks   : _LIST_ENTRY
   +0x0a8 ServiceTagLinks  : _LIST_ENTRY
   +0x0b8 StaticLinks      : _LIST_ENTRY
   +0x0c8 ContextInformation : Ptr64 Void
   +0x0d0 OriginalBase     : Uint8B
   +0x0d8 LoadTime         : _LARGE_INTEGER

```

So if we cast a LIST_ENTRY pointer obtained from PEB_LDR_DATA.InLoadOrderModuleList as a LDR_DATA_TABLE_ENTRY pointer, we get access to more fields such as the BaseDllName and the DllBase:

![](/assets/images/get_dll_base_from_peb1.png)

Here is a snippet of code that uses this technique to obtain the base address of a DLL:

![](/assets/images/get_dll_base_from_peb2.png)
