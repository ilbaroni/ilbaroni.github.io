---
layout: post
title:  "Unpacking RAGNARLOCKER via emulation"
date:   2021-04-15
categories: reversing
---
# Introduction

Packers are a common way for adversaries to protect their payloads, avoid detections and make reverse engineering a bit harder. There are several types of packers, some use anti-analysis techniques and others are a bit more simple to reverse engineer.

Manual analysis of packers helps in tasks such as, making signatures to track more malware using that specific packer and developing tools that allow us to unpack malware automatically. To do that, reverse engineers need to understand how the packer is working, thus this can be a time-consuming task.

I like automating tasks whenever is possible, and I've always wondered about the automation of unpacking.

Recently I've started to read more about emulation and came across the [Qiling framework](https://github.com/qilingframework/qiling). The idea of unpacking malware via emulation seemed very interesting so I started exploring the capabilities of Qiling for this specific use case.

Here I'll try to explain my approach to unpack RAGNARLOCKER with Qiling.

# Reverse engineering the packer

To extract the payload via emulation, I needed to understand a bit of how the 3 stages of this packer work. 

There is no need to understand all the details of the algorithms used by this packer because in the end I'll just let the malware run and unpack the payload itself. 

Instead, I need to understand the flow of the unpacking routines and try to identify a stage where the payload is unpacked in memory so that I could dump it.

### First stage

In this first stage, the packer executes several worthless instructions, functions, and loops to slow down the analysis. It also uses some anti-emulation techniques, possibly to avoid being executed by emulators like Qiling. Below it's possible to see some examples.

**Worthless function:**

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415105305206.png)

**Worthless loop and instructions:**

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415105556495.png)

**Giant loop to slow down the execution (this is costly to an emulator, specially written in python):**

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415104926358.png)

**Anti emulating via GetLastError API:**

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415110306334.png)

Another anti-emulation method used, was the usage of **GetLastError** Windows API, which is used to check the last error code of the calling thread.

The packer calls **SetWindowContextHelpId** with an invalid handle and checks if the last error equals **ERROR_INVALID_WINDOW_HANDLE** that corresponds to the value **0x578**.

### Second stage

In this stage, the packer simply allocates a new memory region, decrypts a shellcode, copies the shellcode to the newly allocated memory, and transfers execution to it.

**Allocating a new memory region:**

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415111539604.png)

**Decrypting the shellcode:**

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415111627124.png)

**Transferring execution to the shellcode:**

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415111727379.png)

As seen, a handle to **KERNEL32.dll** is passed to the shellcode. 

This handle is later used to resolve the needed APIs.

### Third stage - final shellcode

In this last stage, the shellcode decrypts the payload and loads it using a self replacement technique.

**Resolving the needed APIs:**

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415113116136.png)

Summary of the APIs used by the shellcode:

```
VirtualAlloc
GetProcAddress
VirtualProtect
LoadLibraryA
VirtualFree
VirtualFree
VirtualQuery
TerminateThread
```

**Allocating two memory regions:**

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415122917463.png)

**Copying the encrypted payload to the first memory region and decrypting it:**

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415123052129.png)

**Copying the decrypted payload from the first memory region to the second memory region, and calling VirtualFree:**

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415123258348.png)

The perfect time to dump the unpacked RAGNARLOCKER payload is when the shellcode calls **VirtualFree**.

As seen below, at the moment the shellcode calls **VirtualFree** the second memory region allocated by the shellcode contains a PE file (the unpacked payload).

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415174103413.png)

Based on the analysis of the packer the strategy to unpack the payload with Qiling is the following:

| Strategy to unpack                                           |
| ------------------------------------------------------------ |
| Track all the memory regions that are allocated. To do that I used hooks in VirtualAlloc and  VirtualAllocEx. |
| When VirtualFree is called, dump the last allocated memory region. |

This seems simple enough, but I also needed to overcome the anti-emulation tricks and Qiling limitations:

| Strategy to overcome Anti-Emulation tricks and Qiling limitations |
| ------------------------------------------------------------ |
| Bypass GetLastError anti-emulation trick.                    |
| Patch the large anti-emulation loop.                         |
| Implement any missing windows apis. (Qiling limitation)      |

# Qiling Emulation Framework

