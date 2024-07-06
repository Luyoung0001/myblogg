---
title: debugger（三）：dwarf 文件
date: 2024-06-10 21:46:30
tags: c++ debugger
categories: C++ debugger
---

<!--more-->

## 〇、前言
事实上，一个成熟的 **debugger** 是不会利用 `break 0xADDR` 类似的命令来打断点的，这个需要改进，使得它可以直接利用**函数名、行数**等来打断点。这就需要生成编译信息，只需要在编译的时候，在目标文件中加以下参数：

```bash
# 添加编译器标志
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O0 -gdwarf-2")
```

这样，目标文件就携带了 dwarf 格式的 debug info（我们还禁止了优化，这有利于调试）。

## 一、ELF & DWARF
### DWARF 和 ELF 的关系与区别

**DWARF**：
- DWARF 是一种关于调试信息的标准格式，用于在编译时生成的调试信息中描述程序的各种数据结构。**这包括但不限于变量的名称、类型、存储位置，函数的名称、参数列表和源代码中的行号**。
- DWARF 是与平台无关的，这意味着它可以用在各种不同的操作系统和硬件上。

**ELF (Executable and Linkable Format)**：
- ELF 是一种常用的文件格式，用于定义在类 Unix 系统（如 Linux）上运行的可执行文件、可重定位的代码和共享库。
- ELF 文件包含程序的代码和数据，并定义了一个文件结构，这个结构描述了如何在运行时将程序加载到内存中。

**关系与区别**：
- DWARF 信息通常被嵌入到 ELF 文件中（在 `.debug` 节），作为程序的一部分。这意味着 ELF 文件作为容器，包含了执行程序所需的机器代码和（如果编译时指定）调试信息。
- 在调试过程中，**调试器利用 ELF 文件中的 DWARF 信息来提供程序执行的详细视图，比如变量值、程序执行的当前行等**。
- 虽然 DWARF 和 ELF 通常一起使用，**但它们是独立的标准：DWARF 关注于描述调试数据，ELF 关注于程序的布局和执行**。

## 二、DWARF line table & DWARF debug info

在讨论 DWARF 格式的调试信息时，编译单元（Compilation Unit, CU）和行表（Line Table）是两个核心概念。这些信息极大地促进了源码级调试，使调试器能够有效地将执行的机器代码映射回源代码。下面详细介绍这两个概念：

### 编译单元（Compilation Unit, CU）
编译单元通常指的是单个源文件及其相关包含的文件（通过预处理器展开）在编译过程中形成的单元。在 DWARF 调试信息中，每个编译单元生成一组特定的调试信息记录，这些记录描述了该源文件中定义的数据结构、函数、变量、类型等。

#### 编译单元的主要内容包括：
1. **全局变量和类型定义**：全局作用域中定义的变量和类型。
2. **局部变量和类型定义**：函数内部定义的变量和类型。
3. **子程序信息**：包括函数和方法的定义，如函数名、返回类型、参数信息以及函数内的代码结构。
4. **源文件和目录信息**：描述编译单元对应的源文件和其在文件系统中的位置。

编译单元的信息对于调试器来说至关重要，因为它们提供了代码结构的详细视图，使得调试器可以准确地知道在任何时刻程序正在执行的代码部分。
### 行表（Line Table）
行表是 DWARF 调试信息中的一部分，它为编译单元中的每一行源代码提供一个或多个对应的机器指令的映射。这允许调试器将正在执行的机器指令精确地对应到源代码中的具体行。

#### 行表的关键作用：
1. **源代码到机器代码的映射**：行表记录了源代码行与生成的机器代码之间的对应关系。这包括代码地址的起始点和源代码行号。
2. **断点设置**：当在源代码中设置断点时，调试器使用行表来确定应该在哪个具体的机器指令地址上设置断点。
3. **步进和步过操作**：在单步执行（步进）和执行至下一行（步过）时，调试器利用行表来确定执行流程应该停留或跳过的代码段。

行表通常包含以下信息：
- **地址**：对应机器代码的开始地址。
- **行号**：源代码中的行号。
- **文件名**：源代码的文件名，尤其是在项目中包含多个文件时。
- **其他标志**：如是否是语句的开始、是否是基本块的开始等。


而 CU 就是 debug info 的基本组成，每一个 CU（事实上就是一个源代码文件）组成 debug info段的一部分。编译单元是调试信息的构建块，每个编译单元封装了一个源文件的所有相关调试信息。这种组织方式不仅有助于维护信息的结构性和可查询性，也使得调试过程更为高效和直观。

假设一个可执行程序（包含了 DWARF 格式的调试信息）包含了很多的源代码文件，它的 debug_info 可能如下：

