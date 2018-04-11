---
title: "Linux x86 Shellcoding: Analysis of linux/x86/exec with GDB"
date: 2018-04-08T17:16:07+02:00
---

In this post we will analyze the `linux/x86/exec` shellcode that comes bundled with Metasploit. We opt to use GDB to observe its behavior at runtime. First, we will check out which options are needed to generate a sample with `msfvenom`.

```
root@kali:~# msfvenom -p linux/x86/exec --payload-options
Options for payload/linux/x86/exec:


       Name: Linux Execute Command
     Module: payload/linux/x86/exec
   Platform: Linux
       Arch: x86
Needs Admin: No
 Total size: 36
       Rank: Normal

Provided by:
    vlad902 <vlad902@gmail.com>

Basic options:
Name  Current Setting  Required  Description
----  ---------------  --------  -----------
CMD                    yes       The command string to execute

Description:
  Execute an arbitrary command


Advanced options for payload/linux/x86/exec:

...
```

Thus, as we could imagine beforehand, we just need to specify the command to be executed when the shellcode is run by setting the `CMD` option. We choose something like `whoami` for simplicity's sake.

We can then generate the payload with `msfvenom` by using the following command:

```
root@kali:~# msfvenom -p linux/x86/exec CMD=/bin/whoami -f c
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 47 bytes
Final size of c file: 224 bytes
unsigned char buf[] = 
"\x6a\x0b\x58\x99\x52\x66\x68\x2d\x63\x89\xe7\x68\x2f\x73\x68"
"\x00\x68\x2f\x62\x69\x6e\x89\xe3\x52\xe8\x0c\x00\x00\x00\x2f"
"\x62\x69\x6e\x2f\x77\x68\x6f\x61\x6d\x69\x00\x57\x53\x89\xe1"
"\xcd\x80";
```

We add the `-f` parameter to indicate that we are interested in shellcode output that is ready to be inserted into a C program. This way we can directly insert it in our little C wrapper, which will come handy in order to be able to run the shellcode with GDB.

```c
#include<stdio.h>
#include<string.h>

unsigned char code[] =
"\x6a\x0b\x58\x99\x52\x66\x68\x2d\x63\x89\xe7\x68\x2f\x73\x68"
"\x00\x68\x2f\x62\x69\x6e\x89\xe3\x52\xe8\x0c\x00\x00\x00\x2f"
"\x62\x69\x6e\x2f\x77\x68\x6f\x61\x6d\x69\x00\x57\x53\x89\xe1"
"\xcd\x80";

int main() {
	printf("Shellcode length: %d\n", strlen(code));
	int (*ret)() = (int(*)()) code;
	ret();
}
```

We then compile this file and check that it's working:

```shell
$ gcc -ggdb -o shellcode-wrapper-exec shellcode-wrapper-exec.c -fno-stack-protector -z execstack -m32
$ ./shellcode-wrapper-exec
Shellcode length: 15
root
```

We now open the generated executable with GDB and set a breakpoint at the start of our shellcode, which we can reference by the variable name `code`, as in the previously listed C code snippet.

```shell
$ gdb -q shellcode-wrapper-exec 
Reading symbols from shellcode-wrapper-exec...done.
(gdb) break *&code
Breakpoint 1 at 0x2040
```

Let's now let the program run so that the breakpoint is reached, and at that point we list the assembly instructions.

```shell
(gdb) r
Starting program: /home/shellcode/5-linux-x86-exec-analysis/shellcode-wrapper-exec 
Shellcode length: 15

Breakpoint 1, 0x56557040 in code ()
(gdb) disas
Dump of assembler code for function code:
=> 0x56557040 <+0>:	push   0xb
   0x56557042 <+2>:	pop    eax
   0x56557043 <+3>:	cdq    
   0x56557044 <+4>:	push   edx
   0x56557045 <+5>:	pushw  0x632d
   0x56557049 <+9>:	mov    edi,esp
   0x5655704b <+11>:	push   0x68732f
   0x56557050 <+16>:	push   0x6e69622f
   0x56557055 <+21>:	mov    ebx,esp
   0x56557057 <+23>:	push   edx
   0x56557058 <+24>:	call   0x56557069 <code+41>
   0x5655705d <+29>:	das    
   0x5655705e <+30>:	bound  ebp,QWORD PTR [ecx+0x6e]
   0x56557061 <+33>:	das    
   0x56557062 <+34>:	ja     0x565570cc
   0x56557064 <+36>:	outs   dx,DWORD PTR ds:[esi]
   0x56557065 <+37>:	popa   
   0x56557066 <+38>:	ins    DWORD PTR es:[edi],dx
   0x56557067 <+39>:	imul   eax,DWORD PTR [eax],0xe1895357
   0x5655706d <+45>:	int    0x80
   0x5655706f <+47>:	add    BYTE PTR [eax],al
End of assembler dump.
```

Let's focus on the first bunch of instructions:

```x86asm
push   0xb
pop    eax
cdq    
push   edx
pushw  0x632d
mov    edi,esp
```

The first two instructions put `0xb` at `EAX` in a shorter way than performing the typical `mov` instruction. We will come back to the `0xb` value later, just keep it in mind! The `cdq` instruction might look strange on a first look. We find some more information in the *Intel® 64 and IA-32 Architectures Software Developer’s Manual*.

> The CDQ instruction copies the sign (bit 31) of the value in the EAX register into every bit position in the EDX register

Therefore, using `cdq` in this case is a nice one-byte solution to zeroing out the `EDX` register. These zeroes are next put into the stack via `push edx`. The following `pushw` instruction puts a word (16 bytes) in the stack, and then the `mov` instruction is used to save the stack pointer into `EDI`.

Let's examine the stack at this point by placing a new breakpoint right after this last instruction and letting the program execution continue.

```
(gdb) break *0x56557049
Breakpoint 2 at 0x56557049
(gdb) c
Continuing.

Breakpoint 2, 0x56557049 in code ()
(gdb) x/8x $esp
0xffffd616:	0x2d	0x63	0x00	0x00	0x00	0x00	0x9d	0x55
```

We can observe the `0x632d` word at the top of the stack, followed by the four `0x00` bytes that were pushed from `EDX`. What is achieved with this? Let's try printing those characters as a string:

```
(gdb) x/s $esp
0xffffd616:	"-c"
```

Interesting! So we now have the address that points to the string `-c` in `EDI`. We head back to the assembly code.

```x86asm
push   0x68732f
push   0x6e69622f
mov    ebx,esp
```

These instructions put some characters in the stack and then save the corresponding pointer into `EBX`. Let's examine those in the same way.

```
(gdb) break *0x56557055
Breakpoint 3 at 0x56557055
(gdb) c
Continuing.

Breakpoint 3, 0x56557055 in code ()
(gdb) x/s $esp
0xffffd60e:	"/bin/sh"
```

That starts making sense, both `/bin/sh` and `-c` are now already accessible from `EBX` and `EDI` respectively.

```x86asm
push   edx
call   0x56557069 <code+41>
```

These instructions first push some zero bytes into the stack (remember that `EDX` was zeroed-out at the very beginning), and then the execution is moved to another address via the `call` instruction. This instruction has one particularity that will most probably be useful for the following steps: the return address is pushed to the stack so that the execution can be resumed once the called procedure is finished (using the `ret` instruction).

Therefore, since the `call` instruction is at address `0x56557058`, `0x56557058+8` will be pushed to the stack. As we could previously see, the instructions shown by GDB from that address do not make much sense. What do we actually have there?

```
(gdb) x/s 0x5655705d
0x5655705d <code+29>:	"/bin/whoami"
```

Exactly, that is our command! Let's now follow the execution from the destination address of the `call` instruction to find out how this is used.

```
(gdb) break *0x56557069
Breakpoint 4 at 0x56557069
(gdb) c
Continuing.

Breakpoint 4, 0x56557069 in code ()
(gdb) x/8i $eip
=> 0x56557069 <code+41>:	push   edi
   0x5655706a <code+42>:	push   ebx
   0x5655706b <code+43>:	mov    ecx,esp
   0x5655706d <code+45>:	int    0x80
   0x5655706f <code+47>:	add    BYTE PTR [eax],al
   0x56557071:	add    BYTE PTR [eax],al
   0x56557073:	add    BYTE PTR [eax],al
   0x56557075:	add    BYTE PTR [eax],al
```

Having found an `int 0x80` instruction, we can say that there is a system call in there, let's extract the assembly code we are interested in:

```
push   edi
push   ebx
mov    ecx,esp
int    0x80
```

This is setting up some registers for the system call and then performing the call via `int 0x80`. Which call is this? Remember the `0xb` value in `EAX`? That register will indicate which specific syscall is being called, and in this case this value is for syscall `execve`.

By checking the Linux manpages (online at http://man7.org/linux/man-pages/man2/execve.2.html), we see that execve takes three arguments: the first one is the pointer to the program to be executed, the second is a pointer to an array with its arguments, and the third one defines environment variables to be used by the program during its execution.

These arguments are defined in assembly by using the `EBX`, `ECX` and `EDX`, respectively.

Let's check the current status of these registers:

- `EBX`: contains `/bin/sh`, the program to be executed by `execve`.
- `ECX`: check the `mov ecx, esp` instruction. At this point the stack contains the address to `/bin/sh`, `-c`, and `/bin/whoami`, followed by null bytes. By placing the stack pointer into `ECX`, this is effectively storing the address to a three-element array which defines the arguments for our program (note that the first element is the name of the program itself, again).
- `EDX`: containx 0x0, and this is fine since we are not interested in environment variables.

Therefore, the shellcode execution results will be equivalent to the ouput of the `/bin/sh -c /bin/whoami` command.

---

*This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:*

http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/

*Student ID: SLAE-964*
