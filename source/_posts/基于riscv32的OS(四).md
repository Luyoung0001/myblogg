---
title: 基于 riscv32 的 OS 设计：切换上下文
date: 2025-03-19 13:03:13
tags:
    - OS
    - riscv
categories: OS
---

## Context

OS 中实际上并没有什么线程、进程，不过是一个个不同的上下文而已。

上下文非常重要，任务 A 和任务 B 的切换，核心就是保存 A 的上下文、恢复 B 的上下文。

在之前的 MIPS yieldOS 中，已经实现过上下文以及上下文的切换了，本质都是差不多的。

```C
/* task management */
struct context {
	/* ignore x0 */
	reg_t ra;
	reg_t sp;
	reg_t gp;
	reg_t tp;
	reg_t t0;
	reg_t t1;
	reg_t t2;
	reg_t s0;
	reg_t s1;
	reg_t a0;
	reg_t a1;
	reg_t a2;
	reg_t a3;
	reg_t a4;
	reg_t a5;
	reg_t a6;
	reg_t a7;
	reg_t s2;
	reg_t s3;
	reg_t s4;
	reg_t s5;
	reg_t s6;
	reg_t s7;
	reg_t s8;
	reg_t s9;
	reg_t s10;
	reg_t s11;
	reg_t t3;
	reg_t t4;
	reg_t t5;
	reg_t t6;
};
```

## 保存上下文

上下文本质上是当前hart的环境，也就是寄存器的值。但是不是所有的寄存器都是上下文的组成部分。

在 riscv 中，某些寄存器在上下文切换过程中值保持不变，因此就没有必要保存这些寄存器。这样的寄存器有 X0、gp、tp。

X0 是 0 寄存器，它是不变的。gp 是全局寄存器，通常指向全局数据区域，在大多数情况下，它在整个程序执行期间保持不变。至于 tp，它保存着 hartid，这是一个全局值，只要不涉及处理器核心的切换，在上下文切换期间通常不会改变。因此，没有必要在每次上下文切换时保存和恢复 tp。

另外，我们必须了解的是关于保存和恢复和回复上下文的事情：
- 我们用 mscratch 来指向当前 task 的上下文；
- 接着用 t6 寄存器 保存 mscratch 后，让它作为交换的 base；
- mscratch 是 csr 寄存器，它不能被当做 base，因为 load、store 指令没办法操作它。

因此，首先我们得把 mscratch 的值交换到 t6 中，接着以 t6 为 base，保存寄存器 `reg_save t6`：

```asm
.macro reg_save base
	STORE ra,   0*SIZE_REG(\base)
	STORE sp,   1*SIZE_REG(\base)
	STORE t0,   4*SIZE_REG(\base)
	STORE t1,   5*SIZE_REG(\base)
	STORE t2,   6*SIZE_REG(\base)
	STORE s0,   7*SIZE_REG(\base)
	STORE s1,   8*SIZE_REG(\base)
	STORE a0,   9*SIZE_REG(\base)
	STORE a1,  10*SIZE_REG(\base)
	STORE a2,  11*SIZE_REG(\base)
	STORE a3,  12*SIZE_REG(\base)
	STORE a4,  13*SIZE_REG(\base)
	STORE a5,  14*SIZE_REG(\base)
	STORE a6,  15*SIZE_REG(\base)
	STORE a7,  16*SIZE_REG(\base)
	STORE s2,  17*SIZE_REG(\base)
	STORE s3,  18*SIZE_REG(\base)
	STORE s4,  19*SIZE_REG(\base)
	STORE s5,  20*SIZE_REG(\base)
	STORE s6,  21*SIZE_REG(\base)
	STORE s7,  22*SIZE_REG(\base)
	STORE s8,  23*SIZE_REG(\base)
	STORE s9,  24*SIZE_REG(\base)
	STORE s10, 25*SIZE_REG(\base)
	STORE s11, 26*SIZE_REG(\base)
	STORE t3,  27*SIZE_REG(\base)
	STORE t4,  28*SIZE_REG(\base)
	STORE t5,  29*SIZE_REG(\base)
	# we don't save t6 here, due to we have used
	# it as base, we have to save t6 in an extra step
	# outside of reg_save
.endm
```

保存好了以后，就可以在切换时使用这些寄存器了，因为还没有保存 t6，而且 t6 原来的值还在 mscratch 中。

因此，需要将 t6 保存到 t5 中；然后将 mscratch 保存到 t6 中。 这样就恢复了 t6，接着将 t6 保存到上下文中。而 t5 就是刚下的 base。

