---
title: "RISC-V 函数调用实验：调用约定与栈帧观察"
date: 2024-10-14 13:20:40
tags:
  - "RISC-V"
  - "calling convention"
  - "assembly"
categories:
  - "Computer Architecture"
---

<!--more-->

## 前言
昨天开组会，助教在会上问了一个问题：A 函数调用 B 函数，s0 的保存是哪一个函数进行的。我觉得这个问题挺熟悉的，因为之前在看 riscv 的 xv6 操作系统时遇到过。

## 实验
main.c:
```C
#include <add.h>
#include <stdio.h>

int main() {
    int a = 10, b = 5;
    int c = add(a, b);
    return 0;
}
```
add.h、add.c:
```C
#ifndef __ADD_H__
#define __ADD_H__

int add(int a, int b);

#endif


#include <add.h>

int add(int a, int b) {
    a += 1;
    a += 2;
    a += 3;
    a += 5;
    a += 10;
    a += b;
    return a;
}

```

Makefile:

```Makefile
INC_PATH = /home/luyoung/Test/1014_test
C_FLAGS = -I$(INC_PATH)

build: add.o main.o
	riscv64-linux-gnu-gcc -march=rv32g -mabi=ilp32 -nostdlib -O0 -o main add.o main.o

add.o: add.c
	bear -- riscv64-linux-gnu-gcc -march=rv32g -mabi=ilp32 -c add.c -o add.o $(C_FLAGS) -O0

main.o: main.c
	bear -- riscv64-linux-gnu-gcc -march=rv32g -mabi=ilp32 -c main.c -o main.o $(C_FLAGS) -O0

dump: main
	riscv64-linux-gnu-objdump -d -M no-aliases main

clean:
	rm -f *.o main

```
在编译好了之后：

```bash
$ make dump
riscv64-linux-gnu-objdump -d -M no-aliases main

main:     file format elf32-littleriscv


Disassembly of section .text:

000001d8 <add>:
 1d8:   fe010113                addi    sp,sp,-32
 1dc:   00812e23                sw      s0,28(sp)
 1e0:   02010413                addi    s0,sp,32
 1e4:   fea42623                sw      a0,-20(s0)
 1e8:   feb42423                sw      a1,-24(s0)
 1ec:   fec42783                lw      a5,-20(s0)
 1f0:   00178793                addi    a5,a5,1
 1f4:   fef42623                sw      a5,-20(s0)
 1f8:   fec42783                lw      a5,-20(s0)
 1fc:   00278793                addi    a5,a5,2
 200:   fef42623                sw      a5,-20(s0)
 204:   fec42783                lw      a5,-20(s0)
 208:   00378793                addi    a5,a5,3
 20c:   fef42623                sw      a5,-20(s0)
 210:   fec42783                lw      a5,-20(s0)
 214:   00578793                addi    a5,a5,5
 218:   fef42623                sw      a5,-20(s0)
 21c:   fec42783                lw      a5,-20(s0)
 220:   00a78793                addi    a5,a5,10
 224:   fef42623                sw      a5,-20(s0)
 228:   fec42703                lw      a4,-20(s0)
 22c:   fe842783                lw      a5,-24(s0)
 230:   00f707b3                add     a5,a4,a5
 234:   fef42623                sw      a5,-20(s0)
 238:   fec42783                lw      a5,-20(s0)
 23c:   00078513                addi    a0,a5,0
 240:   01c12403                lw      s0,28(sp)
 244:   02010113                addi    sp,sp,32
 248:   00008067                jalr    zero,0(ra)

0000024c <main>:
 24c:   fe010113                addi    sp,sp,-32
 250:   00112e23                sw      ra,28(sp)
 254:   00812c23                sw      s0,24(sp)
 258:   02010413                addi    s0,sp,32
 25c:   00a00793                addi    a5,zero,10
 260:   fef42223                sw      a5,-28(s0)
 264:   00500793                addi    a5,zero,5
 268:   fef42423                sw      a5,-24(s0)
 26c:   fe842583                lw      a1,-24(s0)
 270:   fe442503                lw      a0,-28(s0)
 274:   f65ff0ef                jal     ra,1d8 <add>
 278:   fea42623                sw      a0,-20(s0)
 27c:   00000793                addi    a5,zero,0
 280:   00078513                addi    a0,a5,0
 284:   01c12083                lw      ra,28(sp)
 288:   01812403                lw      s0,24(sp)
 28c:   02010113                addi    sp,sp,32
 290:   00008067                jalr    zero,0(ra)
```

