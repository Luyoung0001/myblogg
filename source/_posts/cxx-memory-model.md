---
title: "C++ 内存模型笔记：从 C 对象布局到语言抽象"
date: 2024-05-20 13:01:08
tags:
  - "C++"
  - "memory model"
categories:
  - "Programming Languages"
---

<!--more-->

## 〇、前言
本文将会讨论：Linux 下 C、C++的内存模型。

## 一、C
以下是一个示例，通过打印地址的值以及借助 `nm` 工具，来判断内存区域：
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int A;            // 全局未初始化的变量
int B = 0;        // 全局已初始化为0的变量
int C = 2;        // 全局初始化变量
static int D;     // 全局静态未初始化变量
static int E = 0; // 全局静态初始化为0变量
static int F = 4; // 全局静态已初始化变量
const int G = 5;  // 全局常量
const char H = 6;

int main(void) {
    int a;        // 局部未初始化变量
    int b = 0;    // 局部已初始化变量
    int c = 2;    // 局部初始化变量
    static int d; // 局部静态未初始化变量
    static int e = 0;
    static int f = 4; // 局部静态已初始化变量
    const int g = 5;  // 局部静态常量

    char char1[] = "abcde"; // 局部字符数组变量
    char *cptr = "123456";  // 指向字符串常量

    int *heap = malloc(sizeof(int) * 4); // 堆

    printf("PID is %d\n\n", getpid());

    printf("Int A         A_addr = %p\n", &A);
    printf("Int B = 0     B_addr = %p\n", &B);
    printf("Int C = 2     C_addr = %p\n", &C);
    printf("Static int D  D_addr = %p\n", &D);
    printf("Static int E = 0 E_addr = %p\n", &E);
    printf("Static int F = 4 F_addr = %p\n", &F);
    printf("Const int G = 5 G_addr = %p\n", &G);
    printf("Const char H = 6 H_addr = %p\n", &H);

    printf("\n");

    printf("int a               a_addr = %p\n", &a);
    printf("int b = 0           b_addr = %p\n", &b);
    printf("int c = 2           c_addr = %p\n", &c);
    printf("static int d        d_addr = %p\n", &d);
    printf("static int e = 0    e_addr = %p\n", &e);
    printf("static int f = 4    f_addr = %p\n", &f);
    printf("const int g = 5     g_addr = %p\n", &g);

    printf("\n");

    printf("Char array char1[] = \"abcde\"\tAddress = %p\n", (void *)char1);
    printf("Address of char1[]\t\t= %p\n", (void *)&char1);
    printf("Char pointer *cptr = '123456'\tAddress = %p\n", (void *)&cptr);
    printf("Value pointed by cptr\t\t= %c\n", *cptr);
    printf("Cptr points to\t\t\t= %p\n", cptr);
    printf("Heap has space of sizeof(int)*4\tAddress = %p\n", (void *)heap);
    printf("Address of heap pointer\t\t= %p\n", (void *)&heap);

    pause(); // 程序暂停在这里直到收到信号才继续，方便观察进程地址空间

    // 分配的堆内存应该在使用完毕后释放
    free(heap);

    return 0;
}

```
运行结果：

```bash
./mainc
PID is 227353

Int A         A_addr = 0x55fd24d58020
Int B = 0     B_addr = 0x55fd24d58024
Int C = 2     C_addr = 0x55fd24d58010
Static int D  D_addr = 0x55fd24d58028
Static int E = 0 E_addr = 0x55fd24d5802c
Static int F = 4 F_addr = 0x55fd24d58014
Const int G = 5 G_addr = 0x55fd24d56008
Const char H = 6 H_addr = 0x55fd24d5600c

int a               a_addr = 0x7ffe78385890
int b = 0           b_addr = 0x7ffe78385894
int c = 2           c_addr = 0x7ffe78385898
static int d        d_addr = 0x55fd24d58030
static int e = 0    e_addr = 0x55fd24d58034
static int f = 4    f_addr = 0x55fd24d58018
const int g = 5     g_addr = 0x7ffe7838589c