这样就完美得保存了所有的上下文：

```asm
    mv	t5, t6		# t5 points to the context of current task
	csrr	t6, mscratch		# read t6 back from mscratch
	STORE	t6, 30*SIZE_REG(t5)	# save t6 with t5 as base
```

## 恢复上下文

在 riscv 中，a0 保存的是函数的返回值或者返回的参数。因此，当调度器调度的时候：

```C
void schedule(){
	struct context *next = &ctx_task;
	switch_to(next);
}
```
因此新的上下文的地址就保存在 a0 中。

因此，需要继续进一步将 a0 的值加载到 mscratch 中。然后将 a0 保存到 t6 中，并以 t6 为 base，恢复上下文：

```asm
	csrw	mscratch, a0
	mv	t6, a0
	reg_restore t6
```

恢复的时候，会将所有的寄存器中的值更新，包括 t6：

```asm
.macro reg_restore base
	LOAD ra,   0*SIZE_REG(\base)
	LOAD sp,   1*SIZE_REG(\base)
	LOAD t0,   4*SIZE_REG(\base)
	LOAD t1,   5*SIZE_REG(\base)
	LOAD t2,   6*SIZE_REG(\base)
	LOAD s0,   7*SIZE_REG(\base)
	LOAD s1,   8*SIZE_REG(\base)
	LOAD a0,   9*SIZE_REG(\base)
	LOAD a1,  10*SIZE_REG(\base)
	LOAD a2,  11*SIZE_REG(\base)
	LOAD a3,  12*SIZE_REG(\base)
	LOAD a4,  13*SIZE_REG(\base)
	LOAD a5,  14*SIZE_REG(\base)
	LOAD a6,  15*SIZE_REG(\base)
	LOAD a7,  16*SIZE_REG(\base)
	LOAD s2,  17*SIZE_REG(\base)
	LOAD s3,  18*SIZE_REG(\base)
	LOAD s4,  19*SIZE_REG(\base)
	LOAD s5,  20*SIZE_REG(\base)
	LOAD s6,  21*SIZE_REG(\base)
	LOAD s7,  22*SIZE_REG(\base)
	LOAD s8,  23*SIZE_REG(\base)
	LOAD s9,  24*SIZE_REG(\base)
	LOAD s10, 25*SIZE_REG(\base)
	LOAD s11, 26*SIZE_REG(\base)
	LOAD t3,  27*SIZE_REG(\base)
	LOAD t4,  28*SIZE_REG(\base)
	LOAD t5,  29*SIZE_REG(\base)
	LOAD t6,  30*SIZE_REG(\base)
.endm
```

接着调用 ret 指令返回到新的 PC。

整个过程的代码如下：

