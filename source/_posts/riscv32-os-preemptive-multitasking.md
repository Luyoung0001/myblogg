---
title: "RISC-V32 OS 实验（九）：抢占式多任务"
date: 2025-03-25 13:03:13
tags:
  - "RISC-V"
  - "preemption"
  - "scheduler"
categories:
  - "Operating Systems"
---

## 前言

关于任务如何切换，可以分为三种：
- 抢占式：任务自己不能决定是否让出，由外部原因引起；
- 协作式：任务自己放弃 CPU；
- 兼容协作式：抢占式任务的同时，自己还可以自愿放弃。

关于协作式，前面已经提到了。task0 通过 yield 来放弃当前 CPU。

## 与定时器配合

抢占式可以基于时钟中断来实现，通过不断地产生时钟中断来做到任务切换。

思路大致为，当时钟中断到达的时候，进入到 trap_vector，接着调用 trap_handler。 trap_handler 对时钟中断进行处理，处理的过程中直接调用 switch_to，这样就可以了吗？

没有这么简单。

之前的 switch_to 很简单，它由 yield 中的 schedule 函数引发。

switch_to 函数保存上下文后，切换到新的上下文，然后通过 jalr 指令返回到新的上下文（借助寄存器 ra）。这显然不怎么“合法”。合法的切换上下文最后应该由 mret 返回。但是 mret 返回的话，依赖的是 mepc 这个寄存器，因此在 mret 之前，得把 context 中的 mepc 恢复到 mepc。这就意味着，保存上下文的时候，得保存当前的 PC 的下一条指令地址（也就是硬件自动保存的 mepc ）到上下文中！

这个和之前的简单上下文切换的关键不同点是什么？

很明显，之前的上下文根本不需要记住 mepc，因为是主动放弃的，下次回来的时候只需要在 context 中持有一个 ra 就行了：

```C
int task_create(void (*start_routin)(void)) {
    if (_top < MAX_TASKS) {
        ctx_tasks[_top].sp = (reg_t)&task_stack[_top][STACK_SIZE];
        ctx_tasks[_top].ra = (reg_t)start_routin;
        _top++;
        return 0;
    } else {
        return -1;
    }
}
```

但是如果是抢占式的，任务自己根本不知道下次回来的地址是什么，只能依靠 trap_vector 把 mepc 保存到上下文中以便恢复。因此得修改上下文结构体：

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
	// upon is trap frame

	// save the pc to run in next schedule cycle
	reg_t pc; // offset: 31 * sizeof(reg_t)
};
```

了解了这些，就可以改造 trap_vector 了。


## 修改 trap_vector

首先保存上下文：

```asm
    csrrw	t6, mscratch, t6
    reg_save t6
    mv	t5, t6
    csrr	t6, mscratch
    STORE	t6, 30*SIZE_REG(t5)
```

这时候，还有一个新的上下文，那就是 mepc：

```asm
    csrr	a0, mepc
    STORE	a0, 31*SIZE_REG(t5)
    # 恢复 mscratch
    csrw	mscratch, t5
```

调用 trap_handler：

```asm
    csrr	a0, mepc
    csrr	a1, mcause
    call	trap_handler
```

我们可能希望 trap_handler 执行完后，从这里返回。但是，事实上由于这里发生的是时钟中断，并且进行了新的任务调度：

```C
reg_t trap_handler(reg_t epc, reg_t cause) {
    reg_t return_pc = epc;
    reg_t cause_code = cause & MCAUSE_MASK_ECODE;

    if (cause & MCAUSE_MASK_INTERRUPT) {
        /* Asynchronous trap - interrupt */
        switch (cause_code) {
            case 3:
                uart_puts("software interruption!\n");
                /*
                 * acknowledge the software interrupt by clearing
                 * the MSIP bit in mip.
                 */
                int id = r_mhartid();
                *(uint32_t*)CLINT_MSIP(id) = 0;

                schedule();

                break;
            case 7:
                uart_puts("timer interruption!\n");
                timer_handler();
                break;
            case 11:
                uart_puts("external interruption!\n");
                external_interrupt_handler();
                break;
            default:
                printf("Unknown async exception! Code = %ld\n", cause_code);
                break;
        }
    } else {
        /* Synchronous trap - exception */
        printf("Sync exceptions! Code = %ld\n", cause_code);
        panic("OOPS! What can I do!");
        // return_pc += 4;
    }

    return return_pc;
}
```

至于怎么处理，就看 time_handler()：

```C
void timer_handler() {
    _tick++;
    printf("tick: %d\n", _tick);

    timer_load(TIMER_INTERVAL);

    schedule();
}
```

可以看到，这里调用了 schedule，而 schedule 调用了 switch_to。我们希望 switch_to 通过 mret 返回而不是 ret (jalr)。继续改造 switch_to。

当 schedule 结束后，它会返回一个新的上下文，这个上下文保存到 a0 中，因此我们需要设置新的 mscratch 寄存器，接着加载上下文：

```asm
    csrw	mscratch, a0

    # 导出 mepc，mret 依赖于其返回
    LOAD	a1, 31*SIZE_REG(a0)
    csrw	mepc, a1

    mv	t6, a0
    reg_restore t6

    # Do actual context switching.
    # Notice this will enable global interrupt
    mret
