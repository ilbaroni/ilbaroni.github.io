---
layout: post
title:  "How to get a DLL base address from the Process Environment Block (PEB)"
date:   2021-08-03
categories: posts
---



Very often, we see malware families dynamically resolving the needed Windows APIs. One of the first steps that many malware families take is to get the base address of either `KERNEL32.DLL` or `NTDLL.DLL` from the Process Environment Block (`PEB`).



The Process Environment Block (`PEB`) is a structure that contains information about the process itself and the loaded modules.



The following is the official documented `PEB` structure ([MSDN](https://docs.microsoft.com/en-us/windows/win32/api/winternl/ns-winternl-peb)):

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



> **Note**: The reserved names are there because Microsoft officially does not document all fields from the PEB structure.



One way of getting the pointer to the `PEB` is, for example, by accessing a specific offset in the FS or GS register (depends on the architecture). 

- FS[0x30] - 32 bit Windows
- GS[0x60] - 64 bit Windows



C code snippet for getting a valid pointer to `PEB`:

```c
#ifndef _WIN64
	    PEB* peb = (PEB*) __readfsdword(0x30);
#else
	    PEB* peb = (PEB*) __readgsqword(0x60);
#endif
```



After getting access to the `PEB` pointer, it's possible to access the `Ldr` field which, points to a `PEB_LDR_DATA` structure.

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



This structure contains the `InLoadOrderModuleList` field that points to a `LIST_ENTRY`.



As seen below, the `LIST_ENTRY` structure is a double-linked list that contains two pointers. One pointer is for the next `LIST_ENTRY`, while the other is for the previous one.

```
0:000> dt ntdll!_LIST_ENTRY
   +0x000 Flink            : Ptr64 _LIST_ENTRY
   +0x008 Blink            : Ptr64 _LIST_ENTRY
```



The **trick** is that in these `LIST_ENTRY` pointers there is another structure (a bigger one), which contains more data. This structure is named `LDR_DATA_TABLE_ENTRY` and below we can see its definition:

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



By casting the `LIST_ENTRY` pointer as a pointer to the `LDR_DATA_TABLE_ENTRY` structure, we get access to some fields such as the `BaseDllName` and the `DllBase`:

![ ](/assets/images/get_dll_base_from_peb/image-20210730234543419.png)



> **Note**: The BaseDllName is in UNICODE.



A snippet of an implementation of this technique as seen by the hex-rays decompiler:

![ ](/assets/images/get_dll_base_from_peb/image-20210731010014781.png)


