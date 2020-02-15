# AS Lab 3 - understanding assembly

Artem Abramov



## 1. Preparation

#### a. You can use any debugger/disassembler as long as it supports the given task architecture (32bit or 64bit)
I choose GDB.



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
sudo docker cp Lab4-binaries testbin:/root/Lab4-binaries
```



#### d. Check some writeups about some CTF to see what you should/shouldn’t include in your report

Ok.



#### e. Try to do the lab in a Linux VM, as you might need to disable ASLR

I will disable ASLR:

```
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```





## 2. Theory



#### a. What is ASLR, and why do we need it?

Address Space Layout Randomization is a technique to make exploits more difficult. Before this kernel feature was introduced the executables were loaded to same addresses in virtual memory. When an exploit (e.g. based on stack overflow) was found, then to access data/function in the exploited binary the author   could just hard code certain memory addresses (i.e. the return-to-libc attack). ASLR makes it much harder because the addresses are not fixed and change between different runs of the program. The programs support this by a linker feature called PIE which inserts GOT (global offset table) into the binary. This allows the program to be relocated when its first loaded into memory by the kernel. 



#### b. How can the debugger insert a breakpoint in the debugged binary/application?

The debugger first installs a handler for the `int 3` interrupts, then when setting a new breakpoint the debugger patches the program (in RAM) to insert a special one-byte instruction `int 3` at the place where execution should break. The original byte value is saved somewhere. When CPU execution reaches the `int 3`, it triggers a software interrupt, which is handled by the compiler. The compiler can deduce which breakpoint was hit and it fixes the patch it made in the beginning, i.e. it restores the original byte value.  Then the program execution can continue as if it was not modified.



sources: 

- https://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints
- https://eli.thegreenplace.net/2011/11/03/position-independent-code-pic-in-shared-libraries/





## 3. Reversing

#### a. Disable ASLR using the following command “sudo sysctl -w kernel.randomize_va_space=0”

I will disable ASLR (this setting is reflected in the docker container as well):

```
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```



#### b. Load the binaries into a disassembler/debugger

Before loading I wanted to get an overview of what the program does, so I run it through strace.

Before running strace I installed the following libraries into docker:

```
sudo apt-get install libc6-i386 lib32stdc++6 lib32gcc1 lib32ncurses5 lib32z1
```

Then I run the 32-bit executable:

```
# strace ./sample32 
execve("./sample32", ["./sample32"], 0x7fffffffe780 /* 9 vars */) = 0
strace: [ Process PID=826 runs in 32 bit mode. ]
brk(NULL)                               = 0x56558000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
mmap2(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xf7fd0000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat64(3, {st_mode=S_IFREG|0644, st_size=13365, ...}) = 0
mmap2(NULL, 13365, PROT_READ, MAP_PRIVATE, 3, 0) = 0xf7fcc000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/lib32/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\1\1\1\3\0\0\0\0\0\0\0\0\3\0\3\0\1\0\0\0\0\220\1\0004\0\0\0"..., 512) = 512
fstat64(3, {st_mode=S_IFREG|0755, st_size=1926828, ...}) = 0
mmap2(NULL, 1935900, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0xf7df3000
mprotect(0xf7fc5000, 4096, PROT_NONE)   = 0
mmap2(0xf7fc6000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1d2000) = 0xf7fc6000
mmap2(0xf7fc9000, 10780, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0xf7fc9000
close(3)                                = 0
set_thread_area({entry_number=-1, base_addr=0xf7fd10c0, limit=0x0fffff, seg_32bit=1, contents=0, read_exec_only=0, limit_in_pages=1, seg_not_present=0, useable=1}) = 0 (entry_number=12)
mprotect(0xf7fc6000, 8192, PROT_READ)   = 0
mprotect(0x56556000, 4096, PROT_READ)   = 0
mprotect(0xf7ffc000, 4096, PROT_READ)   = 0
munmap(0xf7fcc000, 13365)               = 0
fstat64(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 0), ...}) = 0
brk(NULL)                               = 0x56558000
brk(0x56579000)                         = 0x56579000
brk(0x5657a000)                         = 0x5657a000
write(1, "In main(), x is stored at 0xffff"..., 38In main(), x is stored at 0xffffd78c.
) = 38
write(1, "In sample_function(), i is store"..., 49In sample_function(), i is stored at 0xffffd76c.
) = 49
write(1, "In sample_function(), buffer is "..., 54In sample_function(), buffer is stored at 0xffffd762.
) = 54
write(1, "Value of i before calling gets()"..., 45Value of i before calling gets(): 0xffffffff
) = 45
fstat64(0, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 0), ...}) = 0
read(0, dawdad a dasd ad asd sd 
"dawdad a dasd ad asd sd \n", 1024) = 25
write(1, "Value of i after calling gets():"..., 44Value of i after calling gets(): 0x20647361
) = 44
--- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_MAPERR, si_addr=0xff00205c} ---
+++ killed by SIGSEGV (core dumped) +++
Segmentation fault (core dumped)
```



I loaded the binaries in GDB:

```
# gdb sample32 
GNU gdb (Ubuntu 8.1-0ubuntu3.2) 8.1.0.20180409-git
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from sample32...done.
(gdb) break main
Breakpoint 1 at 0x5ef: file sample.c, line 21.
(gdb) run
Starting program: /root/Lab4-binaries/sample32 

