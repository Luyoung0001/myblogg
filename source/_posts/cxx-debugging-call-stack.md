---
title: "C++ 调试笔记：函数调用过程与调用栈"
date: 2024-05-17 16:43:00
tags:
  - "C++"
  - "debugging"
  - "call stack"
categories:
  - "Programming Languages"
---

<!--more-->

## 正文
本文会讨论C++中函数的调用过程，以及堆栈、栈帧等调试信息。平台以及工具为 Apple Silicon M1 + lldb。

这将会从这个例子开始：

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
void test1(int arg) {
    int a = 1;
    int b = 2;
    int c = a + b + arg;
    printf("test1.%d\n", c);
}
void test2(int arg) {
    int a = 1;
    int b = 2;
    int c = a + b + arg;
    printf("test2.%d\n", c);
    test1(1);
}

void test3(int arg) {
    int a = 1;
    int b = 2;
    int c = a + b + arg;
    printf("test3.%d\n", c);
    test2(2);
}

int main(void) {
    test3(3);
    sleep(1);
    return 0;
}
```
编译、调试：

```bash
 g++ lldbtest.cxx -o main
lldb ./main
(lldb) target create "./main"
Current executable set to '/.../main' (arm64).
(lldb)
```

首先打三个断点，然后运行：

```bash
(lldb) b test1
Breakpoint 1: where = main`test1(int), address = 0x0000000100003dd4
(lldb) b test2
Breakpoint 2: where = main`test2(int), address = 0x0000000100003e34
(lldb) b test3
Breakpoint 3: where = main`test3(int), address = 0x0000000100003ea4
(lldb) run
Process 95341 launched: '/Users/luliang/Cppmianshitupo/C++编译与内存相关/main' (arm64)
Process 95341 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 3.1
    frame #0: 0x0000000100003ea4 main`test3(int)
main`test3:
->  0x100003ea4 <+0>:  sub    sp, sp, #0x30             ; =0x30
    0x100003ea8 <+4>:  stp    x29, x30, [sp, #0x20]
    0x100003eac <+8>:  add    x29, sp, #0x20            ; =0x20
    0x100003eb0 <+12>: stur   w0, [x29, #-0x4]
Target 0: (main) stopped.
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 3.1
  * frame #0: 0x0000000100003ea4 main`test3(int)
    frame #1: 0x0000000100003f34 main`main + 32
    frame #2: 0x000000010001508c dyld`start + 520
```
可以看到，现在程序运行到了 `test3`，也正是打断点的地方，这是 `main` 函数中的第一个函数调用。我们可以看到当前函数中，拥有三个栈帧，但是我们只需要关注 `frame0` 就行了。

```bash
(lldb) f 0
frame #0: 0x0000000100003ea4 main`test3(int)
main`test3:
->  0x100003ea4 <+0>:  sub    sp, sp, #0x30             ; =0x30
    0x100003ea8 <+4>:  stp    x29, x30, [sp, #0x20]
    0x100003eac <+8>:  add    x29, sp, #0x20            ; =0x20
    0x100003eb0 <+12>: stur   w0, [x29, #-0x4]
(lldb) register read
General Purpose Registers:
        ...
        fp = 0x000000016fdff000
        lr = 0x0000000100003f34  main`main + 32
        sp = 0x000000016fdfeff0
        pc = 0x0000000100003ea4  main`test3(int)
      cpsr = 0x60001000
```
我们只需要特殊的几个寄存器中的值，`fp` 指明了当前栈帧，`lr` 是当前 `main()` 返回的地址（后面会解释）。`sp` 是当前栈帧的栈顶指针，`pc` 代表着下一条指令（的位置）：`sub    sp, sp, #0x30`。

继续执行，到了第二个断点 `test2`  ：

```bash
(lldb) continue
Process 95341 resuming
test3.6
Process 95341 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
    frame #0: 0x0000000100003e34 main`test2(int)