Char array char1[] = "abcde"    Address = 0x7ffe783858b2
Address of char1[]              = 0x7ffe783858b2
Char pointer *cptr = '123456'   Address = 0x7ffe783858a0
Value pointed by cptr           = 1
Cptr points to                  = 0x55fd24d5600d
Heap has space of sizeof(int)*4 Address = 0x55fd259972a0
Address of heap pointer         = 0x7ffe783858a8
```
再看看符号地址：
```bash
nm -n ./mainc
...
0000000000001000 t _init
0000000000001120 T _start
0000000000001150 t deregister_tm_clones
0000000000001180 t register_tm_clones
00000000000011c0 t __do_global_dtors_aux
0000000000001200 t frame_dummy
0000000000001209 T main
00000000000014d0 T __libc_csu_init
0000000000001540 T __libc_csu_fini
0000000000001548 T _fini
0000000000002000 R _IO_stdin_used
0000000000002008 R G
000000000000200c R H
0000000000002318 r __GNU_EH_FRAME_HDR
0000000000002464 r __FRAME_END__
0000000000003d88 d __frame_dummy_init_array_entry
0000000000003d88 d __init_array_start
0000000000003d90 d __do_global_dtors_aux_fini_array_entry
0000000000003d90 d __init_array_end
0000000000003d98 d _DYNAMIC
0000000000003f88 d _GLOBAL_OFFSET_TABLE_
0000000000004000 D __data_start
0000000000004000 W data_start
0000000000004008 D __dso_handle
0000000000004010 D C
0000000000004014 d F
0000000000004018 d f.0
000000000000401c B __bss_start
000000000000401c b completed.0
000000000000401c D _edata
0000000000004020 B A
0000000000004020 D __TMC_END__
0000000000004024 B B
0000000000004028 b D
000000000000402c b E
0000000000004030 b d.2
0000000000004034 b e.1
0000000000004038 B _end
```


先从地址最低的变量开始分析:
|变量|类型|地址|区段
|:-:|:-:|:-:|:-:|
G|global const int G = 5|0x55fd24d56008|R
H|global const char H = 6|0x55fd24d5600c|R
Cptr|''123456''|0x55fd24d5600d|R
C|global int C = 2| 0x55fd24d58010|D
F|global static int F = 4|0x55fd24d58014|d
f|static int f = 4|0x55fd24d58018|d
A|global int A|0x55fd24d58020|B
B|global int B = 0 |0x55fd24d58024|B
D|global static int D|0x55fd24d58028|b
E|global static int E = 0|0x55fd24d5802c|b
d|static int d |0x55fd24d58030|b
e|static int e = 0|0x55fd24d58034|b


### 变量分析

1. **G** (global const int G = 5)
   - **类型**: `R` (只读数据段)
   - **原因**: 因为 `G` 是一个全局的 `const` 变量，其值在编译时就已确定，且不可变，因此放在只读数据段。

2. **H** (global const char H = 6)
   - **类型**: `R` (只读数据段)
   - **原因**: 与 `G` 相似，`H` 也是一个全局的 `const` 变量，不可变。

3. **Cptr** ('123456')
   - **类型**: `R` (只读数据段)
   - **原因**: `Cptr` 指向的是一个字符串字面量，字符串字面量通常存储在只读数据段中，以防止被修改。

4. **C** (global int C = 2)
   - **类型**: `D` (已初始化的数据段)
   - **原因**: `C` 是一个已初始化的全局变量，存储在数据段中。

5. **F** (global static int F = 4)
   - **类型**: `d` (局部数据段，但这里应理解为私有初始化数据)
   - **原因**: `F` 是一个静态全局变量，已初始化，并且它是私有的，因此与全局非静态变量相比，存取权限有所不同。

6. **f** (static int f = 4)
   - **类型**: `d` (局部数据段，或私有初始化数据)
   - **原因**: 类似于 `F`，`f` 是静态的，已初始化，通常不会被程序的其他部分直接访问。

7. **A** (global int A)
   - **类型**: `B` (未初始化的全局数据段，bss)
   - **原因**: `A` 是一个全局未初始化的变量，默认为 0，放在 bss 段以节约空间。

8. **B** (global int B = 0)
   - **类型**: `B` (bss)
   - **原因**: 虽然 `B` 被显式初始化为 0，但通常未执行任何操作的全局静态数据（即初始化为默认值）也会放在 bss 段。

9. **D** (global static int D)
   - **类型**: `b` (私有未初始化数据)
   - **原因**: `D` 是一个全局静态未初始化变量，通常存储在专门的静态数据段中，但私有。

10. **E** (global static int E = 0)
    - **类型**: `b` (私有未初始化数据)
    - **原因**: 类似于 `D`，尽管 `E` 显式初始化为 0，但它是静态的且私有。

11. **d** (static int d)
    - **类型**: `b` (私有未初始化数据)
    - **原因**: `d` 作为静态变量，未初始化，私有存储。

12. **e** (static int e = 0)
    - **类型**: `b` (私有未初始化数据)
    - **原因**: 尽管初始化为 0，`e` 作为静态私有变量，其处理方式与其他未初始化静态变量相同。

至于局部变量：
```c
int a;        // 局部未初始化变量
int b = 0;    // 局部已初始化变量
int c = 2;    // 局部初始化变量
...
const int g = 5;  // 局部静态常量
```
可以看到，它们都位于栈区（地址很高）：
|变量|类型|地址|区段
|:-:|:-:|:-:|:-:|
a|int a|0x7ffe78385890|stack
b|int b = 0|0x7ffe78385894|stack
c|int c = 2|0x7ffe78385898|stack
g|const int g = 5 |0x7ffe7838589c|stack


## 二、C++
我们稍微对上一个程序进行修改，增加了一些变量：
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int A;            // 全局未初始化的变量
int B = 0;        // 全局已初始化为0的变量
int C = 2;        // 全局初始化变量
long W ;
long X = 0;
long Y = 1;
static int D;     // 全局静态未初始化变量
static int E = 0; // 全局静态初始化为0变量
static int F = 4; // 全局静态已初始化变量
const int G = 5;  // 全局常量
const char H = 6;

int main(void) {
    int a;        // 局部未初始化变量
    int b = 0;    // 局部已初始化变量
    int c = 2;    // 局部初始化变量
    static int d; // 局部静态未初始化变量
    static int e = 0;
    static int f = 4; // 局部静态已初始化变量
    const int g = 5;  // 局部静态常量

    char char1[] = "abcde"; // 局部字符数组变量
    char *cptr = "123456";  // 指向字符串常量

    //int *heap = malloc(sizeof(int) * 4); // 堆
    int *heap = new int[4];

    printf("PID is %d\n\n", getpid());

    printf("Int A         A_addr = %p\n", &A);
    printf("Int B = 0     B_addr = %p\n", &B);
    printf("Int C = 2     C_addr = %p\n", &C);
    printf("long W        W_addr = %p\n", &W);
    printf("long X = 0    X_addr = %p\n", &X);
    printf("long Y = 1    Y_addr = %p\n", &Y);
    printf("Static int D  D_addr = %p\n", &D);
    printf("Static int E = 0 E_addr = %p\n", &E);
    printf("Static int F = 4 F_addr = %p\n", &F);
    printf("Const int G = 5 G_addr = %p\n", &G);
    printf("Const char H = 6 H_addr = %p\n", &H);

    printf("\n");

    printf("int a               a_addr = %p\n", &a);
    printf("int b = 0           b_addr = %p\n", &b);
    printf("int c = 2           c_addr = %p\n", &c);
    printf("static int d        d_addr = %p\n", &d);
    printf("static int e = 0    e_addr = %p\n", &e);
    printf("static int f = 4    f_addr = %p\n", &f);
    printf("const int g = 5     g_addr = %p\n", &g);

    printf("\n");

    printf("Char array char1[] = \"abcde\"\tAddress = %p\n", (void *)char1);
    printf("Address of char1[]\t\t= %p\n", (void *)&char1);
    printf("Char pointer *cptr = '123456'\tAddress = %p\n", (void *)&cptr);
    printf("Value pointed by cptr\t\t= %c\n", *cptr);
    printf("Cptr points to\t\t\t= %p\n", cptr);
    printf("Heap has space of sizeof(int)*4\tAddress = %p\n", (void *)heap);
    printf("Address of heap pointer\t\t= %p\n", (void *)&heap);

    pause(); // 程序暂停在这里直到收到信号才继续，方便观察进程地址空间

    // 分配的堆内存应该在使用完毕后释放
    free(heap);

    return 0;
}
```
运行结果：