Breakpoint 1, main () at sample.c:21
21	sample.c: No such file or directory.
(gdb) disassemble main
Dump of assembler code for function main:
   0x565555d3 <+0>:	lea    0x4(%esp),%ecx
   0x565555d7 <+4>:	and    $0xfffffff0,%esp
   0x565555da <+7>:	pushl  -0x4(%ecx)
   0x565555dd <+10>:	push   %ebp
   0x565555de <+11>:	mov    %esp,%ebp
   0x565555e0 <+13>:	push   %ebx
   0x565555e1 <+14>:	push   %ecx
   0x565555e2 <+15>:	sub    $0x10,%esp
   0x565555e5 <+18>:	call   0x5655561b <__x86.get_pc_thunk.ax>
   0x565555ea <+23>:	add    $0x19ea,%eax
=> 0x565555ef <+28>:	lea    -0xc(%ebp),%edx
   0x565555f2 <+31>:	sub    $0x8,%esp
   0x565555f5 <+34>:	push   %edx
   0x565555f6 <+35>:	lea    -0x1878(%eax),%edx
   0x565555fc <+41>:	push   %edx
   0x565555fd <+42>:	mov    %eax,%ebx
   0x565555ff <+44>:	call   0x565553d0 <printf@plt>
   0x56555604 <+49>:	add    $0x10,%esp
   0x56555607 <+52>:	call   0x5655554d <sample_function>
   0x5655560c <+57>:	mov    $0x0,%eax
   0x56555611 <+62>:	lea    -0x8(%ebp),%esp
   0x56555614 <+65>:	pop    %ecx
   0x56555615 <+66>:	pop    %ebx
   0x56555616 <+67>:	pop    %ebp
   0x56555617 <+68>:	lea    -0x4(%ecx),%esp
   0x5655561a <+71>:	ret    
End of assembler dump.
```



And the 64 bit:

```
# gdb sample64 
GNU gdb (Ubuntu 8.1-0ubuntu3.2) 8.1.0.20180409-git
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from sample64...done.
(gdb) break main
Breakpoint 1 at 0x717: file sample.c, line 22.
(gdb) run
Starting program: /root/Lab4-binaries/sample64 

Breakpoint 1, main () at sample.c:22
22	sample.c: No such file or directory.
(gdb) disassemble main
Dump of assembler code for function main:
   0x000055555555470f <+0>:	push   %rbp
   0x0000555555554710 <+1>:	mov    %rsp,%rbp
   0x0000555555554713 <+4>:	sub    $0x10,%rsp
=> 0x0000555555554717 <+8>:	lea    -0x4(%rbp),%rax
   0x000055555555471b <+12>:	mov    %rax,%rsi
   0x000055555555471e <+15>:	lea    0x163(%rip),%rdi        # 0x555555554888
   0x0000555555554725 <+22>:	mov    $0x0,%eax
   0x000055555555472a <+27>:	callq  0x555555554550 <printf@plt>
   0x000055555555472f <+32>:	mov    $0x0,%eax
   0x0000555555554734 <+37>:	callq  0x55555555468a <sample_function>
   0x0000555555554739 <+42>:	mov    $0x0,%eax
   0x000055555555473e <+47>:	leaveq 
   0x000055555555473f <+48>:	retq   