```

这样，mret 就能利用 mepc 完美跳到另一个上下文中。当然，这也不会影响其它异常处理过程，trap_vector:

```asm
    # trap_handler will return the return address via a0.
    srw	mepc, a0

    # restore context(registers).
    csrr	t6, mscratch
    reg_restore t6

    # return to whatever we were doing before trap.
    mret
```

## 测试

同样设置两个函数：

```C
void user_task0(void) {
    uart_puts("Task 0: Created!\n");

    // task_yield();
    // uart_puts("Task 0: I'm back!\n");
    while (1) {
        uart_puts("Task 0: Running...\n");
        task_delay(DELAY);
    }
}

void user_task1(void) {
    uart_puts("Task 1: Created!\n");
    while (1) {
        uart_puts("Task 1: Running...\n");
        task_delay(DELAY);
    }
}
```
这两个任务并没有调用 yield。

设置好时钟中断后，就可以看效果了：

```bash
Press Ctrl-A and then X to exit QEMU
------------------------------------
Hello, RVOS!
HEAP_START = 0x80007118(aligned to 0x80008000), HEAP_SIZE = 0x07ff8ee8,
num of reserved pages = 8, num of pages to be allocated for heap = 32752
TEXT:   0x80000000 -> 0x8000388c
RODATA: 0x8000388c -> 0x80003bd2
DATA:   0x80004000 -> 0x80004004
BSS:    0x80004010 -> 0x80007118
HEAP:   0x80010000 -> 0x88000000
Task 0: Created!
Task 0: Running...
timer interruption!
tick: 1
Task 1: Created!
Task 1: Running...
QEMU: Terminated
```

可以看到，一切都符合预期。

## 兼容协作式

如果 task0 主动要放弃 CPU 应该怎么做？继续调用 yield 吗？？？

之前 task0 放弃 CPU 调用 swicth_to 就可以，但是现在的 switch_to 已经被重新定义了，要不要写一个 switch_to_1，task0 调用 switch_to_1 后，实现切换？


当 trap 的时候，hart 会自动将 xIE 设置为 0。这意味着中断将会关闭。这时候上下文处于安全状态。


但是当 task0 不是通过 trap 而是直接进入 switch_to 的时候，中断是开着的，这时候如果来一个 trap，上下文没有保存完，接着跳转到 trap 中，再保存一次。从 trap 返回到 switch_to 后，接着保存上下文，switch_to 表现得就像是 task* 的一部分一样，虽然上下文切换的过程被打断了，但这确实没有什么问题：

```bash
$ make run
Task 0: Created!
software interruption!
Task 1: Created!
timer interruption!
tick: 1
Task 0: Running...
Task 0: Running...
timer interruption!
tick: 2
Task 1: Running...
Task 1: Running...
timer interruption!
tick: 3
Task 0: Running...
Task 0: Running...
timer interruption!
tick: 4
Task 1: Running...
Task 1: Running...
QEMU: Terminated
```

但是我们希望，对于切换上下文的过程，永远从 trap 进去，毕竟保存上下文这么重要的操作，在期间，中断应该关闭。那么 task 应该怎么操作一下，就能引发一个中断呢？

好消息是，CLINT 提供了软中断。

## CLINT 提供的软件中断

每个 hart 拥有一个 MSIP 寄存器。RISCV 规范规定，Machine 模式下的 mip.MSIP 对应到一个 memory-mapped 的控制寄存器。为此 QEMU-virt 提供 MSIP，该 MSIP 寄存器为 32-bit，高 31 位不可用，最低位映射到 mip.MSIP。具体寄存器编址采用 base + offset 的格式，且 base 由各个特定 platform 自己定义。针对 QEMU-virt，其 CLINT 的设计参考了 SFIVE，base 为 0x2000000。

对 MSIP 写入 1 时触发 software interrupt，写入 0 表示对该中断进行应答。

也就是说，我们可以操作 CLINT 中的寄存器来实现操作 CPU 中的 MSIP 位，且它们是同步的。

那么这样，我们就可以从 trap 进入到 switch_to 了：

```C
void sched_init() {
    w_mscratch(0);

    /* enable machine-mode software interrupts. */
    w_mie(r_mie() | MIE_MSIE);
}

// 引发中断
void task_yield() {
    /* trigger a machine-level software interrupt */
    int id = r_mhartid();
    *(uint32_t*)CLINT_MSIP(id) = 1;
}
```
还要在 trap_handler 中，手动将这个中断清零：

```C
reg_t trap_handler(reg_t epc, reg_t cause) {
    reg_t return_pc = epc;
    reg_t cause_code = cause & MCAUSE_MASK_ECODE;

    if (cause & MCAUSE_MASK_INTERRUPT) {
        /* Asynchronous trap - interrupt */
        switch (cause_code) {
            case 3:
                uart_puts("software interruption!\n");
                // 清空软终端
                int id = r_mhartid();
                *(uint32_t*)CLINT_MSIP(id) = 0;
                schedule();
                ...
```

运行效果：

```bash
Task 0: Created!
software interruption!
Task 1: Created!
Task 1: Running...
timer interruption!
tick: 1
Task 0: I'm back!
Task 0: Running...
timer interruption!
tick: 2
Task 1: Running...
QEMU: Terminated
```

可以看到，符合预期。