```bash
objdump --dwarf=info hello
hello:     file format elf64-x86-64

Contents of the .debug_info section:

  Compilation Unit @ offset 0x0:
   Length:        0x2612 (32-bit)
   Version:       2
   Abbrev Offset: 0x0
   Pointer Size:  8
 <0><b>: Abbrev Number: 1 (DW_TAG_compile_unit)
    <c>   DW_AT_producer    : (indirect string, offset: 0x109b): GNU C++17 11.4.0 -mtune=generic -march=x86-64 -g -gdwarf-2 -O0 -fasynchronous-unwind-tables -fstack-protector-strong -fstack-clash-protection -fcf-protection
    <10>   DW_AT_language    : 4	(C++)
    <11>   DW_AT_name        : (indirect string, offset: 0x4ba): /home/luyoung/mydebugger/examples/hello.cpp
    <15>   DW_AT_comp_dir    : (indirect string, offset: 0x154): /home/luyoung/mydebugger/build/examples
    <19>   DW_AT_low_pc      : 0x1189
    <21>   DW_AT_high_pc     : 0x1220
    <29>   DW_AT_stmt_list   : 0x0
 <1><2d>: Abbrev Number: 2 (DW_TAG_namespace)
    <2e>   DW_AT_name        : std
    <32>   DW_AT_decl_file   : 6
    <33>   DW_AT_decl_line   : 278
    <35>   DW_AT_decl_column : 11
    <36>   DW_AT_sibling     : <0xbab>
 <2><3a>: Abbrev Number: 3 (DW_TAG_namespace)
    <3b>   DW_AT_name        : (indirect string, offset: 0x859): __cxx11
    <3f>   DW_AT_decl_file   : 6
    <40>   DW_AT_decl_line   : 302
    <42>   DW_AT_decl_column : 65
    <43>   DW_AT_export_symbols: 1
 <2><44>: Abbrev Number: 4 (DW_TAG_imported_module)
 .
 .
 .
  <2><25f2>: Abbrev Number: 0
 <1><25f3>: Abbrev Number: 88 (DW_TAG_subprogram)
    <25f4>   DW_AT_external    : 1
    <25f5>   DW_AT_name        : (indirect string, offset: 0x6b3): main
    <25f9>   DW_AT_decl_file   : 1
    <25fa>   DW_AT_decl_line   : 2
    <25fb>   DW_AT_decl_column : 5
    <25fc>   DW_AT_type        : <0xd3d>
    <2600>   DW_AT_low_pc      : 0x1189
    <2608>   DW_AT_high_pc     : 0x11b1
    <2610>   DW_AT_frame_base  : 0xc0 (location list)
    <2614>   DW_AT_GNU_all_tail_call_sites: 1
 <1><2615>: Abbrev Number: 0
```

这些信息很难看，非常不利于人类阅读，因此我们可以利用更好的工具来理解这些信息，这些工具对这些信息进行了组织，比如 dwarfdump：

```bash

.debug_info

COMPILE_UNIT<header overall offset = 0x00000000>:
< 0><0x0000000b>  DW_TAG_compile_unit
                    DW_AT_producer              GNU C++17 11.4.0 -mtune=generic -march=x86-64 -g -gdwarf-2 -O0 -fasynchronous-unwind-tables -fstack-protector-strong -fstack-clash-protection -fcf-protection
                    DW_AT_language              DW_LANG_C_plus_plus
                    DW_AT_name                  /home/luyoung/mydebugger/examples/hello.cpp
                    DW_AT_comp_dir              /home/luyoung/mydebugger/build/examples
                    DW_AT_low_pc                0x00001189
                    DW_AT_high_pc               0x00001220
                    DW_AT_stmt_list             0x00000000

LOCAL_SYMBOLS:
< 1><0x0000002d>    DW_TAG_namespace
                      DW_AT_name                  std
                      DW_AT_decl_file             0x00000006 /usr/include/x86_64-linux-gnu/c++/11/bits/c++config.h
                      DW_AT_decl_line             0x00000116
                      DW_AT_decl_column           0x0000000b
                      DW_AT_sibling               <0x00000bab>
< 2><0x0000003a>      DW_TAG_namespace
.
.
.
.debug_line: line number info for a single cu
Source lines (from CU-DIE at .debug_info offset 0x0000000b):

            NS new statement, BB new basic block, ET end of text sequence
            PE prologue end, EB epilogue begin
            IS=val ISA number, DI=val discriminator value
<pc>        [lno,col] NS BB ET PE EB IS= DI= uri: "filepath"
0x00001189  [   2,12] NS uri: "/home/luyoung/mydebugger/examples/hello.cpp"
0x00001191  [   3,16] NS
0x000011aa  [   4,10] NS
0x000011af  [   5, 1] NS
0x000011b1  [   5, 1] NS
0x000011c3  [   5, 1] NS
0x000011c9  [   5, 1] DI=0x1
0x000011d2  [  74,25] NS uri: "/usr/include/c++/11/iostream"
0x00001204  [   5, 1] NS uri: "/home/luyoung/mydebugger/examples/hello.cpp"
0x00001207  [   5, 1] NS
0x0000120f  [   5, 1] NS
0x00001220  [   5, 1] NS ET
.
.
.
.debug_str
name at offset 0x00000000, length    6 is 'getenv'
name at offset 0x00000007, length   16 is '__isoc99_vwscanf'
name at offset 0x00000018, length   13 is 'uint_fast16_t'
name at offset 0x00000026, length    7 is '__debug'
name at offset 0x0000002e, length   17 is 'int_p_cs_precedes'
name at offset 0x00000040, length   42 is '_ZNSt15__exception_ptr13exception_ptrC4EPv'
name at offset 0x0000006b, length    8 is 'strtoull'
name at offset 0x00000074, length   16 is '__uint_least64_t'
name at offset 0x00000085, length    7 is 'wcsxfrm'
name at offset 0x0000008d, length   51 is '_ZNSt15__exception_ptr13exception_ptr10_M_releaseEv'
name at offset 0x000000c1, length   14 is '~exception_ptr'
name at offset 0x000000d0, length    4 is 'atol'
name at offset 0x000000d5, length    9 is '_shortbuf'
name at offset 0x000000df, length   10 is '_IO_lock_t'
name at offset 0x000000ea, length    7 is 'setvbuf'
name at offset 0x000000f2, length    9 is 'gp_offset'
.
.
.
.debug_aranges

COMPILE_UNIT<header overall offset = 0x00000000>:
< 0><0x0000000b>  DW_TAG_compile_unit
                    DW_AT_producer              GNU C++17 11.4.0 -mtune=generic -march=x86-64 -g -gdwarf-2 -O0 -fasynchronous-unwind-tables -fstack-protector-strong -fstack-clash-protection -fcf-protection
                    DW_AT_language              DW_LANG_C_plus_plus
                    DW_AT_name                  /home/luyoung/mydebugger/examples/hello.cpp
                    DW_AT_comp_dir              /home/luyoung/mydebugger/build/examples
                    DW_AT_low_pc                0x00001189
                    DW_AT_high_pc               0x00001220
                    DW_AT_stmt_list             0x00000000


arange starts at 0x00001189, length of 0x00000097, cu_die_offset = 0x0000000b
arange end

.debug_frame is not present
```

