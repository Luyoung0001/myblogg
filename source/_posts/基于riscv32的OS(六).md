---
title: 基于 riscv32 的 OS 设计：Trap 和 Exception
date: 2025-03-21 13:03:13
tags:
    - OS
    - riscv
categories: OS
---

## trap 和 exception 相关寄存器

不管是 trap 还是 exception，大致处理流程都是一致的。

### mcause

当 trap 发生时，hart 会设置该寄存器通知我们 trap 发生的原因。最高位 Interrupt 为 1 时标识了当前 trap 为 interrupt，否则是 exception。剩余的 Exception Code 用于标识具体的 interrupt 或者 exception 的种类。

### mtvec（Machine Trap-Vector Base-Address）

BASE：trap 入口函数的基地址，必须保证四字节对齐。MODE：进一步用于控制入口函数的地址配置方式：
- Direct：所有的 exception 和 interrupt 发生后 PC 都跳转到 BASE 指定的地址处。
- Vectored：exception 处理方式同 Direct；但 interrupt 的入口地址以数组方式排列。

### mepc（Machine Exception Program Counter）

- 当 trap 发生时，pc 会被替换为 mtvec 设定的地址，同时 hart 会设置 mepc 为当前指令或者下一条指令的地址，当 我们需要退出 trap 时可以调用特殊的 mret 指令，该指令会将 mepc 中的值恢复到 pc 中（实现返回的效果）。
- 在处理 trap 的程序中我们可以修改 mepc 的值达到改变 mret 返回地址的目的。

### mstatus（Machine Status）

- xIE（x=M/S/U）: 分别用于打开（1）或者关闭（0） M/S/U 模式下的全局中断。当 trap 发生时， hart 会自动将 xIE 设置为 0。
- xPIE（x=M/S/U）:当 trap 发生时用于保存 trap 发生之前的 xIE 值。
- xPP（x=M/S）:当 trap 发生时用于保存 trap 发生之前的权限级别值。
注意没有 UPP。
- 其他标志位涉及内存访问权限、虚拟
内存控制等，暂不考虑。

## riscv trap 的处理流程

主要有四个流程：
- trap 初始化：设置 trap 向量地址；
- trap 的 top half：硬件执行一些操作；
- trap 的 bottom half：软件执行一些操作；
- 从 trap 返回。

### trap 初始化

设置 trap 向量地址。我们通过 kernel 来看：

```C
#include "os.h"
extern void uart_init(void);
extern void page_init(void);
extern void sched_init(void);
extern void schedule(void);
extern void os_main(void);
extern void trap_init(void);

void start_kernel(void) {
    uart_init();
    uart_puts("Hello, RVOS!\n");

    page_init();

    trap_init();

    sched_init();

    os_main();

    schedule();

    uart_puts("Would not go here!\n");
    while (1) {
    };  // stop here!
}

```

首先 start_kernel trap 初始化：

```C
static inline void w_mtvec(reg_t x) {
    asm volatile("csrw mtvec, %0" : : "r"(x));
}

void trap_init() {
    w_mtvec((reg_t)trap_vector);
}
```

这个函数会给 mcsr 寄存器的 mtvec 设置一个 trap 处理函数入口的 base address。

### top half （代码不可见，CPU 硬件自动完成）
trap 发生，hart 自动执行如下状态转换：
- 把 mstatus 的 MIE 值复制到 MPIE 中，清除 mstatus 中的 MIE 标
志位，效果是中断被禁止。
- 设置 mepc ，同时 PC 被设置为 mtvec。（需要注意的是，对于
 exception， mepc 指向导致异常的指令；对于 interrupt，它指向被
中断的指令的下一条指令的位置。）
- 根据 trap 的种类设置 mcause，并根据需要为 mtval 设置附加信息。
- 将 trap 发生之前的权限模式保存在 mstatus 的 MPP 域中，再把
hart 权限模式更改为 M（也就是说无论在任何 Level 下触发 trap，
hart 首先切换到 Machine 模式）。

也就是说，当 trap 发生时，硬件会修改的寄存器有：mstatus、mepc、mcause。

### bottom half (软件做的事情)

#### 保存上下文

这个和前面的保存上下文完全一致：

```asm
trap_vector:
	csrrw	t6, mscratch, t6
	reg_save t6
	mv	t5, t6

	csrr	t6, mscratch
	STORE	t6, 30*SIZE_REG(t5)
	csrw	mscratch, t5
```

