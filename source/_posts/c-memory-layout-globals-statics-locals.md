---
title: "C 程序内存布局：全局、静态、局部变量与常量的位置"
date: 2023-01-10 00:47:29
tags:
  - "C"
  - "memory layout"
  - "ELF"
categories:
  - "Systems Programming"
---

<!--more-->

##  Development tools
- macOS(Apple M1)
- VScode
- otool
- binutils

## Let's start with a simple program

```c
#include <unistd.h>
int global_init_a = 1;
char global_init_b = 'a';

int global_unin_a;
char global_unin_b;

int main(int argc, const char* argv[]) {
    static int local_stat_init_a = 2;
    static char local_stat_init_b = 'b';

    static int local_stat_unin_a;
    static char local_stat_unin_b;

    sleep(-1);
    return 0;
}
```
After compiling it, type:

> otool -l a.out
>
Outputs is:

```c
Load command 0
      cmd LC_SEGMENT_64
  cmdsize 72
  segname __PAGEZERO
   vmaddr 0x0000000000000000
   vmsize 0x0000000100000000
  fileoff 0
 filesize 0
  maxprot 0x00000000
 initprot 0x00000000
   nsects 0
    flags 0x0
Load command 1
      cmd LC_SEGMENT_64
  cmdsize 392
  segname __TEXT
   vmaddr 0x0000000100000000
   vmsize 0x0000000000004000
  fileoff 0
 filesize 16384
  maxprot 0x00000005
 initprot 0x00000005
   nsects 4
    flags 0x0
Section
  sectname __text
   segname __TEXT
      addr 0x0000000100003f4c
      size 0x000000000000003c
    offset 16204
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x80000400
 reserved1 0
 reserved2 0
Section
  sectname __stubs
   segname __TEXT
      addr 0x0000000100003f88
      size 0x000000000000000c
    offset 16264
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x80000408
 reserved1 0 (index into indirect symbol table)
 reserved2 12 (size of stubs)
Section
  sectname __stub_helper
   segname __TEXT
      addr 0x0000000100003f94
      size 0x0000000000000024
    offset 16276
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x80000400
 reserved1 0
 reserved2 0
Section
  sectname __unwind_info
   segname __TEXT
      addr 0x0000000100003fb8
      size 0x0000000000000048
    offset 16312
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
Load command 2
      cmd LC_SEGMENT_64
  cmdsize 152
  segname __DATA_CONST
   vmaddr 0x0000000100004000
   vmsize 0x0000000000004000
  fileoff 16384
 filesize 16384
  maxprot 0x00000003
 initprot 0x00000003
   nsects 1
    flags 0x10
Section
  sectname __got
   segname __DATA_CONST
      addr 0x0000000100004000
      size 0x0000000000000008
    offset 16384
     align 2^3 (8)
    reloff 0
    nreloc 0
     flags 0x00000006
 reserved1 1 (index into indirect symbol table)
 reserved2 0
Load command 3
      cmd LC_SEGMENT_64
  cmdsize 392
  segname __DATA
   vmaddr 0x0000000100008000
   vmsize 0x0000000000004000
  fileoff 32768
 filesize 16384
  maxprot 0x00000003
 initprot 0x00000003
   nsects 4
    flags 0x0
Section
  sectname __la_symbol_ptr
   segname __DATA
      addr 0x0000000100008000
      size 0x0000000000000008
    offset 32768
     align 2^3 (8)
    reloff 0
    nreloc 0
     flags 0x00000007
 reserved1 2 (index into indirect symbol table)
 reserved2 0
Section
  sectname __data
   segname __DATA
      addr 0x0000000100008008
      size 0x0000000000000015
    offset 32776
     align 2^3 (8)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
Section
  sectname __bss
   segname __DATA
      addr 0x0000000100008020
      size 0x0000000000000005
    offset 0
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x00000001
 reserved1 0
 reserved2 0
Section
  sectname __common
   segname __DATA
      addr 0x0000000100008028
      size 0x0000000000000005
    offset 0
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x00000001
 reserved1 0
 reserved2 0
Load command 4
      cmd LC_SEGMENT_64
  cmdsize 72
  segname __LINKEDIT
   vmaddr 0x000000010000c000
   vmsize 0x0000000000004000
  fileoff 49152
 filesize 1154
  maxprot 0x00000001
 initprot 0x00000001
   nsects 0
    flags 0x0
Load command 5
            cmd LC_DYLD_INFO_ONLY
        cmdsize 48
     rebase_off 49152
    rebase_size 8
       bind_off 49160
      bind_size 24
  weak_bind_off 0
 weak_bind_size 0
  lazy_bind_off 49184
 lazy_bind_size 16
     export_off 49200
    export_size 112
Load command 6
     cmd LC_SYMTAB
 cmdsize 24
  symoff 49320
   nsyms 13
  stroff 49544
 strsize 224
Load command 7
            cmd LC_DYSYMTAB
        cmdsize 80
      ilocalsym 0
      nlocalsym 5
     iextdefsym 5
     nextdefsym 6
      iundefsym 11
      nundefsym 2
         tocoff 0
           ntoc 0
      modtaboff 0
        nmodtab 0
   extrefsymoff 0
    nextrefsyms 0
 indirectsymoff 49528
  nindirectsyms 3
      extreloff 0
        nextrel 0
      locreloff 0
        nlocrel 0
Load command 8
          cmd LC_LOAD_DYLINKER
      cmdsize 32
         name /usr/lib/dyld (offset 12)
Load command 9
     cmd LC_UUID
 cmdsize 24
    uuid 100537C2-A683-360E-A421-D1ED64F448DE
Load command 10
      cmd LC_BUILD_VERSION
  cmdsize 32
 platform 1
    minos 11.3
      sdk 11.3
   ntools 1
     tool 3
  version 650.9
Load command 11
      cmd LC_SOURCE_VERSION
  cmdsize 16
  version 0.0
Load command 12
       cmd LC_MAIN
   cmdsize 24
  entryoff 16204
 stacksize 0
Load command 13
          cmd LC_LOAD_DYLIB
      cmdsize 56
         name /usr/lib/libSystem.B.dylib (offset 24)
   time stamp 2 Thu Jan  1 08:00:02 1970
      current version 1292.100.5
compatibility version 1.0.0
Load command 14
      cmd LC_FUNCTION_STARTS
  cmdsize 16
  dataoff 49312
 datasize 8
Load command 15
      cmd LC_DATA_IN_CODE
  cmdsize 16
  dataoff 49320
 datasize 0
Load command 16
      cmd LC_CODE_SIGNATURE
  cmdsize 16
  dataoff 49776
 datasize 530
```
Let's take out the segment information we need to see:

```c
Section
  sectname __text
   segname __TEXT
      addr 0x0000000100003f4c
      size 0x000000000000003c
    offset 16204
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x80000400
 reserved1 0
 reserved2 0
 Section
  sectname __data
   segname __DATA
      addr 0x0000000100008008
      size 0x0000000000000015
    offset 32776
     align 2^3 (8)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
 Section
  sectname __bss
   segname __DATA
      addr 0x0000000100008020
      size 0x0000000000000005
    offset 0
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x00000001
 reserved1 0
 reserved2 0
Section
  sectname __common
   segname __DATA
      addr 0x0000000100008028
      size 0x0000000000000005
    offset 0
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x00000001
 reserved1 0
 reserved2 0
```


Look at its Symbol Table, type:
> nm a.out

The output is:

```c
0000000100008008 d __dyld_private
0000000100000000 T __mh_execute_header
0000000100008010 D _global_init_a
0000000100008014 D _global_init_b
0000000100008028 S _global_unin_a
000000010000802c S _global_unin_b
0000000100003f4c T _main
0000000100008018 d _main.local_stat_init_a
000000010000801c d _main.local_stat_init_b
0000000100008020 b _main.local_stat_unin_a
0000000100008024 b _main.local_stat_unin_b
                 U _sleep
                 U dyld_stub_binder
```

Conclusion:
>- Comparing the addr in the information given above, it can be judged that the initialized global variables and local static variables are in the __data segment, while the uninitialized global variables are in the __common segment, and the uninitialized local static variables are in the _ _bss section.