```asm
#define LOAD		lw
#define STORE		sw
#define SIZE_REG	4

.macro reg_save base
	STORE ra,   0*SIZE_REG(\base)
	STORE sp,   1*SIZE_REG(\base)
	STORE t0,   4*SIZE_REG(\base)
	STORE t1,   5*SIZE_REG(\base)
	STORE t2,   6*SIZE_REG(\base)
	STORE s0,   7*SIZE_REG(\base)
	STORE s1,   8*SIZE_REG(\base)
	STORE a0,   9*SIZE_REG(\base)
	STORE a1,  10*SIZE_REG(\base)
	STORE a2,  11*SIZE_REG(\base)
	STORE a3,  12*SIZE_REG(\base)
	STORE a4,  13*SIZE_REG(\base)
	STORE a5,  14*SIZE_REG(\base)
	STORE a6,  15*SIZE_REG(\base)
	STORE a7,  16*SIZE_REG(\base)
	STORE s2,  17*SIZE_REG(\base)
	STORE s3,  18*SIZE_REG(\base)
	STORE s4,  19*SIZE_REG(\base)
	STORE s5,  20*SIZE_REG(\base)
	STORE s6,  21*SIZE_REG(\base)
	STORE s7,  22*SIZE_REG(\base)
	STORE s8,  23*SIZE_REG(\base)
	STORE s9,  24*SIZE_REG(\base)
	STORE s10, 25*SIZE_REG(\base)
	STORE s11, 26*SIZE_REG(\base)
	STORE t3,  27*SIZE_REG(\base)
	STORE t4,  28*SIZE_REG(\base)
	STORE t5,  29*SIZE_REG(\base)
.endm


.macro reg_restore base
	LOAD ra,   0*SIZE_REG(\base)
	LOAD sp,   1*SIZE_REG(\base)
	LOAD t0,   4*SIZE_REG(\base)
	LOAD t1,   5*SIZE_REG(\base)
	LOAD t2,   6*SIZE_REG(\base)
	LOAD s0,   7*SIZE_REG(\base)
	LOAD s1,   8*SIZE_REG(\base)
	LOAD a0,   9*SIZE_REG(\base)
	LOAD a1,  10*SIZE_REG(\base)
	LOAD a2,  11*SIZE_REG(\base)
	LOAD a3,  12*SIZE_REG(\base)
	LOAD a4,  13*SIZE_REG(\base)
	LOAD a5,  14*SIZE_REG(\base)
	LOAD a6,  15*SIZE_REG(\base)
	LOAD a7,  16*SIZE_REG(\base)
	LOAD s2,  17*SIZE_REG(\base)
	LOAD s3,  18*SIZE_REG(\base)
	LOAD s4,  19*SIZE_REG(\base)
	LOAD s5,  20*SIZE_REG(\base)
	LOAD s6,  21*SIZE_REG(\base)
	LOAD s7,  22*SIZE_REG(\base)
	LOAD s8,  23*SIZE_REG(\base)
	LOAD s9,  24*SIZE_REG(\base)
	LOAD s10, 25*SIZE_REG(\base)
	LOAD s11, 26*SIZE_REG(\base)
	LOAD t3,  27*SIZE_REG(\base)
	LOAD t4,  28*SIZE_REG(\base)
	LOAD t5,  29*SIZE_REG(\base)
	LOAD t6,  30*SIZE_REG(\base)
.endm


.text

.globl switch_to
.balign 4
switch_to:
	csrrw	t6, mscratch, t6
	beqz	t6, 1f
	reg_save t6


	mv	t5, t6
	csrr	t6, mscratch
	STORE	t6, 30*SIZE_REG(t5)

1:
	csrw	mscratch, a0

	mv	t6, a0
	reg_restore t6
	ret
.end
```

上面的 ret 指令其实是 `jalr x0, x1, 0`，也就是说，它会返回到 x1 也就是 ra。如果设计过 riscv CPU，就会知道其实它的硬件逻辑大概是：

```C
    INSTPAT(
        "??????? ????? ????? 000 ????? 11001 11", jalr, I,
        s->dnpc = (src1 + imm) & ~(word_t)1;
```
而且这里也没有很复杂的 csr 寄存器访问，也就是一个 mscratch。



## 新的上下文初始化以及切换

首先需要初始化一个task，这个 task 有必要的栈区、以及返回地址（ret 指令必须从这返回）。

还得有一个上下文，这是一个全局变量，保存在 OS 的栈区：

```C
uint8_t __attribute__((aligned(16))) task_stack[STACK_SIZE];
struct context ctx_task;

static void w_mscratch(reg_t x) {
    asm volatile("csrw mscratch, %0" : : "r"(x));
}

void user_task0(void);
void sched_init() {
    w_mscratch(0);

    ctx_task.sp = (reg_t)&task_stack[STACK_SIZE];
    ctx_task.ra = (reg_t)user_task0;
}
```

接着就可以调用 schedule 了：

```C
void schedule() {
    struct context* next = &ctx_task;
    switch_to(next);
}
```

至于 kernel：

```C
#include "os.h"
extern void uart_init(void);
extern void page_init(void);
extern void sched_init(void);
extern void schedule(void);

void start_kernel(void)
{
	uart_init();
	uart_puts("Hello, RVOS!\n");

	page_init();

	sched_init();

	schedule();

	uart_puts("Would not go here!\n");
	while (1) {}; // stop here!
}
```

就可以运行了。可以预见的是，它会不断打印 `Task 0: Running...`：

```bash
Hello, RVOS!
HEAP_START = 0x8000487c(aligned to 0x80005000), HEAP_SIZE = 0x07ffb784,
num of reserved pages = 8, num of pages to be allocated for heap = 32755
TEXT:   0x80000000 -> 0x80002f70
RODATA: 0x80002f70 -> 0x8000316c
DATA:   0x80004000 -> 0x80004000
BSS:    0x80004000 -> 0x8000487c
HEAP:   0x8000d000 -> 0x88000000
Task 0: Created!
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
```

这就是一个在启动后切到特定 task 的最简单的 OS 了。实现了切换上下文，那么跑多任务就水到渠成了。



