End of assembler dump.

```





And the sample64-2:

```
# gdb sample64-2 
GNU gdb (Ubuntu 8.1-0ubuntu3.2) 8.1.0.20180409-git
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from sample64-2...(no debugging symbols found)...done.
(gdb) break main
Breakpoint 1 at 0x7a6
(gdb) run
Starting program: /root/Lab4-binaries/sample64-2 

Breakpoint 1, 0x00005555555547a6 in main ()
(gdb) disassemble main
Dump of assembler code for function main:
   0x00005555555547a2 <+0>:	push   %rbp
   0x00005555555547a3 <+1>:	mov    %rsp,%rbp
=> 0x00005555555547a6 <+4>:	sub    $0x10,%rsp
   0x00005555555547aa <+8>:	mov    %fs:0x28,%rax
   0x00005555555547b3 <+17>:	mov    %rax,-0x8(%rbp)
   0x00005555555547b7 <+21>:	xor    %eax,%eax
   0x00005555555547b9 <+23>:	lea    -0xc(%rbp),%rax
   0x00005555555547bd <+27>:	mov    %rax,%rsi
   0x00005555555547c0 <+30>:	lea    0x181(%rip),%rdi        # 0x555555554948
   0x00005555555547c7 <+37>:	mov    $0x0,%eax
   0x00005555555547cc <+42>:	callq  0x5555555545c0 <printf@plt>
   0x00005555555547d1 <+47>:	mov    $0x0,%eax
   0x00005555555547d6 <+52>:	callq  0x5555555546fa <sample_function>
   0x00005555555547db <+57>:	mov    $0x0,%eax
   0x00005555555547e0 <+62>:	mov    -0x8(%rbp),%rdx
   0x00005555555547e4 <+66>:	xor    %fs:0x28,%rdx
   0x00005555555547ed <+75>:	je     0x5555555547f4 <main+82>
   0x00005555555547ef <+77>:	callq  0x5555555545b0 <__stack_chk_fail@plt>
   0x00005555555547f4 <+82>:	leaveq 
   0x00005555555547f5 <+83>:	retq   
End of assembler dump.

```







#### c. Does the function prologue and epilogue differ in 32bit and 64bit?

They are different for 32 and 64 bit versions. Derived by reading (https://aaronbloomfield.github.io/pdr/book/x86-32bit-ccc-chapter.pdf) and looking at main and sample_function.

First use the command to change intel flavour:

```
(gdb) set disassembly-flavor intel
```



Prologue in 64-bit mode:

```
push   rbp
mov    rbp,rsp
```

The next instruction normally subtracts from `%rsp` register to reserve space on the stack for variables.

Epilogue in 64 bit:

Sometimes there is a `nop` before `leaveq`, looks like its for alignment.

```
leaveq 
retq
```



Prologue in 32-bit mode:

Main has some extra code before this prologue, because its a special function. 

```
lea    ecx,[esp+0x4]
and    esp,0xfffffff0
push   DWORD PTR [ecx-0x4]
```



The other functions get a normal prologue:

```
push   ebp
mov    ebp,esp
```



Epilogue in 32-bit mode:

```
pop    ebp
mov    esp,ebp
ret 
```

The same instructions can also be generated as (generated by the compiler):

```
leave  
ret 
```



#### d. Does function calls differ in 32bit and 64bit? What about argument passing?

The example appeared to not pass any arguments to `sample_function`, therefore to better understand how argument passing works I constructed the following C code:

```
#include <stdio.h>

struct vec2named {
        char name[16];
        int x;
        int y;
};

int  func1(int a) {
        printf("%d\n", a);
        return a + 10;
}

int func2(struct vec2named v) {
        printf("x: %d, y: %d\n", v.x, v.y);
        v.name[0] = 'a';
        v.name[1] = 'a';
        v.name[2] = 'a';
        v.name[3] = 'a';
        v.name[4] = 'a';
        v.name[5] = 'a';
        v.name[6] = 'a';
        v.name[7] = 'a';
        v.name[8] = '\000';
        return 0;
}

