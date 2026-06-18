---
title: "异常处理再探：最小异常处理程序与运行时支持"
date: 2024-11-19 16:20:40
tags:
  - "exception"
  - "ysyx"
  - "runtime"
categories:
  - "Computer Architecture"
---

<!--more-->

## 前言
阅读本案例需要有做 nemu 的经历，最低要求是是做到 PA3.1。

## 一个最简单的异常处理程序
```C
#include <klib.h>
void handler() {
    uintptr_t mepc;
    asm volatile("csrr %0, mepc" : "=r"(mepc));
    printf("exception at mepc = %p\n", mepc);
    while (1)
        ;
}

int main() {
    asm volatile("csrw mtvec, %0" : : "r"(handler));
    asm volatile(".word 0");  // illegal instruction
    printf("I am alive!\n");
    while (1)
        ;
}
```
这是 Makefile：
```Makefile

NAME = simple-rv-handler
SRCS = simple-rv-handler.c
include $(AM_HOME)/Makefile

```

### 在 riscv32—nemu 运行
这是一个很简单的异常处理程序，将它编译成 riscv32，并尝试在 nemu 上运行：

```bash
$ make ARCH=riscv32-nemu -j8 run
```

就会发现，程序并不能按照想象中的运行，这是因为 nemu 并没有加入指令异常处理机制：
```bash
...
invalid opcode(PC = 0x80000044):
        00 00 00 00 17 05 00 00 ...
        00000000 00000517...
There are two cases which will trigger this unexpected exception:
1. The instruction at PC = 0x80000044 is not implemented.
2. Something is implemented incorrectly.
Find this PC(0x80000044) in the disassembling result to distinguish which case it is.

If it is the first case, see
       _                         __  __                         _
      (_)                       |  \/  |                       | |
  _ __ _ ___  ___ ________   __ | \  / | __ _ _ __  _   _  __ _| |
 | '__| / __|/ __|______\ \ / / | |\/| |/ _` | '_ \| | | |/ _` | |
 | |  | \__ \ (__        \ V /  | |  | | (_| | | | | |_| | (_| | |
 |_|  |_|___/\___|        \_/   |_|  |_|\__,_|_| |_|\__,_|\__,_|_|

for more details.

If it is the second case, remember:
* The machine is always right!
* Every line of untested code is always wrong!

[src/cpu/cpu-exec.c:144 cpu_exec] nemu: ABORT at pc = 0x80000044
...
```

当程序直行到非法0指令的时候：
```asm
8000002c <main>:
8000002c:	ff010113          	addi	sp,sp,-16
80000030:	00112623          	sw	ra,12(sp)
80000034:	00288893          	addi	a7,a7,2
80000038:	00000797          	auipc	a5,0x0
8000003c:	fd878793          	addi	a5,a5,-40 # 80000010 <handler>
80000040:	30579073          	csrw	mtvec,a5
80000044:	00000000          	.word	0x00000000
80000048:	00000517          	auipc	a0,0x0
8000004c:	46450513          	addi	a0,a0,1124 # 800004ac <_etext+0x18>
80000050:	39c000ef          	jal	ra,800003ec <printf>
80000054:	0000006f          	j	80000054 <main+0x28>
```

nemu 模拟器的处理是老办法，已经很熟悉了。

但是我想要让这个程序在 nemu 上运行且能正确处理指令异常的话，需要给 nemu 加一点东西。

### 指令异常处理

首先，我们需要将原来的错误指令处理函数修改成这样：
```C
    INSTPAT("0000000 00001 00000 000 00000 11100 11", ebreak, N,
            NEMUTRAP(s->pc, R(10)));  // R(10) is $a0
    // INSTPAT("??????? ????? ????? ??? ????? ????? ??", inv, N,
    // INV(s->pc));
    INSTPAT("??????? ????? ????? ??? ????? ????? ??", ecall, I,
            ECALL(s->dnpc));  // 这里如果指令出现错误，就触发异常
    INSTPAT_END();
```

这样，当接受到非法指令的时候，就会发起异常。

然后我们要需要模拟硬件保存状态寄存器：

```C
word_t isa_raise_intr(word_t NO, vaddr_t epc) {
    // 实现 etrace
    IFDEF(CONFIG_ETRACE, etrace(NO, epc));
    // 在这里决定是否对 pc + 4：
    // 如果是缺页中断: 不加 4；
    // 如果是陷入指令，那么这条指令执行完了自然要执行下一跳指令，+4
    // 如果这里是指令异常，那么应该不 +4，以确保其指向正确的当前指令
    // 非法指令异常号是 2
    cpu.csr.mcause = NO;
    if (NO == 2) {
        cpu.csr.mepc = epc;
    } else {
        cpu.csr.mepc = epc + 4;
    }
    return cpu.csr.mtvec;
}
```

那么这个 NO 怎么来的呢？如果是在 C语言层次发起的异常，比如通过系统调用，那么会自动的调用这yield()：
```C
void yield() {
#ifdef __riscv_e
    asm volatile("li a5, -1; ecall");
#else
    asm volatile("li a7, 11; ecall");
#endif
}
```
但是上面的例子并不是通过系统调用引发的异常，那应该怎么办呢？我们只需要把上面的程序改造一下，手动给 a7 中写入 2 这个异常类型就好了：

```C
int main() {
    asm volatile("addi a7, a7, 2");
    asm volatile("csrw mtvec, %0" : : "r"(handler));
    asm volatile(".word 0");  // illegal instruction
    printf("I am alive!\n");
    while (1)
        ;
}
```
这下就成功了:
```bash
...
bili/异常处理/build/simple-rv-handler-riscv32-nemu.bin, size = 1301
[src/monitor/monitor.c:30 welcome] Trace: OFF
[src/monitor/monitor.c:36 welcome] Build time: 21:43:22, Nov 21 2024
Welcome to riscv32-NEMU!
For help, type "help"
[src/monitor/monitor.c:40 welcome] Exercise: Please remove me in the source code and compile NEMU again.
exception at mepc = 0x80000044
```
这样就大功告成了，我们的 mepc 正确指向了错误的指令（不能+4）：

```asm
8000002c <main>:
8000002c:	ff010113          	addi	sp,sp,-16
80000030:	00112623          	sw	ra,12(sp)
80000034:	00288893          	addi	a7,a7,2
80000038:	00000797          	auipc	a5,0x0
8000003c:	fd878793          	addi	a5,a5,-40 # 80000010 <handler>
80000040:	30579073          	csrw	mtvec,a5
80000044:	00000000          	.word	0x00000000
80000048:	00000517          	auipc	a0,0x0
8000004c:	46450513          	addi	a0,a0,1124 # 800004ac <_etext+0x18>
80000050:	39c000ef          	jal	ra,800003ec <printf>
80000054:	0000006f          	j	80000054 <main+0x28>
```

解释一下这个程序，就是当遇到非法指令的时候，cpu 应该会自动发起异常，然后由 handler 负责解决异常。

我在硬件层面模拟了硬件过程（设置 mcause、mepc、mtvec），现在对上面的程序进行修改，我们希望能读出来这三个寄存器的值：

```C
#include <klib.h>
void handler() {
    uintptr_t mepc, mcause, mtvec;
    asm volatile("csrr %0, mepc" : "=r"(mepc));
    asm volatile("csrr %0, mcause" : "=r"(mcause));
    asm volatile("csrr %0, mtvec" : "=r"(mtvec));
    printf("exception at mepc = %p\n", mepc);
    printf("exception mcause = %p\n", mcause);
    printf("exception mtvec = %p\n", mtvec);
     while (1);
}

int main() {
    asm volatile("addi a7, a7, 2");
    asm volatile("csrw mtvec, %0" : : "r"(handler));
    asm volatile(".word 0");  // illegal instruction
    printf("I am alive!\n");
    while (1)
        ;
}
```
编译运行：

```bash
make ARCH=riscv32-nemu -j8 run
...

exception at mepc = 0x80000074
exception mcause = 0x2
exception mtvec = 0x80000010

```
对比它的部分反汇编查看：
```asm

80000010 <handler>:
80000010:	ff010113          	addi	sp,sp,-16
80000014:	00112623          	sw	ra,12(sp)
80000018:	00812423          	sw	s0,8(sp)
8000001c:	00912223          	sw	s1,4(sp)
80000020:	341025f3          	csrr	a1,mepc
80000024:	342024f3          	csrr	s1,mcause
80000028:	30502473          	csrr	s0,mtvec
...

8000005c <main>:
...
8000006c:	fa878793          	addi	a5,a5,-88 # 80000010 <handler>
80000070:	30579073          	csrw	mtvec,a5
80000074:	00000000          	.word	0x00000000
80000078:	00000517          	auipc	a0,0x0
...
```

也是符合预期的。

这个是指令异常，指令是 0 指令。假设这个指令问题不大，不会对程序造成任何影响，那么我需要让这个异常处理程序返回到 0 指令的下一条指令，因此这里的 mepc 应该+4。是否+4，完全取决于中断类型。

那么怎么返回呢？可以将 mepc 的值打到 pc 中，绝对不可以用 ret 指令来返回，因为 ret 是一个伪指令，它实际上是 jalr。在回复上下文后使用 ret，我们的现场就又遭到了破坏（jalr 会修改寄存器），就无法恢复。

但是我们可以使用 mret，它实际上是将 mepc 拷贝到 pc。

我们可以在程序中这样操作：

```C
#include <klib.h>
void handler() {
    uintptr_t mepc, mcause, mtvec;
    asm volatile("csrr %0, mepc" : "=r"(mepc));
    asm volatile("csrr %0, mcause" : "=r"(mcause));
    asm volatile("csrr %0, mtvec" : "=r"(mtvec));
    printf("exception at mepc = %p\n", mepc);
    printf("exception mcause = %p\n", mcause);
    printf("exception mtvec = %p\n", mtvec);

    if (mcause == 2) {
        asm volatile("csrw mepc, %0;mret" ::"r"(mepc + 4)); //
    }
    while (1)
        ;
}

int main() {
    asm volatile("addi a7, a7, 2");
    asm volatile("csrw mtvec, %0" : : "r"(handler));
    asm volatile(".word 0");  // illegal instruction
    printf("I am alive!\n");
    while (1)
        ;
}
```

当 mcause 为 2 的时候，我们再执行两条指令，分别是修改 mepc，以及 mret。编译运行：

```bash
make ARCH=riscv32-nemu -j8 run
...

exception at mepc = 0x80000094
exception mcause = 0x2
exception mtvec = 0x80000010
I am alive!

```

当然这种返回不符合规范，因为它没有保存、恢复上下文。正常的异常处理调用 handler 之前，需要保存现场，然后返回到 handler，然后再恢复现场，最后调用 mret。


## 系统调用

CPU 的特权级（riscv）：
- M: 机器模式
- S: 监管模式
- U: 用户模式

怎么管理呢？不是任何程序都能随便指令危险的指令比如内联汇编中操作 csr 寄存器。这个解决方案是由硬件提供的，之前在 nemu 中跑上面的例子，都是以 M 模式直接跑的，真实环境下是不行的。

比如下面这个例子：

```C
#include <stdio.h>
int main() {
    unsigned long csr;
#ifdef __riscv
    asm volatile("csrr %0, mepc" : "=r"(csr));
#else  // x86
    asm volatile("mov %%cr0, %0" : "=r"(csr));
#endif
    printf("csr = %p\n", csr);
    return 0;
}
```

这个程序不管是 riscv 还是 x86 都在尝试执行非法指令，我们先在 rv64 中跑：

```bash
$ rv64gcc test.c
$ ./a.out
[1]    248701 illegal hardware instruction (core dumped)  ./a.out
```

可以看到，它报错，显示非法指令。继续在 x86 上跑：

```bash
$ gcc test.c
$ ./a.out
[1]    248889 segmentation fault (core dumped)  ./a.out
```

x86 的报错是段错误，也是一样的，因为寄存器地址非法。

修改程序，让其执行U模式下可以执行的指令：

```C
#include <stdio.h>
int main() {
    unsigned long a0;
    asm volatile(
        "li a0, 10\n"
        "mv %0, a0"
        : "=r"(a0)
        :
        : "a0");

    printf("a0 = %x\n", a0);
    return 0;
}
```

运行就很正常：
```bash
$ rv64gcc test.c
$ ./a.out
a0 = a
```



一般规定，资源管理程序放在高特权级（S），这实际上就是操作系统的代码；用户程序运行在低特权级（U），比如普通运算，发起系统调用请求等。

那么怎么发起一个系统调用呢？唯一合法方式：自陷类异常-执行一条无条件触发异常的指令。riscv 提供了 ecall 指令，操作系统从而可以根据这个 ecall 指令，去审查 mcause 判断该异常的来源。比如在 U 模式下发起的 ecall，mcause 为 8；在 S 模式下发起的 ecall，mcause 为 9；在 M 模式下发起 ecall，mcause 为 10。

当发起 ecall 的时候，OS 会检查 mcause，从而判断是否合法；如果合法。就会出处理这个 ecall。具体就是保存上下文、返回到 handler，handler 处理完后恢复现场，然后 mret。

当然，请求系统调用需要给系统传参，告诉操作系统我需要何种服务。最合理的方式是通过寄存器来传参。约定：a7 传递系统调用号码。

一个例子：

```C
#include <stdio.h>
int main() {
  printf("Hello World!\n");
  return 0;
}
```

在 rv64 平台下运行：
```bash
$ rv64gcc hello.c
$ ./a.out
Hello World!
```

我们想知道它在运行的时候的 trace，比如 strace，这时候加上一些参数：

```bash
$ qemu-riscv64 -d strace,trace:guest_user_syscall a.out
...
guest_user_syscall cpu=0x55afcb4bf7e0 num=0x0000000000000040 arg1=0x0000000000000001 arg2=0x00000040000032a0 arg3=0x000000000000000d arg4=0x00000040000032a0 arg5=0x0000000000000000 arg6=0x0000000000000000 arg7=0x0000000000000000 arg8=0x0000000000000000
250892 write(1,0x32a0,13)Hello World!
 = 13
...
```

可以看到，printf 其实就是一个系统调用 write：第一个参数为文件描述符 1，说明是标准输出；第二个参数为缓冲区地址0x32a0，第三个参数为字符长度，刚好是 13。

可以看手册：
```bash
man 2 write
...
NAME
       write - write to a file descriptor

SYNOPSIS
       #include <unistd.h>

       ssize_t write(int fd, const void *buf, size_t count);

DESCRIPTION
       write()  writes up to count bytes from the buffer starting at buf to the file referred to by the file
       descriptor fd.
...

```

通过上面的 trace，可以看到这个系统调用的调用号为 num = 0x40，那么我们就可以通过汇编来发起系统调用了：

```C
int main() {
  asm volatile (
      "li a0, 1\n"
      "mv a1, %0\n"
      "mv a2, %1\n"
      "li a7, 0x40\n"
      "ecall\n"
      : : "r"("Hello World_Hello world!\n"), "r"(26));
  return 0;
}
```

编译运行：

```bash
$ rv64gcc a.c
$ ./a.out
Hello World_Hello world!
```

通过 trace 可以发现，几乎一摸一样：
```bash
guest_user_syscall cpu=0x629597e6b7a0 num=0x0000000000000040 arg1=0x0000000000000001 arg2=0x0000004000000658 arg3=0x000000000000001a arg4=0x0000000000000000 arg5=0x000000000000001a arg6=0x0000004000000658 arg7=0x0000000000000000 arg8=0x0000000000000000
252231 write(1,0x658,26)Hello World_Hello world!
 = 26
```