main`test2:
->  0x100003e34 <+0>:  sub    sp, sp, #0x30             ; =0x30
    0x100003e38 <+4>:  stp    x29, x30, [sp, #0x20]
    0x100003e3c <+8>:  add    x29, sp, #0x20            ; =0x20
    0x100003e40 <+12>: stur   w0, [x29, #-0x4]
Target 0: (main) stopped.
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.1
  * frame #0: 0x0000000100003e34 main`test2(int)
    frame #1: 0x0000000100003f08 main`test3(int) + 100
    frame #2: 0x0000000100003f34 main`main + 32
    frame #3: 0x000000010001508c dyld`start + 520
(lldb) f 0
frame #0: 0x0000000100003e34 main`test2(int)
main`test2:
->  0x100003e34 <+0>:  sub    sp, sp, #0x30             ; =0x30
    0x100003e38 <+4>:  stp    x29, x30, [sp, #0x20]
    0x100003e3c <+8>:  add    x29, sp, #0x20            ; =0x20
    0x100003e40 <+12>: stur   w0, [x29, #-0x4]
(lldb) register read
General Purpose Registers:
       ...
        fp = 0x000000016fdfefe0
        lr = 0x0000000100003f08  main`test3(int) + 100
        sp = 0x000000016fdfefc0
        pc = 0x0000000100003e34  main`test2(int)
      cpsr = 0x20001000

```
可以看到，当前栈帧、`lr`、`sp`、`pc` 的信息。继续执行：

```bash
(lldb) continue
Process 95341 resuming
test2.5
Process 95341 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x0000000100003dd4 main`test1(int)
main`test1:
->  0x100003dd4 <+0>:  sub    sp, sp, #0x30             ; =0x30
    0x100003dd8 <+4>:  stp    x29, x30, [sp, #0x20]
    0x100003ddc <+8>:  add    x29, sp, #0x20            ; =0x20
    0x100003de0 <+12>: stur   w0, [x29, #-0x4]
Target 0: (main) stopped.
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  * frame #0: 0x0000000100003dd4 main`test1(int)
    frame #1: 0x0000000100003e98 main`test2(int) + 100
    frame #2: 0x0000000100003f08 main`test3(int) + 100
    frame #3: 0x0000000100003f34 main`main + 32
    frame #4: 0x000000010001508c dyld`start + 520
(lldb) f 0
frame #0: 0x0000000100003dd4 main`test1(int)
main`test1:
->  0x100003dd4 <+0>:  sub    sp, sp, #0x30             ; =0x30
    0x100003dd8 <+4>:  stp    x29, x30, [sp, #0x20]
    0x100003ddc <+8>:  add    x29, sp, #0x20            ; =0x20
    0x100003de0 <+12>: stur   w0, [x29, #-0x4]
(lldb) register read
General Purpose Registers:
     ...
        fp = 0x000000016fdfefb0
        lr = 0x0000000100003e98  main`test2(int) + 100
        sp = 0x000000016fdfef90
        pc = 0x0000000100003dd4  main`test1(int)
      cpsr = 0x20001000
```
现在讲这些寄存器的信息全部罗列出来分析：
| 寄存器|test1|test2|test3|
|:-:|:-:|:-:|:-:|
|fp|0x000000016fdfefb0|0x000000016fdfefe0|0x000000016fdff000
|lr|0x0000000100003e98  main`test2(int) + 100|0x0000000100003f08  main`test3(int) + 100|0x0000000100003f34  main`main + 32
|sp|0x000000016fdfef90|0x000000016fdfefc0|0x000000016fdfeff0
|pc|0x0000000100003dd4  main`test1(int)|0x0000000100003e34  main`test2(int)|0x0000000100003ea4  main`test3(int)

结合栈帧信息：
|栈帧|运行到的地址|
|:-:|:-:|
|frame #0| 0x0000000100003dd4 main`test1(int)
|frame #1| 0x0000000100003e98 main`test2(int) + 100
|frame #2| 0x0000000100003f08 main`test3(int) + 100
|frame #3| 0x0000000100003f34 main`main + 32
|frame #4| 0x000000010001508c dyld`start + 520

可以看到，`test1`运行完后，返回的地址就为 `test2(int) + 100`，同样，`test2` 返回的地址就为`test3(int) + 100`, `test3` 返回地址为`main + 32`。