int main(void) {
        struct vec2named v = {};
        v.x = 2;
        v.y = 3;
        func1(5);
        func2(v);
        return v.x;

}
```

Note that because we pass `v` to `func2` by value, if we try to `printf("%s", v.name);` right after calling func2(v) we would not get any output, because .name field is modified on the copy of the struct. 

##### Analyzing the 64 bit function calls

Disassembly is below:

```
   0x000000000000071b <+0>:	push   rbp
   0x000000000000071c <+1>:	mov    rbp,rsp
   0x000000000000071f <+4>:	sub    rsp,0x20
   0x0000000000000723 <+8>:	mov    rax,QWORD PTR fs:0x28
   0x000000000000072c <+17>:	mov    QWORD PTR [rbp-0x8],rax
   0x0000000000000730 <+21>:	xor    eax,eax
   0x0000000000000732 <+23>:	mov    QWORD PTR [rbp-0x20],0x0
   0x000000000000073a <+31>:	mov    QWORD PTR [rbp-0x18],0x0
   0x0000000000000742 <+39>:	mov    QWORD PTR [rbp-0x10],0x0
   0x000000000000074a <+47>:	mov    DWORD PTR [rbp-0x10],0x2
   0x0000000000000751 <+54>:	mov    DWORD PTR [rbp-0xc],0x3
   0x0000000000000758 <+61>:	mov    edi,0x5
   0x000000000000075d <+66>:	call   0x6aa <func1>
   0x0000000000000762 <+71>:	sub    rsp,0x8
   0x0000000000000766 <+75>:	push   QWORD PTR [rbp-0x10]
   0x0000000000000769 <+78>:	push   QWORD PTR [rbp-0x18]
   0x000000000000076c <+81>:	push   QWORD PTR [rbp-0x20]
   0x000000000000076f <+84>:	call   0x6d3 <func2>
   0x0000000000000774 <+89>:	add    rsp,0x20
   0x0000000000000778 <+93>:	mov    eax,DWORD PTR [rbp-0x10]
   0x000000000000077b <+96>:	mov    rdx,QWORD PTR [rbp-0x8]
   0x000000000000077f <+100>:	xor    rdx,QWORD PTR fs:0x28
   0x0000000000000788 <+109>:	je     0x78f <main+116>
   0x000000000000078a <+111>:	call   0x570 <__stack_chk_fail@plt>
   0x000000000000078f <+116>:	leave  
   0x0000000000000790 <+117>:	ret
```



Calling func1(5) (self explanatory):

```
mov    edi,0x5
call   0x6aa <func1>
```



Calling the func2(v) starting at instruction  <+71>:

```
sub    rsp,0x8
push   QWORD PTR [rbp-0x10]
push   QWORD PTR [rbp-0x18]
push   QWORD PTR [rbp-0x20]
call   0x6d3 <func2>
```

Reserves space (for whatever reason) on the stack for 8 more bytes in addition to what was already reserved. Then push 64 -bit values onto the stack (we know this because of `QWORD PTR`) from three memory locations. This memory location were filled in the beginning of the function as seen here:

```
<+23>:	mov    QWORD PTR [rbp-0x20],0x0
<+31>:	mov    QWORD PTR [rbp-0x18],0x0
<+39>:	mov    QWORD PTR [rbp-0x10],0x0
```

Its the initializer for the struct that gets passed to `func2`.

Inside func2 the struct is consumed. 

First here is the disassembly for func2:

```
(gdb) disassemble func2
Dump of assembler code for function func2:
   0x00000000000006d3 <+0>:	push   rbp
   0x00000000000006d4 <+1>:	mov    rbp,rsp
   0x00000000000006d7 <+4>:	mov    edx,DWORD PTR [rbp+0x24]
   0x00000000000006da <+7>:	mov    eax,DWORD PTR [rbp+0x20]
   0x00000000000006dd <+10>:	mov    esi,eax
   0x00000000000006df <+12>:	lea    rdi,[rip+0x142]        # 0x828
   0x00000000000006e6 <+19>:	mov    eax,0x0
   0x00000000000006eb <+24>:	call   0x580 <printf@plt>
   0x00000000000006f0 <+29>:	mov    BYTE PTR [rbp+0x10],0x61
   0x00000000000006f4 <+33>:	mov    BYTE PTR [rbp+0x11],0x61
   0x00000000000006f8 <+37>:	mov    BYTE PTR [rbp+0x12],0x61
   0x00000000000006fc <+41>:	mov    BYTE PTR [rbp+0x13],0x61
   0x0000000000000700 <+45>:	mov    BYTE PTR [rbp+0x14],0x61
   0x0000000000000704 <+49>:	mov    BYTE PTR [rbp+0x15],0x61
   0x0000000000000708 <+53>:	mov    BYTE PTR [rbp+0x16],0x61
   0x000000000000070c <+57>:	mov    BYTE PTR [rbp+0x17],0x61
   0x0000000000000710 <+61>:	mov    BYTE PTR [rbp+0x18],0x0
   0x0000000000000714 <+65>:	mov    eax,0x0
   0x0000000000000719 <+70>:	pop    rbp
   0x000000000000071a <+71>:	ret 