Qiling is a high-level framework that tries to emulate both the CPU and the OS. To unpack malware samples, we just have to be aware of the anti-emulation tricks (that are often used by packers) and implement any missing APIs.

Description from the official website:

> Qiling is designed as a higher level framework, that leverages Unicorn to emulate CPU instructions, but Qiling understands OS: it has executable format loaders (for PE, MachO & ELF at the moment), dynamic linkers (so we can load & relocate shared libraries), syscall & IO handlers. For this reason, Qiling can run excutable binaries that normally runs in native OS

The advantage of using a framework like this to unpack malware is that the need to fully understand the unpacking algorithm is eliminated. Also, the unpacker script may survive updates in the algorithm of the packer.

### Bypass GetLastError anti-emulation trick

As seen before, this packer uses the **GetLastError** to check if the last error code was **0x578** after calling **SetWindowContextHelpId**.

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415110306334.png)

Fortunately, in Qiling it's possible to set specific error codes in the OS that it's being emulated. The hook implementation for this API is the following:

```python
@winsdkapi(cc=STDCALL, dllname="user32_dll")
def hook_SetWindowContextHelpId(ql, address, params):
    ERROR_INVALID_WINDOW_HANDLE = 0x578 
    ql.os.last_error = ERROR_INVALID_WINDOW_HANDLE  
    return False
```

Additionally, **GetWindowContextHelpId** seems to be called with the **SetWindowContextHelpId**. Since this API is not implemented in Qiling I needed to implement it and make it set the correct error code. 

```python
@winsdkapi(cc=STDCALL, dllname="user32_dll") 
def hook_GetWindowContextHelpId(ql, address, params): 
    ERROR_INVALID_WINDOW_HANDLE = 0x578 
    ql.os.last_error = ERROR_INVALID_WINDOW_HANDLE  
    return False
```

### Patching the large anti-emulation loop

As seen before, the packer uses a large "for" loop possibly to avoid being executed under emulators like Qiling.

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415104926358.png)

Fortunately, Qiling can search for specific byte patterns in memory and patch them.

The bytes that I choose to patch were the ones that make the instruction `cmp     [ebp+var_1B8], 1E8480h`:

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415132658581.png)

The idea was to change it to `cmp     [ebp+var_1B8], 0`. 

This way the code does not enter the "for" loop.

(**Note:** Another approach could be to turn the conditional jump that comes after in an unconditional jump)

Patch function:

```python
# Patch specific byte patterns
def patch_bytes(ql):
    patches = []

    # Patch needed to avoid the anti-emulation loop
    # original bytes -> 81 BD 48 FE FF FF 80 84 1E 00 = cmp dword ptr ss:[ebp-1B8],1E8480
    # patched bytes ->  83 BD 48 FE FF FF 00 90 90 90 = cmp dword ptr ss:[ebp-1B8],0
    patches.append({'original': b'\x81\xBD\x48\xFE\xFF\xFF\x80\x84\x1E\x00', 'patch': b'\x83\xBD\x48\xFE\xFF\xFF\x00\x90\x90\x90'})
    
    for patch in patches:
        addr = ql.mem.search(patch['original'])
        if addr:
            ql.log.warning('found target patch bytes at addr: {}'.format(hex(addr[0])))
            try:
                ql.patch(addr[0], patch['patch'])
                ql.log.info('patch sucessfully applied')
                return 
            except Exception as err:
                ql.log.error('unable to apply the patch. error: {}'.format(str(e)))
        else:
            ql.log.warning('target patch bytes not found')
```

### Overcoming Qiling limitations

Some Windows APIs are not yet implemented in Qiling, thus I needed to implement them. 

In Qiling implementing an API, it's the same as hooking an API. 

```python
'''
Not implemented in Qiling
'''
@winsdkapi(cc=STDCALL, dllname="user32_dll")
def hook_CharUpperW(ql, address, params):
    return params["lpsz"]

'''
Not implemented in Qiling
'''
@winsdkapi(cc=STDCALL, dllname="user32_dll")
def hook_CharUpperBuffW(ql, address, params):
    return 100

'''
This api is giving troubles to Qiling in the way the malware passes arguments.
So let's hook it and making it returning null since the packer does not use the return value for nothing.
'''
@winsdkapi(cc=STDCALL, dllname="kernel32_dll")
def hook_CreateEventA(ql, address, params):
    return 0

'''
Qiling is retuning 0x0 by default and the packer stub only continues if this value is different from 0.
So let's just hook it and make it return a value different then 0
'''
@ winsdkapi(cc=STDCALL, dllname="kernel32_dll") 
def hook_VirtualQuery(ql, address, params): 
    return params['dwLength']
```