We can modify this program to initialize uninitialized variables to 0:

```c
#include <unistd.h>
int global_init_a = 1;
char global_init_b = 'a';

int global_unin_a = 0;
char global_unin_b = 0;

int main(int argc, const char* argv[]) {
    static int local_stat_init_a = 2;
    static char local_stat_init_b = 'b';

    static int local_stat_unin_a = 0;
    static char local_stat_unin_b = 0;

    sleep(-1);
    return 0;
}
```
Check the symbol table after compiling: (for convenience, we use the `objdump -t` command to view)

```c

SYMBOL TABLE:
0000000100008008 l     O __DATA,__data __dyld_private
0000000100008018 l     O __DATA,__data _main.local_stat_init_a
000000010000801c l     O __DATA,__data _main.local_stat_init_b
0000000100008028 l     O __DATA,__bss _main.local_stat_unin_a
000000010000802c l     O __DATA,__bss _main.local_stat_unin_b
0000000100000000 g     F __TEXT,__text __mh_execute_header
0000000100008010 g     O __DATA,__data _global_init_a
0000000100008014 g     O __DATA,__data _global_init_b
0000000100008020 g     O __DATA,__common _global_unin_a
0000000100008024 g     O __DATA,__common _global_unin_b
0000000100003f4c g     F __TEXT,__text _main
0000000000000000         *UND* _sleep
0000000000000000         *UND* dyld_stub_binder
```

Surprisingly, initializing both variables to 0 is equivalent to no initialization. **So we generally do not need to initialize global variables or local static variables to 0, because it is unnecessary. **

##  What is the difference between the __common section and the __bss section?
Observing the address information above, it can be found that the __commond segment and the __bss segment are connected together.
__common section: The __common section stores weakly referenced symbols, that is, uninitialized global variables, and then puts them into the __bss section when linking, and then they can be easily merged together~
__bss segment: The abbreviation of Block Started by Symbol, a memory area that stores uninitialized global variables and global variables initialized to 0 in the program. The BSS segment is a static memory allocation.



## Doesn't the bss section occupy the executable file space, why?

We can take a look directly to find out. Using the command `otool -d`, you can see:

> (__DATA,__data) section
0000000100008008        00000000 00000000 00000001 00000061
0000000100008018        00000002 62


It can be seen from **SYMBOL TABLE** that there are a total of 5 values in the (__DATA,__data) section, which are: 0,1,61,2,62.
- 0 is __dyld_private, skip this for now;
- 1 is global_init_a;
- 61 is global_init_b('a');
- 2 is local_stat_init_a;
- 62 is local_stat_init_b = 'b';
This means that the __data segment holds the values ​​of initialized global variables and local static variables.

Let's take a look at the information in the `__bss` section, use the command `objdump -s` command, see:

```c
Contents of section __common:
<skipping contents of bss section at [100008020, 100008025)>
Contents of section __bss:
<skipping contents of bss section at [100008028, 10000802d)>
```

It turned out that there was nothing inside. This is actually not difficult to understand. Stored in the __bss segment are uninitialized global variables and local static variables. Since there is no initialization, there is no need to record the value of the variable in the executable file (there is no value to record). The only thing needed is to give These variables have a certain memory address (like variables in the __data section). In this way, there are actually two methods: first, write some initial values in the corresponding position like the __data section to occupy the place, and the executable file can be directly mapped when it is loaded, which is exactly the same as the __data section; second, do not give __bss The segment occupies a place in the executable file, and the corresponding area is directly opened in the memory according to the __bss segment information when loading, that is, the place is postponed from the executable file to the time of loading. The compiler is the choice of the second method, which postpones the placeholder from the executable file to the loading time. The advantage of this is to reduce the size of the executable file. takes up any executable space, while approach one increases the executable size by at least 40KB.
But strictly speaking, the bss segment still occupies some executable file space. For example, there is a description of the bss segment in the segment table, and a description of related variables in the bss segment in the symbol table, but here is not the same concept.
## How is the size of the bss segment and the data segment calculated?
Simple addition? Let's see. Type `objdump -h` to see:

```c
Sections:
Idx Name            Size     VMA              Type
  0 __text          0000003c 0000000100003f4c TEXT
  1 __stubs         0000000c 0000000100003f88 TEXT
  2 __stub_helper   00000024 0000000100003f94 TEXT
  3 __unwind_info   00000048 0000000100003fb8 DATA
  4 __got           00000008 0000000100004000 DATA
  5 __la_symbol_ptr 00000008 0000000100008000 DATA
  6 __data          00000015 0000000100008008 DATA
  7 __common        00000005 0000000100008020 BSS
  8 __bss           00000005 0000000100008028 BSS
```
It can be seen that the size of the __data section is 15, but we know that there are only 5 variables in the data section. combine
```c

SYMBOL TABLE:
0000000100008008 l     O __DATA,__data __dyld_private
0000000100008018 l     O __DATA,__data _main.local_stat_init_a
000000010000801c l     O __DATA,__data _main.local_stat_init_b
0000000100008028 l     O __DATA,__bss _main.local_stat_unin_a
000000010000802c l     O __DATA,__bss _main.local_stat_unin_b
0000000100000000 g     F __TEXT,__text __mh_execute_header
0000000100008010 g     O __DATA,__data _global_init_a
0000000100008014 g     O __DATA,__data _global_init_b
0000000100008020 g     O __DATA,__common _global_unin_a
0000000100008024 g     O __DATA,__common _global_unin_b
0000000100003f4c g     F __TEXT,__text _main
0000000000000000         *UND* _sleep
0000000000000000         *UND* dyld_stub_binder
```

 In this way, the total is exactly 15 bytes.However, if we count as 4+4+1+1, it is 10 bytes. why? It turns out that there is an interval of 3 bytes between global_init_b and local_stat_init_a, which is a common "alignment" process in memory. Therefore, the size of the data segment or the bss segment is not necessarily the sum of the sizes of its internal variables, and is generally greater than or equal to the sum of the sizes of its internal variables. This is caused by memory alignment.


## Where are local variables stored?
Let's change the code to:

```c
#include <unistd.h>
// int global_init_a = 1;
// char global_init_b = 'a';

// int global_unin_a = 0;
// char global_unin_b = 0;

int main(int argc, const char* argv[]) {
    // static int local_stat_init_a = 2;
    // static char local_stat_init_b = 'b';

    int local_init_a = 3;
    char local_init_b = 'c';

    int local_unin_a;
    char local_unin_b;

    // static int local_stat_unin_a = 0;
    // static char local_stat_unin_b = 0;

    // sleep(-1);
    return 0;
}
```
After compiling, type `otool -tv` to disassemble, you can see:

```c
_main:
0000000100003f88        sub     sp, sp, #0x20
0000000100003f8c        mov     w8, #0x0
0000000100003f90        str     wzr, [sp, #0x1c]
0000000100003f94        str     w0, [sp, #0x18]
0000000100003f98        str     x1, [sp, #0x10]
0000000100003f9c        mov     w9, #0x3
0000000100003fa0        str     w9, [sp, #0xc]
0000000100003fa4        mov     w9, #0x63
0000000100003fa8        strb    w9, [sp, #0xb]
0000000100003fac        mov     x0, x8
0000000100003fb0        add     sp, sp, #0x20
0000000100003fb4        ret
```
`sp` is the pointer to the top of the stack, we mainly focus on these lines:

```c
0000000100003f9c        mov     w9, #0x3
0000000100003fa0        str     w9, [sp, #0xc]
0000000100003fa4        mov     w9, #0x63
0000000100003fa8        strb    w9, [sp, #0xb]
```
This is actually just these two lines of code:

```c
	int local_init_a = 3;
    char local_init_b = 'c';
```


It can be seen that sp is getting smaller and smaller. We know that the stack grows toward small memory addresses. According to the disassembly code, it is not difficult to see that local variables are stored in the stack.

## What are the default values of global variables, static variables and local variables?
The answer to this question was known when I first started learning C language: the default value of global variables and static variables is 0, and the default value of local variables is uncertain. Here's a closer look at why this is the case.