```

 

The compiler knows that this function will take a struct that will be passed on the stack. Therefore to extract its values it refers to them as 

```
<+4>:	mov    edx,DWORD PTR [rbp+0x24]
<+7>:	mov    eax,DWORD PTR [rbp+0x20]
```

The interesting part is the `+` sign. This function looks into the data ABOVE (at a higher address) its stack, so it takes what was put on the stack by the `main` function. This is argument passing for structs.

To finish the function we write the return value into the EAX register (this function returns an int): 

```
<+65>:	mov    eax,0x0
```

There is no need to adjust ESP, because the function did not allocate any stack space of its own, so we just pop the RBP and return:

```
<+70>:	pop    rbp
<+71>:	ret
```



##### Analyzing the 32 bit function calls

I compiled the same code forcing compilation in 32-bit mode.

Disassembly for the main function:

```
Dump of assembler code for function main:
   0x5655560b <+0>:	lea    ecx,[esp+0x4]
   0x5655560f <+4>:	and    esp,0xfffffff0
   0x56555612 <+7>:	push   DWORD PTR [ecx-0x4]
   0x56555615 <+10>:	push   ebp
   0x56555616 <+11>:	mov    ebp,esp
   0x56555618 <+13>:	push   ecx
=> 0x56555619 <+14>:	sub    esp,0x24
   0x5655561c <+17>:	call   0x565556a4 <__x86.get_pc_thunk.ax>
   0x56555621 <+22>:	add    eax,0x19b3
   0x56555626 <+27>:	mov    eax,gs:0x14
   0x5655562c <+33>:	mov    DWORD PTR [ebp-0xc],eax
   0x5655562f <+36>:	xor    eax,eax
   0x56555631 <+38>:	mov    ecx,0x0
   0x56555636 <+43>:	mov    eax,0x18
   0x5655563b <+48>:	and    eax,0xfffffffc
   0x5655563e <+51>:	mov    edx,eax
   0x56555640 <+53>:	mov    eax,0x0
   0x56555645 <+58>:	mov    DWORD PTR [ebp+eax*1-0x24],ecx
   0x56555649 <+62>:	add    eax,0x4
   0x5655564c <+65>:	cmp    eax,edx
   0x5655564e <+67>:	jb     0x56555645 <main+58>
   0x56555650 <+69>:	mov    DWORD PTR [ebp-0x14],0x2
   0x56555657 <+76>:	mov    DWORD PTR [ebp-0x10],0x3
   0x5655565e <+83>:	sub    esp,0xc
   0x56555661 <+86>:	push   0x5
   0x56555663 <+88>:	call   0x5655557d <func1>
   0x56555668 <+93>:	add    esp,0x10
   0x5655566b <+96>:	sub    esp,0x8
   0x5655566e <+99>:	push   DWORD PTR [ebp-0x10]
   0x56555671 <+102>:	push   DWORD PTR [ebp-0x14]
   0x56555674 <+105>:	push   DWORD PTR [ebp-0x18]
   0x56555677 <+108>:	push   DWORD PTR [ebp-0x1c]
   0x5655567a <+111>:	push   DWORD PTR [ebp-0x20]
   0x5655567d <+114>:	push   DWORD PTR [ebp-0x24]
   0x56555680 <+117>:	call   0x565555b0 <func2>
   0x56555685 <+122>:	add    esp,0x20
   0x56555688 <+125>:	mov    eax,DWORD PTR [ebp-0x14]
   0x5655568b <+128>:	mov    edx,DWORD PTR [ebp-0xc]
   0x5655568e <+131>:	xor    edx,DWORD PTR gs:0x14
   0x56555695 <+138>:	je     0x5655569c <main+145>
   0x56555697 <+140>:	call   0x56555720 <__stack_chk_fail_local>
   0x5655569c <+145>:	mov    ecx,DWORD PTR [ebp-0x4]
   0x5655569f <+148>:	leave  
   0x565556a0 <+149>:	lea    esp,[ecx-0x4]
   0x565556a3 <+152>:	ret    
