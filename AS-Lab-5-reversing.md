# AS Lab 5 - Reversing

Artem Abramov



## 1. Preparation

#### a. You can use any debugger/disassembler as long as it supports the given task architecture (32bit or 64bit)

I choose GDB and 



#### b. Try to stay away from decompilers, as this will help you to better understand the nature of your task based on assembly only, remember in real-world tasks you will have to deal with much larger binaries.

Ok.



#### c. IMPORTANT: please check what does a binary do before running it

I setup a docker container:

```
sudo docker run  --cap-add=SYS_PTRACE --security-opt seccomp=unconfined  --name testbin -it ubuntu /bin/bash
```

The parameters ` --cap-add=SYS_PTRACE --security-opt seccomp=unconfined ` are necessary to allow GDB and strace inside docker. (source: https://stackoverflow.com/questions/35860527/warning-error-disabling-address-space-randomization-operation-not-permitted)

And copy-pasted binaries into it:

```
sudo docker cp Lab5 testbin:/root/Lab5
```



#### d. Check some writeups about some CTF to see what you should/shouldn’t include in your report

Ok.





## 2. Theory

### a. What kind of file did you receive (which arch? 32bit or 64bit?)?
Summary for file info:

```
bin1: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=c5b1692162984ff6555feb261df6b530e5c52945, not stripped

bin2: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=ba3f21bfd29e03a056a158da7cf3ef2e7c113947, not stripped

bin3: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=ba3f21bfd29e03a056a158da7cf3ef2e7c113947, not stripped

bin4: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=774d56c6997e8d57e1db3c7db2562cd170d75395, not stripped

bin5: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=66733c5a908fd5eb4f1189875f252b561f1926e5, stripped

bin6: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 3.2.0, BuildID[sha1]=be24706e7be9ba3fae96939e094dba08023c2461, stripped
```

All of them are x86-64, that is for 64-bit architecture.

Binaries 1 to 5 are identified as shared objects i.e. libraries, and bin6 is an executable. However this is just an artifact of compilation, actually all the files are executable. (an example mechanism for this is described here: https://unix.stackexchange.com/questions/223385/why-and-how-are-some-shared-libraries-runnable-as-though-they-are-executables)



### b. What do stripped binaries mean?

It means that all debug symbols were stripped i.e. removed from the executable. The executable format of the binaries if ELF (Executable and Linkable Format) this format specifies that an elf binary can contain different segments, in particular there is a segment for debug info (this is most commonly in DWARF format). The symbols can provide very important hints when reversing (or debugging) because they normally contain names of all functions (internal and from shared libraries). For this reason and to save space release builds of software are usually stripped of all the symbols and function calls can only be recognized by call/jmp instruction.



### c. What are GOT and PLT?

source: 

- https://reverseengineering.stackexchange.com/questions/1992/what-is-plt-got
- https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html
- https://www.airs.com/blog/archives/39

This are necessary to support loading shared libraries and to support address space randomization. 

The root cause is that during compilation the compiler and linker do not know at what address the operating system will load the binaries (and they can not force the address, because if they did then libraries who would accidentally overlap will misfunction). The idea is to have another level of indirection, so to say its a dictionary (actually an offset table) where addresses of shared objects would register. To access a function or a memory location you check its address in this dictionary and then access it. The lookup itself is done as follows: when the shared library is loaded a dynamic linker checks the address and patches the binary with the correct address + offset (so actually the dictionary analogy is flawed, because that dictionary is only referenced once, during the shared library load). The address patching (vs looking up in dictionary every time) is done for speed. 

GOT stands for Global Offsets Table and handles addresses and offsets for data/variables. 

PLT stands for Procedure Linkage Table and handles addresses and offsets for functions.



### d. What are binary symbols in reverse engineering? How does it help?
Check the symbols present in `bin1`:

```
# nm bin1 
0000000000201010 B __bss_start
0000000000201010 b completed.7697
                 w __cxa_finalize@@GLIBC_2.2.5
0000000000201000 D __data_start
0000000000201000 W data_start
0000000000000660 t deregister_tm_clones
00000000000006f0 t __do_global_dtors_aux
0000000000200da8 t __do_global_dtors_aux_fini_array_entry
0000000000201008 D __dso_handle
0000000000200db0 d _DYNAMIC
0000000000201010 D _edata
0000000000201018 B _end
0000000000000854 T _fini
0000000000000730 t frame_dummy
0000000000200da0 t __frame_dummy_init_array_entry
00000000000009cc r __FRAME_END__
0000000000200fa0 d _GLOBAL_OFFSET_TABLE_
                 w __gmon_start__
0000000000000888 r __GNU_EH_FRAME_HDR
00000000000005b8 T _init
0000000000200da8 t __init_array_end
0000000000200da0 t __init_array_start
0000000000000860 R _IO_stdin_used
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
0000000000000850 T __libc_csu_fini
00000000000007e0 T __libc_csu_init
                 U __libc_start_main@@GLIBC_2.2.5
                 U localtime@@GLIBC_2.2.5
000000000000073a T main
                 U printf@@GLIBC_2.2.5
00000000000006a0 t register_tm_clones
                 U __stack_chk_fail@@GLIBC_2.4
0000000000000630 T _start
                 U time@@GLIBC_2.2.5
0000000000201010 D __TMC_END__
```



Now check the symbols present in `bin6`:

```
# nm bin6
nm: bin6: no symbols
```



Clearly reversing bin6 will be more difficult, because it is stripped, there are fewer hints.

Another interesting program to run:

```
readelf -a bin1
```

Provides maximum information about the binary such as size of functions, symbols, stack arrangement, BSS, etc.

For example there is some meta information:

```
Version symbols section '.gnu.version' contains 10 entries:
 Addr: 0000000000000452  Offset: 0x000452  Link: 5 (.dynsym)
  000:   0 (*local*)       2 (GLIBC_2.2.5)   0 (*local*)       3 (GLIBC_2.4)  
  004:   2 (GLIBC_2.2.5)   2 (GLIBC_2.2.5)   0 (*local*)       2 (GLIBC_2.2.5)
  008:   0 (*local*)       2 (GLIBC_2.2.5)

Version needs section '.gnu.version_r' contains 1 entry:
 Addr: 0x0000000000000468  Offset: 0x000468  Link: 6 (.dynstr)
  000000: Version: 1  File: libc.so.6  Cnt: 2
  0x0010:   Name: GLIBC_2.4  Flags: none  Version: 3
  0x0020:   Name: GLIBC_2.2.5  Flags: none  Version: 2

Displaying notes found in: .note.ABI-tag
  Owner                 Data size	Description
  GNU                  0x00000010	NT_GNU_ABI_TAG (ABI version tag)
    OS: Linux, ABI: 3.2.0

Displaying notes found in: .note.gnu.build-id
  Owner                 Data size	Description
  GNU                  0x00000014	NT_GNU_BUILD_ID (unique build ID bitstring)
    Build ID: c5b1692162984ff6555feb261df6b530e5c52945
```

So we can know what version of GLIBC this executable was build against, this is very useful during reversing, because we can analyze specific patterns and if we would want to find exploits we would look at how to exploit a particular glibc version which helps to narrow our search.



## 3. Reversing

####  a. Inside the ZIP file, you will have multiple binaries, Try to reverse them by recreating them using any programming language of your choice (C is more preferred)



To make my job easier I decided to try using Cutter which is a GUI on top of radare2: https://cutter.re/

They provide an  .appimage format, so running it was extremely easy:

1. Download
2. chmod +x 
3. Execute it



To understand the x86-32 bit calling conventions see (There are loads of them): https://levelup.gitconnected.com/x86-calling-conventions-a34812afe097

For x86-64 bit calling conventions see (There are basically only two of them System-V AMD64 and Microsoft): https://en.wikipedia.org/wiki/X86_calling_conventions#x86-64_calling_conventions

The first six integer or pointer arguments are passed in registers RDI, RSI, RDX, RCX, R8, R9.

Integer return values up to 64 bits in size are stored in RAX while values up to 128 bit are stored in RAX and RDX.

If the callee wishes to use registers RBX, RBP, and R12–R15, it must  restore their original values before returning control to the caller.  All other registers must be saved by the caller if it wishes to preserve their values.



### Bin1

##### Disassembly 

```
; var time_t timer @ rbp-0x18
; var tm*var_10h @ rbp-0x10
; var int64_t canary @ rbp-0x8
0x0000073a      push rbp
0x0000073b      mov rbp, rsp
0x0000073e      sub rsp, 0x20
0x00000742      mov rax, qword fs:[0x28]
0x0000074b      mov qword [canary], rax
0x0000074f      xor eax, eax
0x00000751      mov edi, 0         ; time_t *timer
0x00000756      call time          ; sym.imp.time ; time_t time(time_t *timer)
0x0000075b      mov qword [timer], rax0x00000742      mov rax, qword fs:[0x28]
0x0000074b      mov qword [canary], rax
0x0000075f      lea rax, [timer]
0x00000763      mov rdi, rax       ; const time_t *timer
0x00000766      call localtime     ; sym.imp.localtime ; tm*localtime(const time_t *timer)
0x0000076b      mov qword [var_10h], rax
0x0000076f      mov rax, qword [var_10h]
0x00000773      mov esi, dword [rax + 0x1c]
0x00000776      mov rax, qword [var_10h]
0x0000077a      mov r8d, dword [rax + 4]
0x0000077e      mov rax, qword [var_10h]
0x00000782      mov edi, dword [rax + 0x18]
0x00000785      mov rax, qword [var_10h]
0x00000789      mov ecx, dword [rax + 0xc]
0x0000078c      mov rax, qword [var_10h]
0x00000790      mov edx, dword [rax + 0x10]
0x00000793      mov rax, qword [var_10h]
0x00000797      mov eax, dword [rax + 0x14]
0x0000079a      sub rsp, 8
0x0000079e      push rsi
0x0000079f      mov r9d, r8d0x00000742      mov rax, qword fs:[0x28]
0x0000074b      mov qword [canary], rax
0x000007a2      mov r8d, edi
0x000007a5      mov esi, eax
0x000007a7      lea rdi, str.04d__02d__02d__02d:_02d:_02d ; 0x868 ; const char *format
0x000007ae      mov eax, 0
0x000007b3      call printf        ; sym.imp.printf ; int printf(const char *format)
0x000007b8      add rsp, 0x10
0x000007bc      nop
0x000007bd      mov rax, qword [canary]
0x000007c1      xor rax, qword fs:[0x28]
0x000007ca      je 0x7d1
0x000007cc      call __stack_chk_fail ; sym.imp.__stack_chk_fail ; void __stack_chk_fail(void)
0x000007d1      leave
0x000007d2      ret
0x000007d3      nop word cs:[rax + rax]
0x000007dd      nop dword [rax]
```



##### Stack smashing protection (I will not mention this for other binaries, because its the same)

Create the canary variable at top of stack (this is disasm output, so variable creation is just using the address [rbp-0x8] ):

```
; var int64_t canary @ rbp-0x8
```

Set it:

```
0x00000742      mov rax, qword fs:[0x28]
0x0000074b      mov qword [canary], rax
```

Check it at the end:

```
0x000007bd      mov rax, qword [canary]
0x000007c1      xor rax, qword fs:[0x28]
0x000007ca      je 0x7d1
0x000007cc      call __stack_chk_fail
0x000007d1      leave
```



##### Calling time() to get seconds since Unix epoch

The assembly:

```
0x0000074f      xor eax, eax
0x00000751      mov edi, 0         ; time_t *timer
0x00000756      call time          ; sym.imp.time ; time_t time(time_t *timer)
0x0000075b      mov qword [timer], rax
```

Is equivalent to:

```C
int64_t timer = time(0);
```



##### Calling localtime() to get date

The assembly:

```
0x0000075f      lea rax, [timer]
0x00000763      mov rdi, rax       ; const time_t *timer
0x00000766      call localtime     ; sym.imp.localtime ; tm*localtime(const time_t *timer)
0x0000076b      mov qword [var_10h], rax
```

Is equivalent to:

```C
struct tm* var_10h = localtime( &timer );
```



The function localtime allocates the memory internally, thus we only get a pointer to that internal struct, so there is no need to create struct on the stack. From `man localtime`:

```
	                         The return value points to a statically allocated
       string which might be overwritten by subsequent calls  to  any  of  the
       date and time functions.
```



##### Calling printf() to print them

This assembly:

```
0x0000076f      mov rax, qword [var_10h]
0x00000773      mov esi, dword [rax + 0x1c]
0x00000776      mov rax, qword [var_10h]
0x0000077a      mov r8d, dword [rax + 4]
0x0000077e      mov rax, qword [var_10h]
0x00000782      mov edi, dword [rax + 0x18]
0x00000785      mov rax, qword [var_10h]
0x00000789      mov ecx, dword [rax + 0xc]
0x0000078c      mov rax, qword [var_10h]
0x00000790      mov edx, dword [rax + 0x10]
0x00000793      mov rax, qword [var_10h]
0x00000797      mov eax, dword [rax + 0x14]
0x0000079a      sub rsp, 8
0x0000079e      push rsi
0x0000079f      mov r9d, r8d
0x000007a2      mov r8d, edi
0x000007a5      mov esi, eax
0x000007a7      lea rdi, str.04d__02d__02d__02d:_02d:_02d ; 0x868 ; const char *format
0x000007ae      mov eax, 0
0x000007b3      call printf        ; sym.imp.printf ; int printf(const char *format)
```

The argument string for printf is at address [0x868] as shown by the disassembly:

```
0x00000868          .string "%04d-%02d-%02d %02d:%02d:%02d\n" ; len=31
```



So this is the following C code, just putting the values from certain memory location into registers and calling printf. Of course the order is reversed so the format string is supplied last, but its the first argument:

```C
printf("%04d-%02d-%02d %02d:%02d:%02d\n", var_10h->tm_year, var_10h->tm_mon, var_10h->tm_mday, var_10h->tm_wday, var_10h->tm_min, var_10h->tm_yday);
```



##### Final C code

The final equivalent C code would be the following:

```C
#include <stdio.h>
#include <time.h>
#include <stdint.h>

void main() {
        int64_t timer = time( 0 );
        struct tm* var_10h = localtime( &timer );
        printf("%04d-%02d-%02d %02d:%02d:%02d\n", var_10h->tm_year, var_10h->tm_mon, var_10h->tm_mday, var_10h->tm_wday, var_10h->tm_min, var_10h->tm_yday);
}
```



### Bin2

##### Dissasembly

```
134: int main (int argc, char **argv, char **envp);
; var signed int64_t var_64h @ rbp-0x64
; var int64_t canary @ rbp-0x8
0x000006aa      push rbp
0x000006ab      mov rbp, rsp
0x000006ae      sub rsp, 0x70
0x000006b2      mov rax, qword fs:[0x28]
0x000006bb      mov qword [canary], rax
0x000006bf      xor eax, eax
0x000006c1      mov dword [var_64h], 0
0x000006c8      jmp 0x6dd
0x000006ca      mov eax, dword [var_64h]
0x000006cd      lea edx, [rax + rax]
0x000006d0      mov eax, dword [var_64h]
0x000006d3      cdqe
0x000006d5      mov dword [rbp + rax*4 - 0x60], edx
0x000006d9      add dword [var_64h], 1
0x000006dd      cmp dword [var_64h], 0x13
0x000006e1      jle 0x6ca
0x000006e3      mov dword [var_64h], 0
0x000006ea      jmp 0x70f
0x000006ec      mov eax, dword [var_64h]
0x000006ef      cdqe
0x000006f1      mov edx, dword [rbp + rax*4 - 0x60]
0x000006f5      mov eax, dword [var_64h]
0x000006f8      mov esi, eax
0x000006fa      lea rdi, str.a__d___d ; 0x7b4 ; const char *format
0x00000701      mov eax, 0
0x00000706      call printf        ; sym.imp.printf ; int printf(const char *format)
0x0000070b      add dword [var_64h], 1
0x0000070f      cmp dword [var_64h], 0x13
0x00000713      jle 0x6ec
0x00000715      mov eax, 0
0x0000071a      mov rcx, qword [canary]
0x0000071e      xor rcx, qword fs:[0x28]
0x00000727      je 0x72e
0x00000729      call __stack_chk_fail ; void __stack_chk_fail(void)
0x0000072e      leave
0x0000072f      ret
```



##### Loop number one

```
0x000006c1      mov dword [var_64h], 0
0x000006c8      jmp 0x6dd
0x000006ca      mov eax, dword [var_64h]
0x000006cd      lea edx, [rax + rax]
0x000006d0      mov eax, dword [var_64h]
0x000006d3      cdqe
0x000006d5      mov dword [rbp + rax*4 - 0x60], edx
0x000006d9      add dword [var_64h], 1
0x000006dd      cmp dword [var_64h], 0x13
0x000006e1      jle 0x6ca
```



Init the loop counter variable var_64h (note that it is DWORD == int32_t ) with 0 and jump unconditionally to loop end, where we compare to 0x13 = 19 and start the loop if var_64h is less than 0x13:

```
0x000006c1      mov dword [var_64h], 0
0x000006c8      jmp 0x6dd
0x000006ca      ...
...
0x000006dd      cmp dword [var_64h], 0x13
0x000006e1      jle 0x6ca
```

This is C code:

```C
int32_t var_64h = 0;
while (var_64h < 19) {
	...
}
```

For the loop body, note that the `CDQE` instruction sign-extends a DWORD (32-bit value) in the `EAX` register to a QWORD (64-bit value) in the `RAX` register (i.e. casts to larger type).

The disassembly below needs special comment:

```
0x000006d5      mov dword [rbp + rax*4 - 0x60], edx
```

It writes from register edx into memory area. There are two things noticeable from the formula:

```
rbp - 0x60 
```

This is basically accessing the stack at byte 96. (note that rbp-0x64 holds the var_64h variable and rpb-0x8 holds the stack canary value)

```
 + rax*4
```

This is an offset of 4 bytes into the stack that changes according to value of rax (so its offset in terms of int32_t).

In other words this is an array of size 0x60 - 0x8 (because 0x8 is the canary) = 0x58 which is 88 bytes. The array of 20 int32_t will take up 80 bytes, leaving 8 bytes of free space. Between the canary and the last element of the array at address `rbp - 0x60 + 0x50`.

The body of the loop in C code:

```
array[var_64h] = var_64h + var_64h;
var_64h++;
```

Therefore loop one in C code:

```C
int32_t var_64h = 0;
while (var_64h < 19) {
	array[var_64h] = var_64h + var_64h;
	var_64h++;
}
```



##### Loop number two

Disassembly:

```
0x000006e3      mov dword [var_64h], 0
0x000006ea      jmp 0x70f
0x000006ec      mov eax, dword [var_64h]
0x000006ef      cdqe
0x000006f1      mov edx, dword [rbp + rax*4 - 0x60]
0x000006f5      mov eax, dword [var_64h]
0x000006f8      mov esi, eax
0x000006fa      lea rdi, str.a__d___d ; 0x7b4 ; const char *format
0x00000701      mov eax, 0
0x00000706      call printf        ; sym.imp.printf ; int printf(const char *format)
0x0000070b      add dword [var_64h], 1
0x0000070f      cmp dword [var_64h], 0x13
0x00000713      jle 0x6ec
```

The printf specifier:

```
0x000007b4          .string "a[%d]=%d\n" ; len=10
```



Really similar to loop one, in C code:

```C
var_64h = 0;
while (var_64h < 19) {
	printf("a[%d]=%d\n", var_64h, array[var_64h]);
}
```



##### Final C code:

```c
#include <stdio.h>
#include <time.h>
#include <stdint.h>

int main() {
    int32_t array[20];
	int32_t var_64h = 0;
	while (var_64h < 19) {
		array[var_64h] = var_64h + var_64h;
		var_64h++;
	}
	var_64h = 0;
	while (var_64h < 19) {
		printf("a[%d]=%d\n", var_64h, array[var_64h]);
	}
	return 0;
}
```







### Bin3

```
# sha1sum *
3783812e71c5774e844d7d869fd1a2c6c25cb338  bin1
6cfa97eec60ebc883dcac513c40a90741036b588  bin2
6cfa97eec60ebc883dcac513c40a90741036b588  bin3
ee7a988aeebe4c7b7db06ab48bfe9188768964a1  bin4
40f66560bfb2cf00f2840fa06f95d630f274bc7e  bin5
e1a370626712c8297b657a48a4e63f0d094462f9  bin6
```

Same file as bin2 (or I am a victim of a SHA1 collision attack :)

```
6cfa97eec60ebc883dcac513c40a90741036b588  bin2
6cfa97eec60ebc883dcac513c40a90741036b588  bin3
```





### Bin4

Disassembly:

```
147: int main (int argc, char **argv, char **envp);
; var int64_t var_ch @ rbp-0xc
; var int64_t canary @ rbp-0x8
0x0000071a      push    rbp
0x0000071b      mov     rbp, rsp
0x0000071e      sub     rsp, 0x10
0x00000722      mov     rax, qword fs:[0x28]
0x0000072b      mov     qword [canary], rax
0x0000072f      xor     eax, eax
0x00000731      lea     rdi, str.Enter_an_integer: ; 0x834 ; const char *format
0x00000738      mov     eax, 0
0x0000073d      call    printf     ; sym.imp.printf ; int printf(const char *format)
0x00000742      lea     rax, [var_ch]
0x00000746      mov     rsi, rax
0x00000749      lea     rdi, [0x00000847] ; const char *format
0x00000750      mov     eax, 0
0x00000755      call    __isoc99_scanf ; int scanf(const char *format)
0x0000075a      mov     eax, dword [var_ch]
0x0000075d      and     eax, 1
0x00000760      test    eax, eax
0x00000762      jne     0x77c
0x00000764      mov     eax, dword [var_ch]
0x00000767      mov     esi, eax
0x00000769      lea     rdi, str.d_is_even. ; 0x84a ; const char *format
0x00000770      mov     eax, 0
0x00000775      call    printf     ; sym.imp.printf ; int printf(const char *format)
0x0000077a      jmp     0x792
0x0000077c      mov     eax, dword [var_ch]
0x0000077f      mov     esi, eax
0x00000781      lea     rdi, str.d_is_odd. ; 0x856 ; const char *format
0x00000788      mov     eax, 0
0x0000078d      call    printf     ; sym.imp.printf ; int printf(const char *format)
0x00000792      mov     eax, 0
0x00000797      mov     rdx, qword [canary]
0x0000079b      xor     rdx, qword fs:[0x28]
0x000007a4      je      0x7ab
0x000007a6      call    __stack_chk_fail ; void __stack_chk_fail(void)
0x000007ab      leave
0x000007ac      ret
0x000007ad      nop     dword [rax]
```



##### Printing a prompt

Disasm:

```
0x0000072f      xor     eax, eax
0x00000731      lea     rdi, str.Enter_an_integer: ; 0x834 ; const char *format
0x00000738      mov     eax, 0
0x0000073d      call    printf     ; sym.imp.printf ; int printf(const char *format)
```

Format string:

```
0x00000834          .string "Enter an integer: " ; len=19
```

The C code:

```C
printf("Enter an integer: ");
```



##### Getting user input

```
; var int64_t var_ch @ rbp-0xc
...
0x00000742      lea     rax, [var_ch]
0x00000746      mov     rsi, rax
0x00000749      lea     rdi, [0x00000847] ; const char *format
0x00000750      mov     eax, 0
0x00000755      call    __isoc99_scanf ; int scanf(const char *format)
```

Format string is at address 0x00000847:

```
0x00000847      and     eax, 0x64250064
```

We see that Cutter tries to interpret it as an instruction:

```
and eax, 0x64250064
```

Which does not make sense, but looking at the binary at this address we find the bytes:

```
0x25 0x64 0x00
```

 which is equivalent to ascii characters  0x25 =  `%` and 0x64 = `d` and 0 byte for string terminator used in scanf.

C code:

```C
int64_t i;
...
scanf("%d", &i);
```



##### Checking the input

```
0x0000075a      mov     eax, dword [var_ch]
0x0000075d      and     eax, 1
0x00000760      test    eax, eax
0x00000762      jne     0x77c
0x00000764      mov     eax, dword [var_ch]
0x00000767      mov     esi, eax
0x00000769      lea     rdi, str.d_is_even. ; 0x84a ; const char *format
0x00000770      mov     eax, 0
0x00000775      call    printf     ; sym.imp.printf ; int printf(const char *format)
0x0000077a      jmp     0x792
0x0000077c      mov     eax, dword [var_ch]
0x0000077f      mov     esi, eax
0x00000781      lea     rdi, str.d_is_odd. ; 0x856 ; const char *format
0x00000788      mov     eax, 0
0x0000078d      call    printf     ; sym.imp.printf ; int printf(const char *format)
```



Format strings:

```
;-- str.d_is_even.:
0x0000084a          .string "%d is even." ; len=12
;-- str.d_is_odd.:
0x00000856          .string "%d is odd." ; len=11
```



The actual test is here:

```
0x0000075a      mov     eax, dword [var_ch]
0x0000075d      and     eax, 1
0x00000760      test    eax, eax
```

Testing eax & 1 is just checking if the last bit is set, which is equivalent to checking % 2 == 0.  

C code:

```C
if (num & 1 == 0) {
	printf("%d is even.", i);
} else {
	printf("%d is odd.", i);
}
```



##### Final C code:

```C
#include <stdio.h>
#include <stdint.h>

void main(){
	int64_t i;
	printf("Enter an integer: ");
	scanf("%d", &i);
	if (num & 1 == 0) {
		printf("%d is even.", i);
	} else {
		printf("%d is odd.", i);
	}
	return;
}
```





## Bin5

Using cutter there was no problem finding main.

One way to find main would be to find calls to printf, or uses of strings.

If cutter would have failed then a sure way would be to look at the elf header to find the entry point to the program, this would allow to start unwrapping the code. Then we just have to trace jumps and function calls until we arrive to something that looks like function signature:

```C
int main(int argc, char **argv)
```



Disassembler:

```
189: int main (int argc, char **argv, char **envp);
; var int64_t var_18h @ rbp-0x18
; var int64_t var_14h @ rbp-0x14
; var int64_t var_10h @ rbp-0x10
; var int64_t var_8h @ rbp-0x8
0x0000071a      push rbp
0x0000071b      mov rbp, rsp
0x0000071e      sub rsp, 0x20
0x00000722      mov rax, qword fs:[0x28]
0x0000072b      mov qword [var_8h], rax
0x0000072f      xor eax, eax
0x00000731      mov qword [var_10h], 1
0x00000739      lea rdi, str.Enter_an_integer: ; 0x864 ; const char *format
0x00000740      mov eax, 0
0x00000745      call printf        ; sym.imp.printf ; int printf(const char *format)
0x0000074a      lea rax, [var_18h]
0x0000074e      mov rsi, rax
0x00000751      lea rdi, [0x00000877] ; 2167 ; const char *format
0x00000758      mov eax, 0
0x0000075d      call __isoc99_scanf ; int scanf(const char *format)
0x00000762      mov eax, dword [var_18h]
0x00000765      test eax, eax
0x00000767      jns 0x77c
0x00000769      lea rdi, str.Error ; 0x87a ; const char *format
0x00000770      mov eax, 0
0x00000775      call printf        ; sym.imp.printf ; int printf(const char *format)
0x0000077a      jmp 0x7bc
0x0000077c      mov dword [var_14h], 1
0x00000783      jmp 0x79a
0x00000785      mov eax, dword [var_14h]
0x00000788      cdqe
0x0000078a      mov rdx, qword [var_10h]
0x0000078e      imul rax, rdx
0x00000792      mov qword [var_10h], rax
0x00000796      add dword [var_14h], 1
0x0000079a      mov eax, dword [var_18h]
0x0000079d      cmp dword [var_14h], eax
0x000007a0      jle 0x785
0x000007a2      mov eax, dword [var_18h]
0x000007a5      mov rdx, qword [var_10h]
0x000007a9      mov esi, eax
0x000007ab      lea rdi, str.Result_is__d____llu ; 0x881 ; const char *format
0x000007b2      mov eax, 0
0x000007b7      call printf        ; sym.imp.printf ; int printf(const char *format)
0x000007bc      mov eax, 0
0x000007c1      mov rcx, qword [var_8h]
0x000007c5      xor rcx, qword fs:[0x28]
0x000007ce      je 0x7d5
0x000007d0      call __stack_chk_fail ; void __stack_chk_fail(void)
0x000007d5      leave
0x000007d6      ret
```



This binary is similar to bin4 so I will be more brief. The canary value for stack protection here is called var_8h in the disassembly.

Ask for input:

```
0x0000072f      xor eax, eax
0x00000731      mov qword [var_10h], 1
0x00000739      lea rdi, str.Enter_an_integer: ; 0x864 ; const char *format
0x00000740      mov eax, 0
0x00000745      call printf        ; sym.imp.printf ; int printf(const char *format)
```

C code:

```C
printf("Enter an integer: ");
```



Read user input:

```
; var int64_t var_18h @ rbp-0x18
...
0x0000074a      lea rax, [var_18h]
0x0000074e      mov rsi, rax
0x00000751      lea rdi, [0x00000877] ; 2167 ; const char *format
0x00000758      mov eax, 0
0x0000075d      call __isoc99_scanf ; int scanf(const char *format)
```

scanf string at address 0x00000877 in the binary looks like:

```
0x25 0x64 0x00
```

which is `%d` and 0 byte.

C code:

```C
int64_t var_18h;
...
scanf("%d", &var_18h);
```



Conditional check

```
0x00000762      mov eax, dword [var_18h]
0x00000765      test eax, eax
0x00000767      jns 0x77c
```

C code:

```C
if (var_18h < 0) 
```



Conditional arm if the check is true:

```
0x00000769      lea rdi, str.Error ; 0x87a ; const char *format
0x00000770      mov eax, 0
0x00000775      call printf        ; sym.imp.printf ; int printf(const char *format)
0x0000077a      jmp 0x7bc
...
0x000007bc      mov eax, 0
```

C code:

```C
printf("Error!");
return 0;
```



Loop

```
0x00000731      mov qword [var_10h], 1
...
0x0000077c      mov dword [var_14h], 1
0x00000783      jmp 0x79a
0x00000785      mov eax, dword [var_14h]
0x00000788      cdqe
0x0000078a      mov rdx, qword [var_10h]
0x0000078e      imul rax, rdx
0x00000792      mov qword [var_10h], rax
0x00000796      add dword [var_14h], 1
0x0000079a      mov eax, dword [var_18h]
0x0000079d      cmp dword [var_14h], eax
0x000007a0      jle 0x785
```

Multiplication done at:

```
0x0000078e      imul rax, rdx
```

C code:

```C
var_10h = 1;
...
var_14h = 1;
while (var_14h < var_18h) {
	var_10h *= var_14h;
	var_14h++;
}
```



Print answer:

```
0x000007a2      mov eax, dword [var_18h]
0x000007a5      mov rdx, qword [var_10h]
0x000007a9      mov esi, eax
0x000007ab      lea rdi, str.Result_is__d____llu ; 0x881 ; const char *format
0x000007b2      mov eax, 0
0x000007b7      call printf        ; sym.imp.printf ; int printf(const char *format)
```

C code:

```Enter tC
printf("Result is %d = %d", in, res);
```



##### Final C code:

```C
#include <stdio.h>
#include <stdint.h>

int main(void) {
	int32_t var_10h = 1;
	int32_t var_14h = 1;
	int64_t var_18h;
	
	printf("Enter an integer: ");
	scanf("%d", &var_18h);
	
	if (var_18h < 0) {
		printf("Error!");
		return 0;
	}
	
	while (var_14h < var_18h) {
		var_10h *= var_14h;
		var_14h++;
	}
	
	printf("Result is %d = %d", in, res);
}
```





## Bin6

The file has about 100 random functions, but Cutter was able to find main!

Disassembly:

```
134: int main (int argc, char **argv, char **envp);
; var int64_t var_14h @ rbp-0x14
; var int64_t var_10h @ rbp-0x10
; var int64_t var_ch @ rbp-0xc
; var int64_t var_8h @ rbp-0x8
0x00400b6d      push    rbp
0x00400b6e      mov     rbp, rsp
0x00400b71      sub     rsp, 0x20
0x00400b75      mov     rax, qword fs:[0x28]
0x00400b7e      mov     qword [var_8h], rax
0x00400b82      xor     eax, eax
0x00400b84      lea     rdi, str.Enter_two_integers: ; 0x4abc64 ; int64_t arg_8h
0x00400b8b      mov     eax, 0
0x00400b90      call    fcn.0040f6e0
0x00400b95      lea     rdx, [var_10h] ; int64_t arg3
0x00400b99      lea     rax, [var_14h]
0x00400b9d      mov     rsi, rax   ; int64_t arg2
0x00400ba0      lea     rdi, str.d__d ; 0x4abc79 ; int64_t arg1
0x00400ba7      mov     eax, 0
0x00400bac      call    fcn.0040f860 ; fcn.0040f7a0+0xc0
0x00400bb1      mov     edx, dword [var_14h]
0x00400bb4      mov     eax, dword [var_10h]
0x00400bb7      add     eax, edx
0x00400bb9      mov     dword [var_ch], eax
0x00400bbc      mov     edx, dword [var_10h] ; int64_t arg2
0x00400bbf      mov     eax, dword [var_14h]
0x00400bc2      mov     ecx, dword [var_ch] ; int64_t arg3
0x00400bc5      mov     esi, eax   ; int64_t arg1
0x00400bc7      lea     rdi, str.d____d____d ; 0x4abc7f ; int64_t arg_8h
0x00400bce      mov     eax, 0
0x00400bd3      call    fcn.0040f6e0
0x00400bd8      mov     eax, 0
0x00400bdd      mov     rsi, qword [var_8h]
0x00400be1      xor     rsi, qword fs:[0x28]
0x00400bea      je      0x400bf1
0x00400bec      call    fcn.0044bc20
0x00400bf1      leave
0x00400bf2      ret
```



The printf and scanf strings:

```
;-- str.Enter_two_integers::
0x004abc64          .string "Enter two integers: " ; len=21
;-- str.d__d:
0x004abc79          .string "%d %d" ; len=6
;-- str.d____d____d:
0x004abc7f          .string "%d + %d = %d" ; len=13
```



Calling printf:

```
0x00400b7e      mov     qword [var_8h], rax
0x00400b82      xor     eax, eax
0x00400b84      lea     rdi, str.Enter_two_integers: ; 0x4abc64 ; int64_t arg_8h
0x00400b8b      mov     eax, 0
0x00400b90      call    fcn.0040f6e0
```

Calling scanf:

```
0x00400b95      lea     rdx, [var_10h] ; int64_t arg3
0x00400b99      lea     rax, [var_14h]
0x00400b9d      mov     rsi, rax   ; int64_t arg2
0x00400ba0      lea     rdi, str.d__d ; 0x4abc79 ; int64_t arg1
0x00400ba7      mov     eax, 0
0x00400bac      call    fcn.0040f860 ; fcn.0040f7a0+0xc0
```

Doing the addition:

```
0x00400bb1      mov     edx, dword [var_14h]
0x00400bb4      mov     eax, dword [var_10h]
0x00400bb7      add     eax, edx
0x00400bb9      mov     dword [var_ch], eax
```



Calling printf with result:

```
0x00400bbc      mov     edx, dword [var_10h] ; int64_t arg2
0x00400bbf      mov     eax, dword [var_14h]
0x00400bc2      mov     ecx, dword [var_ch] ; int64_t arg3
0x00400bc5      mov     esi, eax   ; int64_t arg1
0x00400bc7      lea     rdi, str.d____d____d ; 0x4abc7f ; int64_t arg_8h
```



And another call to check for stack smashing:

```
0x00400bea      je      0x400bf1
0x00400bec      call    fcn.0044bc20
```



The equivalent C code:

```c
#include <stdio.h>
#include <stdint.h>

void main(void) {
	int32_t var_10h, var_14h, var_ch;
	printf("Enter two integers: ");
	scanf("%d %d", &var_10h, &var_14h);
	var_ch = var_10h + var_14h;
	printf("%d + %d = %d", var_10h, var_14h, var_ch);
}
```