### Defining the needed hooks

With the anti-emulation loop patched and the Qiling limitations been taken care of, it was a matter of hooking the rest of the needed functions to keep up with the unpacking strategy.

```python
@winsdkapi(cc=STDCALL, dllname="kernel32_dll")
def hook_VirtualFree(ql, address, params):
    global mem_regions
    lpAddress = params['lpAddress']

    ql.log.warning('VirtualFree called. lpAddress = {}'.format(hex(lpAddress)))
    ql.log.warning('time to dump last allocated memory...')
    unpacked_mem_region = mem_regions[-1]
    dump_memory_region(ql, unpacked_mem_region['start'], unpacked_mem_region['size'])
    ql.os.heap.free(lpAddress)
    exit()
    return 1

@winsdkapi(cc=STDCALL, dllname="kernel32_dll")
def hook_VirtualProtect(ql, address, params):
    return 1

@winsdkapi(cc=STDCALL, dllname="kernel32_dll")
def hook_VirtualAllocEx(ql, address, params):
    global mem_regions

    dw_size = params["dwSize"]
    addr = ql.os.heap.alloc(dw_size) # allocate memory in heap
    
    ql.log.warning('VirtualAllocEx hook allocated a new memory on the heap at -> {} with size -> {} bytes'.format(hex(addr), hex(dw_size)))

    mem_reg = {"start": addr, "size": dw_size}
    mem_regions.append(mem_reg)
    return addr

@winsdkapi(cc=STDCALL, dllname="kernel32_dll")
def hook_VirtualAlloc(ql, address, params):
    global mem_regions
    
    dw_size = params["dwSize"]
    addr = ql.os.heap.alloc(dw_size) # allocate memory in heap
    
    ql.log.warning('VirtualAlloc hook allocated a new memory on the heap at -> {} with size -> {} bytes'.format(hex(addr), hex(dw_size)))
    
    mem_reg = {"start": addr, "size": dw_size}
    mem_regions.append(mem_reg)
    return addr
```

Things to notice in the above definitions:

- Both the **VirtualAlloc** and **VirtualAllocEx** hooks save the memory regions being allocated to a global variable.
- The **VirtualFree** hook calls a function to dump the last memory region that was saved in the global variable. 

The function that dumps the memory region:

```python
def dump_memory_region(ql, address, size):
    ql.log.warning('dumping memory section at: {}'.format(hex(address)))
    ql.log.warning('size: {}'.format(hex(size)))
    try:
        exec_mem = ql.mem.read(address, size)
        with open('{}.bin'.format(hex(address)), "wb") as f:
            f.write(exec_mem)
    except Exception as e:
        ql.log.error(str(e))
```

# Unpacking RAGNARLOCKER

Script output:

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415153300032.png)

As seen, the script was able to dump the RAGNARLOCKER payload:

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415153522554.png)

The file that was unpacked should open with no problems in a PE viewer like [PE-BEAR](https://github.com/hasherezade/pe-bear-releases/releases):

![ ](/assets/images/unpacking_ragnarlocker_via_emulation/image-20210415153657912.png)

# Conclusion

I can see the potential in using a framework like Qiling to automate certain tasks in reverse engineering, and I'll keep exploring emulation and other use cases for this great framework.



**Full Script:**

You can find the full script in [**this gist**](https://gist.github.com/jnzer0/3114db9dfc84c62b1ba752e465ea8c15).



**Sample used in this proof of concept:**

```
68eb2d2d7866775d6bf106a914281491d23769a9eda88fc078328150b8432bb3
```

**References:**

- [Automated malware unpacking with binary emulation](https://lopqto.me/posts/automated-malware-unpacking)
- [Using Qiling Framework to Unpack TA505 packed samples](https://www.blueliv.com/cyber-security-and-cyber-threat-intelligence-blog-blueliv/using-qiling-framework-to-unpack-ta505-packed-samples/)
- [Qiling Framework Documentation](https://docs.qiling.io/en/latest/)