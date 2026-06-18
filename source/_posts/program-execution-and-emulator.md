---
title: "程序执行与模拟器：freestanding 环境下的运行机制"
date: 2024-08-09 11:22:33
tags:
  - "emulator"
  - "freestanding"
  - "runtime"
categories:
  - "Computer Architecture"
---

<!--more-->

## freestanding 运行时环境

示例程序：
```c
#include <stdint.h>
void _start() {
    volatile uint8_t* p = (uint8_t*)(uintptr_t)0x10000000;
    *p = 'A';
    while (1)
        ;
}
```

编译：`rv32gcc -ffreestanding -nostdlib -Wl,-Ttext=0x80000000 -O2 a.c`:

命令解析：

### -ffreestanding
- 指示编译器，代码是在“自由环境”（freestanding environment）下运行的，这意味着标准库可能不完全可用，程序不能依赖标准启动环境。这常用于系统软件开发，如操作系统内核。

### -nostdlib
- 禁止链接器链接标准启动文件和标准库。这个选项确保不会将标准库的任何部分链接到最终的可执行文件中，这对于需要直接控制程序运行环境的系统级编程至关重要。

### -Wl,-Ttext=0x80000000
- 这是一个向链接器传递参数的选项。`-Wl,option` 表示将 `option` 传递给链接器。这里的 `-Ttext=0x80000000` 设置程序的入口地址（即 `.text` 段开始的位置）为 `0x80000000`。这通常用于指定程序在内存中的加载位置，是嵌入式系统和操作系统内核中常见的需求。

### -O2
- 启用优化级别为 2。这是一种中等程度的优化，能够在不影响编译时间太多的前提下，提高程序的运行效率。这种优化级别会尝试不改变程序行为的前提下，提高程序的执行速度和/或减少占用的内存空间。

运行：`qemu-system-riscv32 -nographic -M virt -bios none -kernel a.out`:


### qemu-system-riscv32
- 这是 QEMU 的一个变体，专门用于模拟 32位的 RISC-V 架构。

### -nographic
- 这个选项指示 QEMU 在没有图形界面的情况下运行，所有的输出都会发送到控制台。这对于运行服务器或无头系统（没有显示器、键盘等外围设备的系统）非常有用。

### -M virt
- `-M` 参数用于指定模拟的机器类型。在这里，`virt` 是一个为各种客户端操作系统提供足够支持的通用 RISC-V 系统。这种类型的虚拟机提供了基本的设备和是进行各种测试的理想选择。

### -bios none
- 此参数指定不使用任何 BIOS。这意味着 QEMU 不会尝试从任何传统的 BIOS 固件启动，这通常用于直接启动裸机应用或特定的操作系统，这些操作系统不依赖于传统 BIOS 启动。

### -kernel a.out
- `-kernel` 用于直接加载并启动一个内核或可执行文件。这里的 `a.out` 是要被加载的文件，通常是一个编译好的内核或应用程序。这使得可以直接在模拟环境中测试编译后的程序。

这个程序是往0x10000000中写入了一个字符，这是因为0x10000000 是 virt 机器串口地址，因此想要打印出这个字符，必须加上参数-M virt。

## 怎么结束程序？

在qemu-system-riscv32中的virt机器模型中, 往一个特殊的地址写入一个特殊的字符即可结束QEMU的运行。

```c
#include <stdint.h>
void _start() {
    volatile uint8_t* p = (uint8_t*)(uintptr_t)0x10000000;
    *p = 'A';
    volatile uint32_t* exit = (uint32_t*)(uintptr_t)0x100000;
    *exit = 0x5555;  // magic number
    _start();
}

```

```bash
$ qemu-system-riscv32 -nographic -M virt -bios none -kernel a.out
A%
```

## 自制 freestanding 运行时环境

- 程序从地址0开始执行
- 只支持两条指令
  - addi指令
  - ebreak指令
  - - 寄存器a0=0时, 输出寄存器a1低8位的字符
  - - 寄存器a0=1时, 结束运行
   - ABI Mnemonic


示例代码：

