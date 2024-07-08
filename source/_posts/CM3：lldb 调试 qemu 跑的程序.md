---
title: CM3：lldb 调试 qemu 跑的程序
date: 2024-04-28 11:50:57
tags:
    - 汇编
    - QEMU
    - CM3
categories: CM3
---

<!--more-->

## 正文
环境：macOS M1。
QEMU可以通过启动一个GDB调试端口来允许使用GDB调试正在运行的虚拟机，我们要做的就是通过 gdb 或者 lldb 连接到这个端口，然后进行调试。
我们先写一个简单的CM3 程序：

```c
    .equ STACK_TOP, 0x20000800
    .text
    .global _start
    .code 16
    .syntax unified

_start:

    .word STACK_TOP,start
    .type start,function
start:
    movs r0, #10
    movs r1, #0
loop:
    adds r1, r0
    subs r0, #1
    bne loop
deadloop:
    b deadloop

    .end
```

这个程序的作用是来计算 `10+9+...+1`的结果。

我们通过这几个命令来完成汇编、链接、目标拷贝，甚至还可以生成反汇编代码，用来辅助查看：

```bash
arm-none-eabi-as -mcpu=cortex-m3 -mthumb example1.s -o example1.o
arm-none-eabi-ld -Ttext=0x0 -o example1.out example1.o
arm-none-eabi-objcopy -O binary example1.out example1.bin
arm-none-eabi-objdump -S example1.out > example1.list
```

反汇编的代码 list：

```c

example1.out:     file format elf32-littlearm


Disassembly of section .text:

00000000 <_start>:
   0:	20000800 	.word	0x20000800
   4:	00000009 	.word	0x00000009

00000008 <start>:
   8:	200a      	movs	r0, #10
   a:	2100      	movs	r1, #0

0000000c <loop>:
   c:	1809      	adds	r1, r1, r0
   e:	3801      	subs	r0, #1
  10:	d1fc      	bne.n	c <loop>

00000012 <deadloop>:
  12:	e7fe      	b.n	12 <deadloop>

```
然后我们启动调试功能：

```bash
qemu-system-arm -M lm3s6965evb -serial stdio -kernel example1.bin -s -S
```

这个命令使用QEMU模拟器来模拟`lm3s6965evb`开发板上的ARM处理器，并加载`example1.bin`文件作为内核镜像。具体来说，这个命令的各个参数的作用如下：

- `-M lm3s6965evb`：指定使用QEMU模拟器模拟`lm3s6965evb`开发板，该开发板基于Cortex-M3处理器。
- `-serial stdio`：将串口输出重定向到标准输入输出，这样可以在控制台上查看虚拟机的输出信息。
- `-kernel example1.bin`：指定将`example1.bin`文件作为内核镜像加载到虚拟机中运行。
- `-s`：启动GDB服务器，监听默认端口`1234`，允许通过GDB进行远程调试。
- `-S`：在启动时暂停虚拟机，等待GDB连接后再开始执行。

然后我们来通过 lldb 连接到这个端口：

```bash
lldb
(lldb) gdb-remote 1234
Process 1 stopped
* thread #1, stop reason = signal SIGTRAP
    frame #0: 0x00000008
->  0x8: .long  0x2100200a                ; unknown opcode
    0xc: stmdalo r1, {r0, r3, r11, r12}
    0x10: udf    #0xed1c
    0x14: andeq  r0, r0, r0
Target 0: (No executable module.) stopped.
```

这个信息表明已经成功通过LLDB连接到QEMU的GDB调试端口，并且虚拟机已经被暂停在一个位置。在这个特定的情况下，虚拟机暂停在地址`0x00000008`处，这是一个未知的指令。下面是对输出信息的解释：

- `Process 1 stopped`：虚拟机中的进程已经停止。
- `thread #1, stop reason = signal SIGTRAP`：线程1停止的原因是收到了`SIGTRAP`信号，这通常是由调试器发送给进程的信号，用于暂停执行。
- `frame #0: 0x00000008`：当前帧在地址`0x00000008`处。
- `0x8: .long  0x2100200a`：在地址`0x00000008`处，存储的是一个未知的指令`0x2100200a`。
- `0xc: stmdalo r1, {r0, r3, r11, r12}`：接下来的指令是`stmdalo r1, {r0, r3, r11, r12}`。
- `0x10: udf    #0xed1c`：接下来的指令是`udf #0xed1c`。
- `0x14: andeq  r0, r0, r0`：接下来的指令是`andeq r0, r0, r0`。

最后一行`Target 0: (No executable module.) stopped.`表示当前没有可执行模块，即虚拟机中没有正在执行的程序。
继续单步执行，之后打印寄存器，看看运行情况：

```bash
(lldb) register read
general:
        r0 = 0x00000006
        r1 = 0x00000022
        r2 = 0x00000000
        r3 = 0x00000000
        r4 = 0x00000000
        r5 = 0x00000000
        r6 = 0x00000000
        r7 = 0x00000000
        r8 = 0x00000000
        r9 = 0x00000000
       r10 = 0x00000000
       r11 = 0x00000000
       r12 = 0x00000000
        sp = 0x20000800
        lr = 0xffffffff
        pc = 0x00000010
      xpsr = 0x21000000
       msp = 0x20000800
       psp = 0x00000000
   primask = 0x00000000
   control = 0x00000000
   basepri = 0x00000000
  faultmask = 0x00000000
```
可以看到，`r0` 此时为 6，这也和 `r1` = `10+9+8+7=0x22` 吻合。

这就是一次简单的调试任务。