```bash
./maincpp 
PID is 227730

Int A         A_addr = 0x5651e3311030
Int B = 0     B_addr = 0x5651e3311034
Int C = 2     C_addr = 0x5651e3311010
long W        W_addr = 0x5651e3311038
long X = 0    X_addr = 0x5651e3311040
long Y = 1    Y_addr = 0x5651e3311018
Static int D  D_addr = 0x5651e3311048
Static int E = 0 E_addr = 0x5651e331104c
Static int F = 4 F_addr = 0x5651e3311020
Const int G = 5 G_addr = 0x5651e330f008
Const char H = 6 H_addr = 0x5651e330f00c

int a               a_addr = 0x7ffc06dd9ab0
int b = 0           b_addr = 0x7ffc06dd9ab4
int c = 2           c_addr = 0x7ffc06dd9ab8
static int d        d_addr = 0x5651e3311050
static int e = 0    e_addr = 0x5651e3311054
static int f = 4    f_addr = 0x5651e3311024
const int g = 5     g_addr = 0x7ffc06dd9abc

Char array char1[] = "abcde"    Address = 0x7ffc06dd9ad2
Address of char1[]              = 0x7ffc06dd9ad2
Char pointer *cptr = '123456'   Address = 0x7ffc06dd9ac0
Value pointed by cptr           = 1
Cptr points to                  = 0x5651e330f00d
Heap has space of sizeof(int)*4 Address = 0x5651e4a3deb0
Address of heap pointer         = 0x7ffc06dd9ac8
```