```c
static void ebreak(long arg0, long arg1) {
    asm volatile(
        "addi a0, x0, %0;"
        "addi a1, x0, %1;"
        "ebreak"
        :
        : "i"(arg0), "i"(arg1));
}
static void putch(char ch) {
    ebreak(0, ch);
}
static void halt(int code) {
    ebreak(1, code);
    while (1)
        ;
}

void _start() {
    putch('A');
    halt(0);
}

```

编译、查看反汇编：
```bash
rv32gcc -ffreestanding -nostdlib -static -Wl,-Ttext=0 -O2 -o prog a.c
rvobjdump -M no-aliases -d prog

prog:     file format elf32-littleriscv


Disassembly of section .text:

00000000 <_start>:
   0:   00000513                addi    a0,zero,0
   4:   04100593                addi    a1,zero,65
   8:   00100073                ebreak
   c:   00100513                addi    a0,zero,1
  10:   00000593                addi    a1,zero,0
  14:   00100073                ebreak
  18:   0000006f                jal     zero,18 <_start+0x18>
```


可以看到确实只有两种汇编指令addi、ebreak。事实上，这一点都不神奇，因为 gcc 支持在 c 中插入汇编代码，上面的程序除了汇编逻辑，没有任何其他逻辑。展开之后，自然就只有那两种汇编指令了（jal 是跳转指令，不用考虑）。

## 怎么让这个程序运行呢？

我们可以找一个真实的 RISCV CPU 直接在裸机上运行，也可以在虚拟机比如 qemu-system-riscv32 上运行（事实上，这是刚做的事情，现在自己写一个程序来运行它），还可以自己写一个模拟器，来运行它。

如果想要在qemu-system-riscv32 运行它，必须对源程序做一些改造，比如：

```c
static void ebreak(long arg0, long arg1) {
    asm volatile(
        "addi a0, x0, %0;"
        "addi a1, x0, %1;"
        "ebreak"
        :
        : "i"(arg0), "i"(arg1));
}
static void putch(char ch) {
    ebreak(0, ch);
}
static void halt(int code) {
    ebreak(1, code);
    while (1)
        ;
}

void _start() {
    // 将字符 'A' 写入内存地址 0x10000000
    char* ptr = (char*)0x10000000;
    *ptr = 'A';

    putch('A');
    halt(0);
}

```
```bash
$ rv32gcc -ffreestanding -nostdlib -static -Wl,-Ttext=0x80000000 -O2 -o prog b.c

$ qemu-system-riscv32 -nographic -M virt -bios none -kernel prog

A
```

## YEMU

可以读取指令，然后分析指令，然后运行指令，顺便将 PC++。以上全部用 C语言进行语义模拟，就得到了一个模拟器，看起来很简单。这有点像用22世纪的 CPU 模拟 20 世界 80 年代的 8086，效率非常低，但是却能说明模拟器工作的方式。


### ISA 状态机

- 状态集合S={<R,M>}：
- - R={PC,x0,x1,x2,…}
- - RISC-V手册 -> 2.1 Programmers’ Model for Base Integer ISA
- - PC = 程序计数器 = 当前执行的指令位置
- - M = 内存
- - RISC-V手册 -> 1.4 Memory
- 激励事件E={指令}
- - 执行PC指向的指令
- 状态转移规则next:S×E→S
- - 指令的语义(semantics)
- 初始状态S0=<R0,M0>

换句话说，就是初始状态、状态转移(具体状态转移函数)、状态集合。

如果要用 C语言来模拟，那就得：

- 用C语言变量实现寄存器和内存；
- 用C语言语句实现指令的语义：
- - 指令采用符号化表示 -> 汇编模拟器
- - 指令采用编码表示 -> 传统的(二进制)指令集模拟器


### 实现寄存器和内存

```c
#include <stdint.h>
uint32_t R[32], PC; // according to the RISC-V manual
uint8_t M[64];      // 64-Byte memory
```
这里内存为 64 字节的无符号数，至于为什么，这个和 pa1 中用无符号数作表达式一样，因为无符号数没有溢出的概念，如果换成有符号数，会 UB。

### 用语句实现指令的语义

指令周期(instruction cycle): 执行一条指令的步骤：