可以看到，**dwarfdump** 输出的信息更好理解，它对信息进行了分类整理。还必须要理解的是，这里的 **pc** 地址都是 **offset**，在使用的时候需要加上 **load_addr**，另外：

```bash
<pc>        [lno,col] NS BB ET PE EB IS= DI= uri: "filepath"
0x00001189  [   2,12] NS uri: "/home/luyoung/mydebugger/examples/hello.cpp"
0x00001191  [   3,16] NS
0x000011aa  [   4,10] NS
0x000011af  [   5, 1] NS
0x000011b1  [   5, 1] NS
0x000011c3  [   5, 1] NS
0x000011c9  [   5, 1] DI=0x1
0x000011d2  [  74,25] NS uri: "/usr/include/c++/11/iostream"
0x00001204  [   5, 1] NS uri: "/home/luyoung/mydebugger/examples/hello.cpp"
0x00001207  [   5, 1] NS
0x0000120f  [   5, 1] NS
0x00001220  [   5, 1] NS ET
```
和源代码相对应，这为源代码 level 调试提供了基础：

```cpp
#1 #include <iostream>
#2 int main() {
#3  std::cerr << "hello,world0.\n";
#4  return 0;
#5}
```

如果我们想在源代码第三行处打断点，就应该把地址定在 `0x00001191`，这也可以在 elf 中找到依据：

```bash
0000000000001189 <main>:
    1189:       f3 0f 1e fa             endbr64 
    118d:       55                      push   %rbp
    118e:       48 89 e5                mov    %rsp,%rbp
    1191:       48 8d 05 6c 0e 00 00    lea    0xe6c(%rip),%rax        # 2004 <_IO_stdin_used+0x4>
    1198:       48 89 c6                mov    %rax,%rsi
    119b:       48 8d 05 7e 2e 00 00    lea    0x2e7e(%rip),%rax        # 4020 <_ZSt4cerr@GLIBCXX_3.4>
    11a2:       48 89 c7                mov    %rax,%rdi
    11a5:       e8 d6 fe ff ff          call   1080 <_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@plt>
    11aa:       b8 00 00 00 00          mov    $0x0,%eax
    11af:       5d                      pop    %rbp
    11b0:       c3               
```

另外还可以看到，编译单元中的：

```bash
COMPILE_UNIT<header overall offset = 0x00000000>:
< 0><0x0000000b>  DW_TAG_compile_unit
...
                    DW_AT_low_pc                0x00001189
                    DW_AT_high_pc               0x00001220
                    DW_AT_stmt_list             0x00000000
                    ...
```

它们的 **DW_AT_low_pc**和 **DW_AT_high_pc** 和 line table 中的范围一致，因此这也是重要的信息。