我们再来详细分析分析 C++编译器在函数跳转前，是如何布局的，就拿 test3 跳转进入 test2 之前所做的工作来研究：

```bash
frame #0: 0x0000000100003ea4 main`test3(int)
main`test3:
->  0x100003ea4 <+0>:  sub    sp, sp, #0x30             ; =0x30
    0x100003ea8 <+4>:  stp    x29, x30, [sp, #0x20]
    0x100003eac <+8>:  add    x29, sp, #0x20            ; =0x20
    0x100003eb0 <+12>: stur   w0, [x29, #-0x4]
```

这些指令是 ARM64 架构中的函数序言（prologue）的一部分，用于设置函数的栈帧。每条指令具体完成了以下操作：

### 指令详解

1. **`sub sp, sp, #0x30`**
   - 这条指令从栈指针 `sp` 中减去 48 (`0x30` 十六进制等于 48 十进制)，为局部变量预留空间。这样做是为了在栈上分配足够的空间来存储局部变量、保存的寄存器值等。
   - **效果**：栈向下增长，空间增加。

2. **`stp x29, x30, [sp, #0x20]`**
   - `stp` 是“Store Pair”的缩写，这条指令将两个寄存器（`x29` 和 `x30`）的值存储到栈上。`x29` 通常用作帧指针（fp），而 `x30` 是链接寄存器（lr），存储了返回地址。
   - `[sp, #0x20]` 指的是从当前栈指针 `sp` 的位置向上偏移 32 个字节的地方，这里存放这两个寄存器的值。这里的偏移是为了在栈上预留出其他空间（可能用于保存更多的寄存器或临时数据）。
   - **效果**：保存当前函数的帧指针和返回地址，以便在函数返回时可以恢复。

单行运行汇编指令，我们看看 `x29`、`x30` 寄存器中的值：