#### 调用 trap handler

调用 函数需要传参，riscv 规定将参数保存到 a*寄存器：

```asm
	csrr	a0, mepc
	csrr	a1, mcause
	call	trap_handler
```

#### 从 trap handler 返回

由于硬件已经给相关的寄存器中打入相应的信号了，比如 mepc、mcause 等，因此我们认为底层可靠，只需要按照riscv 约定来写代码。

这个函数就是 trap 处理函数了：

```C

reg_t trap_handler(reg_t epc, reg_t cause) {
    reg_t return_pc = epc;
    reg_t cause_code = cause & MCAUSE_MASK_ECODE;

    if (cause & MCAUSE_MASK_INTERRUPT) {
    // 中断
        switch (cause_code) {
            case 3:
                uart_puts("software interruption!\n");
                break;
            case 7:
                uart_puts("timer interruption!\n");
                break;
            case 11:
                uart_puts("external interruption!\n");
                break;
            default:
                printf("Unknown async exception! Code = %ld\n", cause_code);
                break;
        }
    } else {
    // 异常
        printf("Sync exceptions! Code = %ld\n", cause_code);
        return_pc += 4;
    }

    return return_pc;
}
```

这里的异常处理相当简单，什么都不做直接打印完信息后，直接执行下一跳指令。事实上，这里只是一个演示。异常返回地址应该永远是 epc，意味着 retry。

#### 恢复上下文

返回后，参数在 a0 中：

```asm
    csrw	mepc, a0
	csrr	t6, mscratch
	reg_restore t6
```

将 a0 复制到 mepc 中，接着恢复当前上下文（这个和切换 task 不一样，这trap 处理的过程中，mscratch 保持不变）。


#### 执行 mret

调用 mret 指令实现 trap 的返回。

如果写过 riscv 的 CPU，mret 的语义如下：

```C
#define MRET(dnpc) { dnpc = cpu.csr.mepc; }

```

## 测试

首先创建两个task：

```C
void user_task0(void){
	uart_puts("Task 0: Created!\n");
	while (1) {
		uart_puts("Task 0: Running...\n");

		trap_test();

		task_delay(DELAY);
		task_yield();
	}
}

void user_task1(void){
	uart_puts("Task 1: Created!\n");
	while (1) {
		uart_puts("Task 1: Running...\n");
		task_delay(DELAY);
		task_yield();
	}
}
```

其中 task0 会调用 trap_test，而 trap_test 会触发 类型为异常的 trap：

```C
void trap_test() {
    /*
     * Synchronous exception code = 7
     * Store/AMO access fault
     */
    *(int*)0x00000000 = 100;
    uart_puts("Yeah! I'm return back from trap!\n");
}
```

接着进入 trap 执行流程。处理完后，接着调用 task_yiled 切换到 task1。

```bash
Press Ctrl-A and then X to exit QEMU
------------------------------------
Hello, RVOS!
HEAP_START = 0x800070ec(aligned to 0x80008000), HEAP_SIZE = 0x07ff8f14,
num of reserved pages = 8, num of pages to be allocated for heap = 32752
TEXT:   0x80000000 -> 0x8000339c
RODATA: 0x8000339c -> 0x8000369e
DATA:   0x80004000 -> 0x80004004
BSS:    0x80004010 -> 0x800070ec
HEAP:   0x80010000 -> 0x88000000
Task 0: Created!
Task 0: Running...
Sync exceptions! Code = 7
Yeah! I'm return back from trap!
Task 1: Created!
Task 1: Running...
Task 0: Running...
Sync exceptions! Code = 7
Yeah! I'm return back from trap!
Task 1: Running...

```

但是按照标准的异常处理来说，我们应该尝试再次执行指令：

```C
    else {
        /* Synchronous trap - exception */
        printf("Sync exceptions! Code = %ld\n", cause_code);
    }
return return_pc;
```

这时候：
```bash
...
Sync exceptions! Code = 7
Sync exceptions! Code = 7
Sync exceptions! Code = 7
Sync exceptions! Code = 7
Sync exceptions! Code = 7
Sync exceptions! Code = 7
Sync exceptions! Code = 7
Sync exceptions! Code = 7
Sync exceptions! Code = 7
Sync exceptions! Code = 7
...
```

以上就是 trap 的大概情况了。

