End of assembler dump.
```



Analysing argument passing to func1(5):

```
   0x5655565e <+83>:	sub    esp,0xc
   0x56555661 <+86>:	push   0x5
   0x56555663 <+88>:	call   0x5655557d <func1>
```

First reserve 12 bytes on the stack, push a further 4 byte value on the stack. 

To find the size of int execute the following the following in gdb:

```
(gdb) p sizeof(int)
$1 = 4
```

How does func1 consume the argument? 

Here is the disassembly for reference:

```
Dump of assembler code for function func1:
   0x5655557d <+0>:	push   ebp
   0x5655557e <+1>:	mov    ebp,esp
   0x56555580 <+3>:	push   ebx
   0x56555581 <+4>:	sub    esp,0x4
   0x56555584 <+7>:	call   0x565556a4 <__x86.get_pc_thunk.ax>
   0x56555589 <+12>:	add    eax,0x1a4b
   0x5655558e <+17>:	sub    esp,0x8
   0x56555591 <+20>:	push   DWORD PTR [ebp+0x8]
   0x56555594 <+23>:	lea    edx,[eax-0x1884]
   0x5655559a <+29>:	push   edx
   0x5655559b <+30>:	mov    ebx,eax
   0x5655559d <+32>:	call   0x56555400 <printf@plt>
   0x565555a2 <+37>:	add    esp,0x10
   0x565555a5 <+40>:	mov    eax,DWORD PTR [ebp+0x8]
   0x565555a8 <+43>:	add    eax,0xa
   0x565555ab <+46>:	mov    ebx,DWORD PTR [ebp-0x4]
   0x565555ae <+49>:	leave  
   0x565555af <+50>:	ret    
End of assembler dump.
```



First func1 must printf the value it received, to pass the value to printf we pass the int by stack again (for some reason). Reserve 8 bytes, then pushing a further DWORD (32 bits) onto the stack from address EBP + 0x8 (i.e. take it from the stack of parent function):

```
   0x5655558e <+17>:	sub    esp,0x8
   0x56555591 <+20>:	push   DWORD PTR [ebp+0x8]
```

At the end the function must return `a + 10` in EAX register. This is visible in assembly. 

Load variable `a` into the EAX register and add `0xa` which is 10 in hex:

```
   0x565555a5 <+40>:	mov    eax,DWORD PTR [ebp+0x8]
   0x565555a8 <+43>:	add    eax,0xa
```



So we can see that even a single integer was passed on the stack, NOT via the registers (as it was in 64 bit mode).



Next analysing how the struct gets passed. First some compiler idiosyncrasy, we can see it frees 16 bytes on the stack and then reserves 8 bytes on the stack:

```
   0x56555668 <+93>:	add    esp,0x10
   0x5655566b <+96>:	sub    esp,0x8
```



Then pushes the whole structure onto the stack (24 bytes):

```
   0x5655566e <+99>:	push   DWORD PTR [ebp-0x10]
   0x56555671 <+102>:	push   DWORD PTR [ebp-0x14]
   0x56555674 <+105>:	push   DWORD PTR [ebp-0x18]
   0x56555677 <+108>:	push   DWORD PTR [ebp-0x1c]
   0x5655567a <+111>:	push   DWORD PTR [ebp-0x20]
   0x5655567d <+114>:	push   DWORD PTR [ebp-0x24]
   0x56555680 <+117>:	call   0x565555b0 <func2>