```bash
main`test3:
->  0x100003ea4 <+0>:  sub    sp, sp, #0x30             ; =0x30
    0x100003ea8 <+4>:  stp    x29, x30, [sp, #0x20]
    0x100003eac <+8>:  add    x29, sp, #0x20            ; =0x20
    0x100003eb0 <+12>: stur   w0, [x29, #-0x4]
Target 0: (main) stopped.
(lldb) n
Process 96869 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step over
    frame #0: 0x0000000100003ea8 main`test3(int) + 4
main`test3:
->  0x100003ea8 <+4>:  stp    x29, x30, [sp, #0x20]
    0x100003eac <+8>:  add    x29, sp, #0x20            ; =0x20
    0x100003eb0 <+12>: stur   w0, [x29, #-0x4]
    0x100003eb4 <+16>: mov    w8, #0x1
Target 0: (main) stopped.
(lldb) n
Process 96869 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step over
    frame #0: 0x0000000100003eac main`test3(int) + 8
main`test3:
->  0x100003eac <+8>:  add    x29, sp, #0x20            ; =0x20
    0x100003eb0 <+12>: stur   w0, [x29, #-0x4]
    0x100003eb4 <+16>: mov    w8, #0x1
    0x100003eb8 <+20>: stur   w8, [x29, #-0x8]
Target 0: (main) stopped.
(lldb) register read x29
      fp = 0x000000016fdff000
(lldb)
```
从打印的信息可以看到，这个确实存储的是当前的 fp 以及 lr，也就是本函数返回之后的返回地址。

3. **`add x29, sp, #0x20`**
   - 这条指令将 `sp` 的当前值加上 32，结果存储到 `x29`。由于 `x29` 用作帧指针，这条指令实际上是在设置新的帧指针位置，指向刚才通过 `stp` 指令保存的 `x29` 和 `x30` 的位置。
   - **效果**：更新帧指针 `x29`，使其指向当前栈帧中关键数据（旧的帧指针和返回地址）的保存位置。

```bash
    frame #0: 0x0000000100003eac main`test3(int) + 8
main`test3:
->  0x100003eac <+8>:  add    x29, sp, #0x20            ; =0x20
    0x100003eb0 <+12>: stur   w0, [x29, #-0x4]
    0x100003eb4 <+16>: mov    w8, #0x1
    0x100003eb8 <+20>: stur   w8, [x29, #-0x8]
Target 0: (main) stopped.
(lldb) n
Process 96869 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step over
    frame #0: 0x0000000100003eb0 main`test3(int) + 12
main`test3:
->  0x100003eb0 <+12>: stur   w0, [x29, #-0x4]
    0x100003eb4 <+16>: mov    w8, #0x1
    0x100003eb8 <+20>: stur   w8, [x29, #-0x8]
    0x100003ebc <+24>: mov    w8, #0x2
Target 0: (main) stopped.
(lldb) register read x29
      fp = 0x000000016fdfefe0
```
从打印结果来看，这个地址确实是新的栈帧（`test2`）。

**至此，prologue差不多已经完成（只剩下了传参），接着完成 test3 函数的功能**。

4. **`stur w0, [x29, #-0x4]`**
   - `stur` 是“Store Register”的缩写，这条指令将 `w0`（通常用于传递函数的第一个参数）的值存储到由 `x29` 指向的地址向下偏移 4 个字节的位置。
   - 这里使用的是帧指针 `x29`，并向上调整 4 字节，可能是为了在栈帧中保存一个参数值或局部变量。
   - **效果**：将函数的一个参数或局部变量保存在新的栈帧内部，通过帧指针进行偏移访问。



```bash
* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step over
    frame #0: 0x0000000100003eb0 main`test3(int) + 12
main`test3:
->  0x100003eb0 <+12>: stur   w0, [x29, #-0x4]
    0x100003eb4 <+16>: mov    w8, #0x1
    0x100003eb8 <+20>: stur   w8, [x29, #-0x8]
    0x100003ebc <+24>: mov    w8, #0x2
Process 96869 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step over
    frame #0: 0x0000000100003eb4 main`test3(int) + 16
main`test3:
->  0x100003eb4 <+16>: mov    w8, #0x1
    0x100003eb8 <+20>: stur   w8, [x29, #-0x8]
    0x100003ebc <+24>: mov    w8, #0x2
    0x100003ec0 <+28>: stur   w8, [x29, #-0xc]
Target 0: (main) stopped.
(lldb) register read w0
      w0 = 0x00000003
```
w0是传入的参数，这个参数应该是 3，这也和寄存器打印出来的值吻合。
剩下的一些代码就是在执行这样的事：

```cpp
void test3(int arg) {
    int a = 1;
    int b = 2;
    int c = a+b+arg;
    printf("test3.%d\n",c);
    test2(2);
}
```
当然，也可以通过编译生成的汇编代码来查看：

```c
__Z5test3i:                             ; @_Z5test3i
	.cfi_startproc
; %bb.0:
	sub	sp, sp, #48                     ; =48
	stp	x29, x30, [sp, #32]             ; 16-byte Folded Spill
	add	x29, sp, #32                    ; =32
	.cfi_def_cfa w29, 16
	.cfi_offset w30, -8
	.cfi_offset w29, -16
	stur	w0, [x29, #-4]
	mov	w8, #1
	stur	w8, [x29, #-8]
	mov	w8, #2
	stur	w8, [x29, #-12]
	ldur	w9, [x29, #-8]
	ldur	w10, [x29, #-12]
	add	w9, w9, w10
	ldur	w10, [x29, #-4]
	add	w9, w9, w10
	str	w9, [sp, #16]
	ldr	w9, [sp, #16]
                                        ; implicit-def: $x1
	mov	x1, x9
	adrp	x0, l_.str.2@PAGE
	add	x0, x0, l_.str.2@PAGEOFF
	mov	x11, sp
	str	x1, [x11]
	str	w8, [sp, #12]                   ; 4-byte Folded Spill
	bl	_printf
	ldr	w8, [sp, #12]                   ; 4-byte Folded Reload
	mov	x0, x8
	bl	__Z5test2i
	ldp	x29, x30, [sp, #32]             ; 16-byte Folded Reload
	add	sp, sp, #48                     ; =48
	ret
	.cfi_endproc
                                        ; -- End function
	.globl	_main                           ; -- Begin function main
	.p2align	2
```
可以看到：

```bash
	mov	x0, x8
	bl	__Z5test2i
	ldp	x29, x30, [sp, #32]             ; 16-byte Folded Reload
	add	sp, sp, #48                     ; =48
```
这几代码很关键，其中有传值、跳转。**最后两条指令是函数结尾处的序曲，通常称作函数的 epilogue（尾声），它们的主要作用是恢复函数开始时保存的状态，并释放栈空间，以便函数可以正确地返回到调用者。** 具体来说：

### 指令详解

1. **`ldp x29, x30, [sp, #32]`**：
   - 这条指令是“Load Pair”的缩写，用于从栈上加载两个寄存器的值。这里从栈指针 `sp` 偏移 32 字节的位置同时加载 `x29` 和 `x30` 寄存器的值。
   - `x29` 通常作为帧指针（Frame Pointer），在函数开始时保存了前一个栈帧的帧指针值。
   - `x30` 是链接寄存器（Link Register），保存了函数调用的返回地址。这样，函数可以通过这个地址返回到调用者。

2. **`add sp, sp, #48`**：
   - 这条指令将栈指针 `sp` 增加 48，目的是释放当前函数在栈上使用的空间。在函数的 prologue（序言）部分，我们通常会看到栈指针减少相同的值以分配空间给局部变量和保存的寄存器。现在，通过增加相同的值，恢复栈指针到函数调用前的状态。

### 功能和目的

这部分代码标志着函数即将结束，准备返回到其调用者。执行以下操作：

- **恢复寄存器状态**：通过 `ldp` 指令恢复 `x29` 和 `x30`，确保函数能够恢复调用前的栈帧结构，并通过 `x30` 存储的地址返回到正确的位置。
- **释放栈空间**：通过增加 `sp` 的值，函数释放它在栈上分配的所有局部变量和其他临时存储空间。

**这两步是函数正确结束并且栈保持平衡的关键，确保程序的调用栈在多次函数调用后仍然保持一致和正确。**

这样的机制是所有现代编译器生成的代码中常见的做法，用于确保函数能够按照预期安全地执行和返回，不会造成栈内存的混乱或泄漏。

通过以上例子，我们可以大概明白一个栈帧的抽象结构应该和下表类似：





|值 |解释|
|:---------------------:| :------:|
| ---------------------| <- 更高的内存地址
|  参数 n             |
|  ...                |
|  参数 2             |
|  参数 1             |
|  返回地址           |
|  旧的帧指针         | <- 帧指针 (ebp/rbp)
|  局部变量 1         |
|  局部变量 2         |
|  ...                |
|  局部变量 m         |
|---------------------|- <- 更低的内存地址


接着回答前面的一个问题，`main` 函数返回到哪里。
首先可以看到，有一个这个栈帧：**frame #2: 0x000000010001508c dyld`start + 520**

这是由动态链接器 `dyld` 提供的一个特殊的 **启动栈帧，用于初始化和启动程序。这是每个应用程序启动时都会执行的代码，其作用是设置程序运行环境，加载必要的库等，然后调用 `main` 函数。** 因此，`main` 函数会返回到这里。
### `main` 函数的返回地址

在大多数操作系统和运行时环境中，当 `main` 函数完成执行后，它将返回到其调用者，这通常是操作系统或程序的运行时环境提供的启动代码（如 C 或 C++ 程序中的 `dyld` 的 `start` 函数）。返回地址是在 `main` 函数被调用时自动放置在栈上的，这部分通常由程序的启动代码处理。

- **操作系统环境**：在程序的执行即将结束时，`main` 函数的返回值（通常是通过 `return` 语句提供的整数）会被用作程序的退出代码。这个退出代码被返回到操作系统，表明程序是成功执行还是出错退出。

- **结束程序执行**：`main` 函数返回后，控制权交回操作系统，操作系统根据返回的状态码来进行后续操作（例如，释放资源、处理程序退出状态等）。