可以看到，main 函数在刚开始：

```asm
 24c:   fe010113                addi    sp,sp,-32
 250:   00112e23                sw      ra,28(sp)
 254:   00812c23                sw      s0,24(sp)
 258:   02010413                addi    s0,sp,32
```
首先修改 sp，由于 riscv 的栈空间是向下增长的，这是分配了 32 字节的空间。然后将寄存器 ra、s0 保存到 main 空间，接着重置 s0 为 main 栈空间起始地址。

我们一般把这个叫做前言。

sp（栈指针）：通常指向当前栈的顶部，管理局部变量、临时数据、返回地址等。sp 在函数调用时减少，为当前函数的局部变量和参数分配空间。
s0（帧指针）：用于指向栈帧的基准位置，便于访问局部变量和参数。它在函数执行期间保持不变，以方便访问栈中的数据。

这个 ra （return address）是 main 函数的返回地址，因为 main 中也要进行函数调用，而这个过程需要修改 ra 以让 add 函数能正确返回。因此在前言中也要把 ra 保存起来。

在 main 函数通过 jal 调用 add 之后，首先它会把返回地址，也就是 `278:fea42623 sw a0,-20(s0)` 这个指令的地址放到 ra 中。然后无条件跳转到 `1d8`。

跳到 add 之后，依然要进行函数前言：

```asm
 1d8:   fe010113                addi    sp,sp,-32
 1dc:   00812e23                sw      s0,28(sp)
 1e0:   02010413                addi    s0,sp,32
```

它依然会设置栈帧、帧指针。但是它并没有保存 ra，这是因为 add 函数没有进行函数调用。

执行完了之后，它就会进入后记：

```asm
 23c:   00078513                addi    a0,a5,0
 240:   01c12403                lw      s0,28(sp)
 244:   02010113                addi    sp,sp,32
 248:   00008067                jalr    zero,0(ra)
```

可以看到，它会把返回值放到 a0，然后恢复调用它的函数 main 的 sp、s0，然后返回。

函数返回相当简单，它的 rd 为 0 号寄存器，这意味着它不用保存返回地址，即它是一个函数返回，不是一个函数调用。另外，它的参数 ra 和 ra 是一个值，这意味着它确实要返回。

下面的C 语言描述更能说明这两个指令的情况：

```C
    INSTPAT("??????? ????? ????? ??? ????? 11011 11", jal, J,
            s->dnpc = s->pc + imm;
            R(rd) = s->pc + 4; IFDEF(
                CONFIG_FTRACE,
                if (rd != 0) { print_ftrace_call(s->pc, s->dnpc); }));

    INSTPAT(
        "??????? ????? ????? 000 ????? 11001 11", jalr, I,
        s->dnpc = (src1 + imm) &
                  ~(word_t)1;  // 在 RISC-V 架构中，指令地址必须对齐到偶数地址
        if (rd != 0) R(rd) = s->pc + 4; IFDEF(
            CONFIG_FTRACE,
            if (rd != 0) {  // 跳转
                print_ftrace_call(s->pc, s->dnpc);
            } else if (rd == 0 && src1 == R(1)) {  // src1 和 ra 是否相同
                print_ftrace_ret(s->pc, s->dnpc);
            }));
```

同样，返回之后，就到了 main 函数的这里：
```asm
 278:   fea42623                sw      a0,-20(s0)
 27c:   00000793                addi    a5,zero,0
 280:   00078513                addi    a0,a5,0
 284:   01c12083                lw      ra,28(sp)
 288:   01812403                lw      s0,24(sp)
 28c:   02010113                addi    sp,sp,32
 290:   00008067                jalr    zero,0(ra)
```

可以看到，它首先保存返回值，然后把返回值重新放到 a0。之后就是启动函数（调用 main 的函数）的返回了，恢复 ra、s0、sp之后，直接返回。

可以看到，main 函数调用 add 之后， sp 的开辟、s0 的保存、s0 的设置，都是 add 函数进行的。遵守着`谁使用，谁负责整理和维护`的规则。比如，main 要调用函数，那么它就得保存 ra，然后给 ra 中放置add的返回地址；add 要返回，那么add 就得恢复 s0、sp。