```



How does func2 consume the struct? 

Below is disassembly of func2:

```
Dump of assembler code for function func2:
   0x565555b0 <+0>:	push   ebp
   0x565555b1 <+1>:	mov    ebp,esp
   0x565555b3 <+3>:	push   ebx
   0x565555b4 <+4>:	sub    esp,0x4
   0x565555b7 <+7>:	call   0x565556a4 <__x86.get_pc_thunk.ax>
   0x565555bc <+12>:	add    eax,0x1a18
   0x565555c1 <+17>:	mov    ecx,DWORD PTR [ebp+0x1c]
   0x565555c4 <+20>:	mov    edx,DWORD PTR [ebp+0x18]
   0x565555c7 <+23>:	sub    esp,0x4
   0x565555ca <+26>:	push   ecx
   0x565555cb <+27>:	push   edx
   0x565555cc <+28>:	lea    edx,[eax-0x1880]
   0x565555d2 <+34>:	push   edx
   0x565555d3 <+35>:	mov    ebx,eax
   0x565555d5 <+37>:	call   0x56555400 <printf@plt>
   0x565555da <+42>:	add    esp,0x10
   0x565555dd <+45>:	mov    BYTE PTR [ebp+0x8],0x61
   0x565555e1 <+49>:	mov    BYTE PTR [ebp+0x9],0x61
   0x565555e5 <+53>:	mov    BYTE PTR [ebp+0xa],0x61
   0x565555e9 <+57>:	mov    BYTE PTR [ebp+0xb],0x61
   0x565555ed <+61>:	mov    BYTE PTR [ebp+0xc],0x61
   0x565555f1 <+65>:	mov    BYTE PTR [ebp+0xd],0x61
   0x565555f5 <+69>:	mov    BYTE PTR [ebp+0xe],0x61
   0x565555f9 <+73>:	mov    BYTE PTR [ebp+0xf],0x61
   0x565555fd <+77>:	mov    BYTE PTR [ebp+0x10],0x0
   0x56555601 <+81>:	mov    eax,0x0
   0x56555606 <+86>:	mov    ebx,DWORD PTR [ebp-0x4]
   0x56555609 <+89>:	leave  
   0x5655560a <+90>:	ret    
End of assembler dump.
```



First we load the `v.x` and `v.y` into registers ECX and EDX and then we push them:

```
   0x565555c1 <+17>:	mov    ecx,DWORD PTR [ebp+0x1c]
   0x565555c4 <+20>:	mov    edx,DWORD PTR [ebp+0x18]
   ...
   0x565555ca <+26>:	push   ecx
   0x565555cb <+27>:	push   edx
```

Now they are ready for printf to consume.



Finally the function does a `return 0` as follows:

```
   0x56555601 <+81>:	mov    eax,0x0
```



This concludes the analysis of function calls and argument passing differences between 32 bit and 64 bit.



#### e. What does the command  ldd  do? “ldd BINARY-NAME”

From `man ldd`:

```
ldd  prints the shared objects (shared libraries) required by each program or shared object specified on the command line.
```

Actually to run a 32 bit binary on the 64 bit machine I had to install a number of libraries. Running LDD gives me as the user information what shared objects must be installed on the system to run a certain application. 

Running ldd on the given binaries:

```
# ldd sample32 
	linux-gate.so.1 (0xf7fd5000)
	libc.so.6 => /lib32/libc.so.6 (0xf7df0000)
	/lib/ld-linux.so.2 (0xf7fd6000)
```



```
# ldd sample64 
	linux-vdso.so.1 (0x00007ffff7ffb000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ffff77e2000)
	/lib64/ld-linux-x86-64.so.2 (0x00007ffff7dd5000)
```



```
# ldd sample64-2 
	linux-vdso.so.1 (0x00007ffff7ffb000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ffff77e2000)
	/lib64/ld-linux-x86-64.so.2 (0x00007ffff7dd5000)
