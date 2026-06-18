---
title: "RISC-V32 OS 实验（二）：启动最小内核"
date: 2025-03-16 13:03:13
tags:
  - "RISC-V"
  - "OS kernel"
  - "bare metal"
categories:
  - "Operating Systems"
---

## 启动最简单的 OS

这个 OS 超级简单，就是一个裸机程序：

```C
void start_kernel(void)
{
	while (1) {}; // stop here!
}
```
我们只需要将这个程序编译成二进制文件，然后丢给 qemu，就可以跑了。

但是，本节会提到一个大大的问题，这个问题在上一个博客中并没有提到。

## CPU 是多核心的

如果在多核心上启动一个 kernel，会发生什么？

则个程序很简单，打印一串字符：

```C
extern void uart_init(void);
extern void uart_puts(char* s);

void start_kernel(void) {
    uart_init();
    uart_puts("Hello, RVOS!\n");

    while (1) {
    };  // stop here!
}

```

还有启动代码：
```asm
#include "platform.h"
	.equ	STACK_SIZE, 1024

	.global	_start

	.text
_start:
	csrr	t0, mhartid
	mv	tp, t0
	bnez	t0, park
	slli	t0, t0, 10
	la	sp, stacks + STACK_SIZE

	add	sp, sp, t0

	j	start_kernel

park:
	wfi
	j	park

.balign 16
stacks:
	.skip	STACK_SIZE * MAXNUM_CPU

	.end
```

如果给 QEMU 指定了 8 个核心，那么就只能让核心 0 去执行我们的 代码了，不然就会出现混乱。在当前环境下，其它核心都会跳转到 park，只有核心 0 会继续打印，因此是没有任何问题的：

```bash
$ make run
Press Ctrl-A and then X to exit QEMU
------------------------------------
Hello, RVOS!
```

但是如果我们把 `bnez	t0, park` 注释掉，就会发生错误：

```bash
$ make run
Press Ctrl-A and then X to exit QEMU
------------------------------------
Hello, RVOS!
HelloeHell, RVOS!
llo, RVOS!
llo, RVOS!
Hello, RVOS!
o, RVOS!
Hello, RVOS!
ello, RVOS!
```

8 个核心各自打印自己的，实际上，CPU 在通电的时候，每个核心都是平等的，它们都会从第一条指令各自执行这个 OS。因此如果不加以控制，将会非常混乱：

```bash
$ make debug
Press Ctrl-C and then input 'quit' to exit GDB and QEMU
-------------------------------------------------------
Reading symbols from out/os.elf...
Breakpoint 1 at 0x80000000: file start.S, line 11.
0x00001000 in ?? ()
=> 0x00001000:  97 02 00 00     auipc   t0,0x0

Thread 1 hit Breakpoint 1, _start () at start.S:11
11              csrr    t0, mhartid             # read current hart id
=> 0x80000000 <_start+0>:       f3 22 40 f1     csrr    t0,mhartid
(gdb) si
12              mv      tp, t0                  # keep CPU's hartid in its tp for later usage.
=> 0x80000004 <_start+4>:       13 82 02 00     mv      tp,t0
(gdb) si
[Switching to Thread 1.3]

Thread 3 hit Breakpoint 1, _start () at start.S:11
11              csrr    t0, mhartid             # read current hart id
=> 0x80000000 <_start+0>:       f3 22 40 f1     csrr    t0,mhartid
(gdb) si
12              mv      tp, t0                  # keep CPU's hartid in its tp for later usage.
=> 0x80000004 <_start+4>:       13 82 02 00     mv      tp,t0
(gdb) si
[Switching to Thread 1.8]

Thread 8 hit Breakpoint 1, _start () at start.S:11
11              csrr    t0, mhartid             # read current hart id
=> 0x80000000 <_start+0>:       f3 22 40 f1     csrr    t0,mhartid
(gdb)
```
上面的例子就说明这样的事实。

## uart

可以看到上面的 kernel 打印出了字符。事实上，qemu 不仅仅提供了 cpu 和内存，它还提供了一系列设备，其中就包括 uart 设备。

设备在使用前必须经过配置，之后就可以使用了，比如：

```C

void uart_init() {
    uart_write_reg(IER, 0x00);
    uint8_t lcr = uart_read_reg(LCR);
    uart_write_reg(LCR, lcr | (1 << 7));
    uart_write_reg(DLL, 0x03);
    uart_write_reg(DLM, 0x00);

    lcr = 0;
    uart_write_reg(LCR, lcr | (3 << 0));
}

int uart_putc(char ch) {
    while ((uart_read_reg(LSR) & LSR_TX_IDLE) == 0)
        ;
    return uart_write_reg(THR, ch);
}

void uart_puts(char* s) {
    while (*s) {
        uart_putc(*s++);
    }
}
```

这样，我们访问 uart 串口地址就可以实现输出了。