打印一下符号表：

```bash
nm -n ./maincpp
...
0000000000001000 t _init
0000000000001120 T _start
0000000000001150 t deregister_tm_clones
0000000000001180 t register_tm_clones
00000000000011c0 t __do_global_dtors_aux
0000000000001200 t frame_dummy
0000000000001209 T main
0000000000001560 T __libc_csu_init
00000000000015d0 T __libc_csu_fini
00000000000015d8 T _fini
0000000000002000 R _IO_stdin_used
0000000000002008 r _ZL1G
000000000000200c r _ZL1H
0000000000002368 r __GNU_EH_FRAME_HDR
00000000000024b4 r __FRAME_END__
0000000000003d78 d __frame_dummy_init_array_entry
0000000000003d78 d __init_array_start
0000000000003d80 d __do_global_dtors_aux_fini_array_entry
0000000000003d80 d __init_array_end
0000000000003d88 d _DYNAMIC
0000000000003f88 d _GLOBAL_OFFSET_TABLE_
0000000000004000 D __data_start
0000000000004000 W data_start
0000000000004008 D __dso_handle
0000000000004010 D C
0000000000004018 D Y
0000000000004020 d _ZL1F
0000000000004024 d _ZZ4mainE1f
0000000000004028 B __bss_start
0000000000004028 b completed.0
0000000000004028 D _edata
0000000000004028 D __TMC_END__
0000000000004030 B A
0000000000004034 B B
0000000000004038 B W
0000000000004040 B X
0000000000004048 b _ZL1D
000000000000404c b _ZL1E
0000000000004050 b _ZZ4mainE1d
0000000000004054 b _ZZ4mainE1e
0000000000004058 B _end
```