```



#### f. Why in the “sample64-2“ binary, the value of  i  didn’t change even if our input was very long?

First piece of information is given in output:

```
./sample64-2 
In main(), x is stored at 0x7fffffffe644.
In sample_function(), i is stored at 0x7fffffffe610.
In sample_function(), buffer is stored at 0x7fffffffe61e.
```

We can see that i is stored at address earlier than buffer.

The disassembly of the interesting function sample_function is below:

```
Dump of assembler code for function sample_function:
   0x00005555555546fa <+0>:	push   rbp
   0x00005555555546fb <+1>:	mov    rbp,rsp
   0x00005555555546fe <+4>:	sub    rsp,0x20
   0x0000555555554702 <+8>:	mov    rax,QWORD PTR fs:0x28
   0x000055555555470b <+17>:	mov    QWORD PTR [rbp-0x8],rax
=> 0x000055555555470f <+21>:	xor    eax,eax
   0x0000555555554711 <+23>:	mov    eax,0xffffffff
   0x0000555555554716 <+28>:	mov    QWORD PTR [rbp-0x20],rax
   0x000055555555471a <+32>:	lea    rax,[rbp-0x20]
   0x000055555555471e <+36>:	mov    rsi,rax
   0x0000555555554721 <+39>:	lea    rdi,[rip+0x160]        # 0x555555554888
   0x0000555555554728 <+46>:	mov    eax,0x0
   0x000055555555472d <+51>:	call   0x5555555545c0 <printf@plt>
   0x0000555555554732 <+56>:	lea    rax,[rbp-0x12]
   0x0000555555554736 <+60>:	mov    rsi,rax
   0x0000555555554739 <+63>:	lea    rdi,[rip+0x178]        # 0x5555555548b8
   0x0000555555554740 <+70>:	mov    eax,0x0
   0x0000555555554745 <+75>:	call   0x5555555545c0 <printf@plt>
   0x000055555555474a <+80>:	mov    rax,QWORD PTR [rbp-0x20]
   0x000055555555474e <+84>:	mov    rsi,rax
   0x0000555555554751 <+87>:	lea    rdi,[rip+0x190]        # 0x5555555548e8
   0x0000555555554758 <+94>:	mov    eax,0x0
   0x000055555555475d <+99>:	call   0x5555555545c0 <printf@plt>
   0x0000555555554762 <+104>:	lea    rax,[rbp-0x12]
   0x0000555555554766 <+108>:	mov    rdi,rax
   0x0000555555554769 <+111>:	mov    eax,0x0
   0x000055555555476e <+116>:	call   0x5555555545d0 <gets@plt>
   0x0000555555554773 <+121>:	mov    rax,QWORD PTR [rbp-0x20]
   0x0000555555554777 <+125>:	mov    rsi,rax
   0x000055555555477a <+128>:	lea    rdi,[rip+0x197]        # 0x555555554918
   0x0000555555554781 <+135>:	mov    eax,0x0
   0x0000555555554786 <+140>:	call   0x5555555545c0 <printf@plt>
   0x000055555555478b <+145>:	nop
   0x000055555555478c <+146>:	mov    rax,QWORD PTR [rbp-0x8]
   0x0000555555554790 <+150>:	xor    rax,QWORD PTR fs:0x28
   0x0000555555554799 <+159>:	je     0x5555555547a0 <sample_function+166>
   0x000055555555479b <+161>:	call   0x5555555545b0 <__stack_chk_fail@plt>
   0x00005555555547a0 <+166>:	leave  
   0x00005555555547a1 <+167>:	ret    
End of assembler dump.
```



We see that there are 4 printf calls and 1 gets call. The function `gets` has a bad reputation because it just accepts any length from STDIN and puts it into a buffer, this is the easiest way to cause stack overflow. If we type a long string we will override the contents of the buffer, write past the end of the buffer and will cause the `__stack_chk_fail@plt` function to notice and kill our program. However the variable  `i` will not change because its address is higher than the address of the buffer! 

In C code sample64-2 looks something like below:

```
int i;
char buffer[n];
```



But sample64 looks like below:

```
char buffer[n];
int i;
```



Therefore if we write past the end of buffer in sample64 we change the value of variable`i`, but in sample64-2 we dont.



More proof (value of i is lower than address of buffer, so buffer overflow will affect variable `i`):

```
# ./sample64   
In main(), x is stored at 0x7fffffffe69c.
In sample_function(), i is stored at 0x7fffffffe678.
In sample_function(), buffer is stored at 0x7fffffffe66e.
```