- 取指(fetch): 从PC所指示的内存位置读取一条指令；
- 译码(decode): 按手册解析指令的操作码(opcode)和操作数(operand)；
- 执行(execute): 按解析出的操作码, 对操作数进行处理；
- - 若写入<R,M>, 则更新状态
- 更新PC: 让PC指向下一条指令；
- - 更新状态

RTFM 之后，发现这两条指令的定义为：

```txt
 31           20 19 15 14 12 11  7 6       0
+---------------+-----+-----+-----+---------+
|   imm[11:0]   | rs1 | 000 | rd  | 0010011 |    ADDI
+---------------+-----+-----+-----+---------+
+---------------+-----+-----+-----+---------+
| 000000000001  |00000| 000 |00000| 1110011 |   EBREAK
+---------------+-----+-----+-----+---------+
```
因此，首先得判断指令，然后执行指令，一个不成熟的实现：

```c

void inst_cycle() {
    uint32_t inst = *(uint32_t*)&M[PC];
    if (((inst & 0x7f) == 0x13) && ((inst >> 12) & 0x7) == 0) {  // addi
        if (((inst >> 7) & 0x1f) != 0) {
            R[(inst >> 7) & 0x1f] =
                R[(inst >> 15) & 0x1f] +
                (((inst >> 20) & 0x7ff) - ((inst & 0x80000000) ? 4096 : 0));
        }
    } else if (inst == 0x00100073) {  // ebreak
        if (R[10] == 0) {
            putchar(R[11] & 0xff);
        } else if (R[10] == 1) {
            halt = true;
        } else {
            printf("Unsupported ebreak command\n");
        }
    } else {
        printf("Unsupported instuction\n");
    }
    PC += 4;
}
```

判断是哪一条指令，要看这个指令的特征，比如 addi 的特征就是最低 7 位为 0x13 并且 12~14 位 为 0x0。判断出来后，就要取出 rd,这是目标寄存器，不能为 0；接着就是计算了，这里需要注意，`(inst & 0x80000000) ? 4096 : 0`：这里检查指令的最高位（第 31 位），即立即数的符号位。在 RISC-V 中，立即数使用的是补码表示。如果符号位为 1（即 inst & 0x80000000 为真），表示这是一个负数，那么需要进行符号扩展。在这种情况下，将整个立即数扩展为 32 位负数，而 -4096（二进制为 1111 0000 0000 0000，补码表示的 -4096）正是扩展这 12 位为全 32 位所需的值。这个数值补充了高位的 1 们，完成从 12 位到 32 位的扩展。

至于 ebreak，R[10]其实就是 a0, R[11]就是a1。a0 为 0,输出a1的低八位；a0为 1，终止程序。可以看到以上所有语义都是由 C 语言进行模拟的。至于初始状态的定义，自然就是：

```c
R[0] = 0;
PC = 0;
M = 程序;//ABI
```

因此就有了第一版的 YEMU：

```c
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
uint32_t R[32], PC;
uint8_t M[64] = {
    0x13, 0x05, 0x00, 0x00, 0x93, 0x05, 0x10, 0x04, 0x73, 0x00,
    0x10, 0x00, 0x13, 0x05, 0x10, 0x00, 0x93, 0x05, 0x00, 0x00,
    0x73, 0x00, 0x10, 0x00, 0x6f, 0x00, 0x00, 0x00,
};
bool halt = false;

void inst_cycle() {
    uint32_t inst = *(uint32_t*)&M[PC];
    if (((inst & 0x7f) == 0x13) && ((inst >> 12) & 0x7) == 0) {  // addi
        if (((inst >> 7) & 0x1f) != 0) {
            R[(inst >> 7) & 0x1f] =
                R[(inst >> 15) & 0x1f] +
                (((inst >> 20) & 0x7ff) - ((inst & 0x80000000) ? 4096 : 0));
        }
    } else if (inst == 0x00100073) {  // ebreak
        if (R[10] == 0) {
            putchar(R[11] & 0xff);
        } else if (R[10] == 1) {
            halt = true;
        } else {
            printf("Unsupported ebreak command\n");
        }
    } else {
        printf("Unsupported instuction\n");
    }
    PC += 4;
}

int main() {
    PC = 0;
    R[0] = 0;  // can be omitted since uninitialized global variables are
               // initialized with 0
    while (!halt) {
        inst_cycle();
    }
    return 0;
}

```