先从地址最低的变量开始分析:
|变量|类型|地址|区段
|:-:|:-:|:-:|:-:|
G|global const int G = 5|0x5651e330f008|r
H|global const char H = 6|0x5651e330f00c|r
Cptr|''123456''|0x5651e330f00d|R
C|global int C = 2| 0x5651e3311010|D
Y|global long Y = 1| 0x5651e3311018|D
F|global static int F = 4|0x5651e3311020|d
f|static int f = 4|0x5651e3311024|d
A|global int A|0x5651e3311030|B
B|global int B = 0 |0x5651e3311034|B
W|global long W |0x5651e3311038|B
X|global long X = 0 |0x5651e3311040|B
D|global static int D|0x5651e3311048|b
E|global static int E = 0|0x5651e331104c|b
d|static int d |0x5651e3311050|b
e|static int e = 0|0x5651e3311054|b

可以看到，C/C++中，`.data` 和 `.bss` 的距离非常近。但是 C++对 .rodata 做出了 R、r 的区分。还可以看到内存对齐，比如：
|变量|类型|地址|区段
|:-:|:-:|:-:|:-:|
C|global int C = 2| 0x5651e3311010|D
Y|global long Y = 1| 0x5651e3311018|D
F|global static int F = 4|0x5651e3311020|d

`C` 的地址为 `0x5651e3311010`，而 `Y` 的地址为 `0x5651e3311018`。从 `C` 到 `Y` 的间隔是 `8` 个字节，而非 `4` 个字节。这表明编译器在 `C` 和 `Y` 之间插入了填充（padding），以确保 `Y` 能够在 8 字节边界上对齐。

## 三、总结

我们可以看到可执行程序内部都是分段进行存储的：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/93aa44359ba942fcabbf50c03b8f004a_1720253671766.png?token=ANB4BCIWR6S5CECRDSLV4CDGRD6SW)

- `.text section`：代码段。通常存放已编译程序的机器代码，一般操作系统加载后，这部分是只读的。

- `.rodatasection`：只读数据段。此段的数据不可修改，存放程序中会使用的**常量**。比如程序中的常量字符串 "aasdasdaaasdasd"。

- `.datasection`：数据段。主要用于存放**已初始化的不为 0 全局变量、静态变量**。

- `.bsssection`:  `bss` 段。该段主要存储**未初始化或者初始化为 0 的全局变量、未初始化以及初始化为 0 的全局静态变量、未初始化以及初始化为 0 的静态局部变量。**

操作系统在加载 ELF 文件时会将按照标准依次读取每个段中的内容，并将其加载到内存中，同时为该进程分配栈空间，并将 `pc` 寄存器指向代码段的起始位置，然后启动进程。

从操作系统的本身来讲，以上存储区在该程序内存中的虚拟地址分布是如下形式（虚拟地址从低地址到高地址，实际的物理地址可能是随机的）：`.text→.data→.bss→heap→unused→stack→...`：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/ab6b61a46b554d9d9ea81b9930829204_1720253679644.png?token=ANB4BCPWLLOQJAMFBWNBT4LGRD6S2)



**C++ 程序在运行时也会按照不同的功能划分不同的段**，C++ 程序使用的内存分区一般包括：**栈、堆、全局/静态存储区、常量存储区、代码区。**
- 栈：目前绝大部分 
- CPU 体系都是基于栈来运行程序，栈中主要存放函数的局部变量、函数参数、返回地址等，栈空间一般由操作系统进行默认分配或者程序指定分配，栈空间在进程生存周期一直都存在，当进程退出时，操作系统才会对栈空间进行回收。

- 堆：动态申请的内存空间，就是由 `malloc` 函数或者 `new` 函数分配的内存块，由程序控制它的分配和释放，可以在程序运行周期内随时进行申请和释放，如果进程结束后还没有释放，操作系统会自动回收。我们可以利用

- 全局区/静态存储区：主要为 `.bss` 段和 `.data` 段，存放全局变量和静态变量，程序运行结束操作系统自动释放，在 C 中，未初始化的放在 `.bss` 段中，初始化的放在 `.data` 段中，C++ 中不再区分了。

- 常量存储区：`.rodata` 段，存放的是常量，不允许修改，程序运行结束自动释放。

- 代码区：`.text` 段，存放代码，不允许修改，但可以执行。编译后的二进制文件存放在这里。

**参考**：

> 作者：LeetCode
> 链接：https://leetcode.cn/leetbook/read/cmian-shi-tu-po/vv6a76/