Both global variables and local static variables are stored in the __bss segment and the __common segment. The default value only needs to be analyzed in units of segments. Here we take the __bss segment as an example.
The information of __bss is as follows:

```c
Section
  sectname __bss
   segname __DATA
      addr 0x0000000100001028
      size 0x0000000000000005
    offset 0
     align 2^2 (4)
    reloff 0
    nreloc 0
      type S_ZEROFILL
```

Note that type S_ZEROFILL is to require all the contents of this section to be filled with 0. Why is the default value of global variables and static variables 0, while local variables are undefined?
It's actually easy to understand. Global variables and static variables are stored in the __bss segment and __common segment. The memory address is known when loading, and the overhead of filling 0 is not too large, and global variables and static variables act on the entire program life cycle, and it is also valuable to initialize them of. In contrast to local variables, there are a large number of local variables with a short life cycle. They are stored on the stack and their memory addresses cannot be determined in advance. If the local variables are filled with 0 and initialized every time, it will not only consume resources, but also have small benefits, which is not worth the candle.
## Will variables modified by const have different locations in memory?
In order to study this problem, we modify the code again:

```c
#include <unistd.h>

int global_init_a = 1;
char global_init_b = 'a';

const int global_con_init_a = 2;
const char global_con_init_b = 'b';

int global_unin_a;
char global_unin_b;

const int global_con_unin_a;
const char global_con_unin_b;

int main(int argc, const char *argv[]) {
  static int local_stat_init_a = 3;
  static char local_stat_init_b = 'c';

  const static int local_con_stat_init_a = 4;
  const static char local_con_stat_init_b = 'd';

  static int local_stat_unin_a;
  static char local_stat_unin_b;

  const static int local_con_stat_unin_a;
  const static char local_con_stat_unin_b;

  int local_init_a = 5;
  char local_init_b = 'e';

  const int local_con_init_a = 6;
  const char local_con_init_b = 'f';

  int local_unin_a;
  char local_unin_b;

  const int local_con_unin_a;
  const char local_con_unin_b;

  sleep(-1);
  return 0;
}
```
The section information is as follows:

```c
Section
  sectname __text
   segname __TEXT
      addr 0x0000000100003f14
      size 0x000000000000005c
    offset 16148
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x80000400
 reserved1 0
 reserved2 0
 Section
  sectname __const
   segname __TEXT
      addr 0x0000000100003fa0
      size 0x0000000000000015
    offset 16288
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
 Section
  sectname __data
   segname __DATA
      addr 0x0000000100008008
      size 0x0000000000000015
    offset 32776
     align 2^3 (8)
    reloff 0
    nreloc 0
     flags 0x00000000
 reserved1 0
 reserved2 0
Section
  sectname __bss
   segname __DATA
      addr 0x0000000100008020
      size 0x0000000000000005
    offset 0
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x00000001
 reserved1 0
 reserved2 0
Section
  sectname __common
   segname __DATA
      addr 0x0000000100008028
      size 0x000000000000000d
    offset 0
     align 2^2 (4)
    reloff 0
    nreloc 0
     flags 0x00000001
 reserved1 0
 reserved2 0
```
Look at the corresponding symbol table again:

```c
SYMBOL TABLE:
0000000100003fa8 l     O __TEXT,__const _main.local_con_stat_init_a
0000000100003fac l     O __TEXT,__const _main.local_con_stat_init_b
0000000100003fb0 l     O __TEXT,__const _main.local_con_stat_unin_a
0000000100003fb4 l     O __TEXT,__const _main.local_con_stat_unin_b
0000000100008008 l     O __DATA,__data __dyld_private
0000000100008018 l     O __DATA,__data _main.local_stat_init_a
000000010000801c l     O __DATA,__data _main.local_stat_init_b
0000000100008020 l     O __DATA,__bss _main.local_stat_unin_a
0000000100008024 l     O __DATA,__bss _main.local_stat_unin_b
0000000100000000 g     F __TEXT,__text __mh_execute_header
0000000100003fa0 g     O __TEXT,__const _global_con_init_a
0000000100003fa4 g     O __TEXT,__const _global_con_init_b
0000000100008028 g     O __DATA,__common _global_con_unin_a
000000010000802c g     O __DATA,__common _global_con_unin_b
0000000100008010 g     O __DATA,__data _global_init_a
0000000100008014 g     O __DATA,__data _global_init_b
0000000100008030 g     O __DATA,__common _global_unin_a
0000000100008034 g     O __DATA,__common _global_unin_b
0000000100003f14 g     F __TEXT,__text _main
0000000000000000         *UND* _sleep
0000000000000000         *UND* dyld_stub_binder
```
Finally, look at the contents of the __const section:

```c
Contents of section __const:
 100003fa0 02000000 62000000 04000000 64000000  ....b.......d...
 100003fb0 00000000 00
```

It can be found that all global variables and local static variables decorated with const are placed in the read-only __const segment. Compared with global variables and local static variables without const modification, there are mainly the following differences:

- Exposed (external) global variables and internal (non-external) local static variables are in the __const segment;
- Both initialized and uninitialized variables are in the __const section;
- The __const section does not have the feature of saving executable file space like the __bss section.

That is, both uninitialized global variables and local static variables take up executable space.

Let's take a look at the situation of local variables again, disassembly:

```c
(__TEXT,__text) section
_main:
0000000100003f14        sub     sp, sp, #0x50
0000000100003f18        stp     x29, x30, [sp, #0x40]
0000000100003f1c        add     x29, sp, #0x40
0000000100003f20        mov     w8, #0x0
0000000100003f24        stur    wzr, [x29, #-0x4]
0000000100003f28        stur    w0, [x29, #-0x8]
0000000100003f2c        stur    x1, [x29, #-0x10]
0000000100003f30        mov     w9, #0x5
0000000100003f34        stur    w9, [x29, #-0x14]
0000000100003f38        mov     w9, #0x65
0000000100003f3c        sturb   w9, [x29, #-0x15]
0000000100003f40        mov     w9, #0x6
0000000100003f44        stur    w9, [x29, #-0x1c]
0000000100003f48        mov     w9, #0x66
0000000100003f4c        sturb   w9, [x29, #-0x1d]
0000000100003f50        mov     w0, #-0x1
0000000100003f54        str     w8, [sp, #0xc]
0000000100003f58        bl      0x100003f70 ; symbol stub for: _sleep
0000000100003f5c        ldr     w8, [sp, #0xc]
0000000100003f60        mov     x0, x8
0000000100003f64        ldp     x29, x30, [sp, #0x40]
0000000100003f68        add     sp, sp, #0x50
0000000100003f6c        ret
```
What we need to pay attention to are these lines:

```c
0000000100003f30        mov     w9, #0x5
0000000100003f34        stur    w9, [x29, #-0x14]
0000000100003f38        mov     w9, #0x65
0000000100003f3c        sturb   w9, [x29, #-0x15]
0000000100003f40        mov     w9, #0x6
0000000100003f44        stur    w9, [x29, #-0x1c]
0000000100003f48        mov     w9, #0x66
0000000100003f4c        sturb   w9, [x29, #-0x1d]
```
According to the method analyzed above, it can be explained that for the initialized local variables, the presence or absence of const modification has no effect on the location of the variables, and they are all in the stack in order.
Conclusions:
- Global variables and local static variables modified by const will be placed in the read-only __const segment; for local variables, whether there is const modification has no effect on the location of the variable, and they are all on the stack.

Note that there is a problem here: the read-only __const segment is protected by the operating system, and its internal value cannot be modified; but there is no protection mechanism for local variables modified by const (because they are placed on the stack like ordinary local variables), at this time C language If you want to realize the function of const, you can only intervene in the compilation process to achieve constantization. This question is left for later consideration.

## Afterwords
Good tools can help us get to the bottom of a problem, and this article makes use of some great tools. Of course, we must not only have one tool. In this article, we used `nm`, `otool`, `objdump` to successfully analyze various parts of the C programs.

*This essay is over here, thank you for reading.*

