可以看到：

```bash
$ rvobjdump -M no-aliases -d prog

prog:     file format elf32-littleriscv


Disassembly of section .text:

00000000 <_start>:
   0:   00000513                addi    a0,zero,0
   4:   04100593                addi    a1,zero,65
   8:   00100073                ebreak
   c:   00100513                addi    a0,zero,1
  10:   00000593                addi    a1,zero,0
  14:   00100073                ebreak
  18:   0000006f                jal     zero,18 <_start+0x18>
```

```c
uint8_t M[64] = {
    0x13, 0x05, 0x00, 0x00, 0x93, 0x05, 0x10, 0x04, 0x73, 0x00,
    0x10, 0x00, 0x13, 0x05, 0x10, 0x00, 0x93, 0x05, 0x00, 0x00,
    0x73, 0x00, 0x10, 0x00, 0x6f, 0x00, 0x00, 0x00,
};
```
之所以 uint32_t 取的字符和原来的指令逆序，是因为计算机遵从小端序列：低地址内存中的数据，放在数据（uint32_t、数组等）的低位。对于内存中的指令：0x00000513，0x13位于高地址，放在数组高位，这里需要注意。

### 执行

```bash
$ gcc yemu.c -o yemu

$ ./yemu prog
A%

```

可以看见，工作很正常。

### 改进程序，从文件读入程序

这样才是一个正常的工作流程，谁都不可能有把一个二进制文件一个一个写入数组的精力：

```c
...
int main(int argc, char* argv[]) {
    PC = 0;
    R[0] = 0;
    FILE* fp = fopen(argv[1], "r");
    fread(M, 1, 1024, fp);
    fclose(fp);
    while (!halt) {
        inst_cycle();
    }
    return 0;
}
```
在运行 prog 之前，必须将里面的指令序列抽取到 prog.bin，毕竟：

```bash
$ riscv64-linux-gnu-objcopy -j .text -O binary prog prog.bin

$ ll
total 56K
-rw-rw-r-- 1 luyoung luyoung  224 Aug  9 10:31 a.c
-rwxrwxr-x 1 luyoung luyoung 9.4K Aug  9 10:31 a.out
-rw-rw-r-- 1 luyoung luyoung  351 Aug  9 11:48 b.c
-rwxrwxr-x 1 luyoung luyoung 5.1K Aug  9 11:48 prog
-rwxrwxr-x 1 luyoung luyoung   28 Aug  9 11:58 prog.bin
-rwxrwxr-x 1 luyoung luyoung  17K Aug  9 11:57 yemu
-rw-rw-r-- 1 luyoung luyoung 1.2K Aug  9 11:57 yemu.c

$ ./yemu prog.bin
A%

```

可以看到，prog 和 prog.bin 的大小差距很大。

### 运行更复杂的程序

我们用 yemu 执行的程序只能输出一个字符，我们可以尝试输出得更多一点：

```c
void _start() {
  putch('H'); putch('e'); putch('l'); putch('l'); putch('o'); putch(','); putch(' ');
  putch('R'); putch('I'); putch('S'); putch('C'); putch('-'); putch('V'); putch('!');
  putch('\n');
  halt(0);
}
```
重新编写 prog，编译，抽取指令序列，运行 yemu：
```bash
luyoung at luyoungUbt in ~/ysyx_details/预学习阶段/slides/程序的执行和模拟器 (main●●)
$ rv32gcc -ffreestanding -nostdlib -static -Wl,-Ttext=0 -O2 -o prog b.c

luyoung at luyoungUbt in ~/ysyx_details/预学习阶段/slides/程序的执行和模拟器 (main●●)
$ riscv64-linux-gnu-objcopy -j .text -O binary prog prog.bin

luyoung at luyoungUbt in ~/ysyx_details/预学习阶段/slides/程序的执行和模拟器 (main●●)
$ ./yemu prog.bin
Hello, RISC-V!
```

至于其它编程技巧，用宏增强代码复用，就不累赘了。












