---
title: 基于 riscv32 的 OS 设计：外部设备中断（uart）
date: 2025-03-23 13:03:13
tags:
    - OS
    - riscv
categories: OS
---

## RISC-V 中断（Interrupt）的分类

一共有三种中断：
- 本地软件中断；
- 本地时钟中断；
- 外部中断；

中断的控制很复杂，具有多个级别的开关，以确保系统能够灵活、高效地处理各种中断：

**1. 设备级别：**

* 这是中断控制的最底层。每个外围设备（例如，UART、SPI、GPIO）都有自己的中断使能位。
* 通过设置这些位，可以控制设备是否能够产生中断请求。
* 如果设备级别的中断被禁用，即使设备产生了中断事件，也不会向上一级的中断控制器（PLIC）发送中断请求。

**2. PLIC 级别：**

* PLIC（Platform-Level Interrupt Controller，平台级中断控制器）是 RISC-V 系统中负责管理和分发外部中断的核心组件。
* PLIC 具有以下几个关键的控制级别：
    * **中断使能位：**
        * PLIC 允许为每个外部中断源独立设置使能位。
        * 只有当 PLIC 中的相应中断使能位被设置时，来自该中断源的中断请求才会被传递给 CPU。
    * **中断优先级：**
        * PLIC 允许为每个中断源分配优先级。
        * 当多个中断同时发生时，PLIC 根据优先级决定哪个中断先被处理。
    * **中断目标：**
        * PLIC 允许将中断路由到特定的 CPU 核心（hart）。
        * 这使得系统能够灵活地将中断分配给不同的 CPU 核心进行处理。

**3. CPU 全局级别：**

* RISC-V CPU 具有全局中断使能位，通常位于 `mstatus` 寄存器中。
* 当全局中断使能位被禁用时，CPU 将忽略所有来自 PLIC 的中断请求。
* 全局中断使能位通常用于在执行关键代码段时临时禁用中断，以确保代码的原子性。

**4. CPU 局部级别：**

* 除了全局中断使能位之外，RISC-V CPU 还具有局部中断使能位，用于控制特定类型的中断。
* 例如，`mie` 寄存器用于控制机器模式下的中断使能，`sie` 寄存器用于控制超级用户模式下的中断使能。
* 通过设置这些局部中断使能位，可以更细粒度地控制 CPU 响应哪些类型的中断。


## RISC-V 中断编程中涉及的寄存器

### mstatus（Machine Status）

用于跟踪和控制 hart 的当前操作状态（特别地，包括关闭和打开全局中断）。

### mie、mip

mie(Machine Interrupt Enable) ：打开（1）或者关闭（0）M/S/U 模式下对应的 External/Timer/Software 中断。

mip(Machine Interrupt Pending) ：获取当前 M/S/U 模式下对应的 External/Timer/Software 中断是否发生。


## RISC-V 中断处理流程

中断发生时 hart 自动执行如下状态转换：

- 把 mstatus 的 MIE 值复制到 MPIE 中，清除 mstatus 中的 MIE 标
志位，效果是中断被禁止。
- 当前的 PC 的下一条指令地址被复制到 mepc 中，同时 PC 被设置为
mtvec。注意如果我们设置 mtvec.MODE = vetcored，PC =
mtvec.BASE + 4 × exception-code。
- 根据 interrupt 的种类设置 mcause，并根据需要为 mtval 设置附加
信息。
- 将 trap 发生之前的权限模式保存在 mstatus 的 MPP 域中，再把
hart 权限模式更改为 M。

退出中断 ：编程调用 MRET 指令。

## PLIC 介绍

PLIC 编程接口 - 寄存器
- RISC-V 规范规定，PLIC 的寄存器编址采用内存映射（memory map）方式。每个寄存器的宽度为 32-bit。
- 具体寄存器编址采用 base + offset 的格式，且 base 由各个特定platform 自己定义。针对 QEMU-virt，其 PLIC 的设计参考了 FU540-C000，base 为 0x0c000000。

- 每个 PLIC 中断源对应一个寄存器，用于配置该中断源的优先级。
- QEMU-virt 支持 7 个优先级。 0 表示对该中断源禁用中断。其余优先级，1 最低，7 最高。
- 如果两个中断源优先级相同，则根据中断源的 ID 值进一步区分优先级，ID 值越小的优先级越高。

- 每个 PLIC 包含 2 个 32 位的 Pending 寄存器，每一个 bit 对应
一个中断源，如果为 1 表示该中断源上发生了中断（进入 Pending 状态），有待 hart 处理，否则表示该中断源上当前无中断发生。
- Pending 寄存器中断的 Pending 状态可以通过 claim 方式清除。
- 第一个 Pending 寄存器的第 0 位对应不存在的 0 号中断源，其
值永远为 0。

- 每个 Hart 有 2 个 Enable 寄存器 （Enable1 和 Enable2）用
于针对该 Hart 启动或者关闭某路中断源。
- 每个中断源对应 Enable 寄存器的一个 bit，其中 Enable1 负责控制 1 ~ 31 号中断源；Enable2 负责控制 32 ~ 53 号中断源。 将对应的 bit 位设置为 1 表示使能该中断源，否则表示关闭该中断源。

- 每个 Hart 有 1 个 Threshold 寄存器用于设置中断优先级的阈值。
- 所有小于或者等于（<=）该阈值的中断源即使发生了也会被 PLIC 丢弃。特别地，当阈值为 0 时允许所有中断源上发生的中断；当阈值为 7 时丢弃所有中断源上发生的中断。

- Claim 和 Complete 是同一个寄存器，每个 Hart 一个。
- 对该寄存器执行读操作称之为 Claim，即获取当前发生的最高优先级的中断源 ID。Claim 成功后会清除对应的 Pending 位。
- 对该寄存器执行写操作称之为 Complete。所谓 Complete 指的是通知 PLIC 对该路中断的处理已经结束。

## 采用中断方式从 UART 实现输入
本实验尝试使用 uart 进行字符输入，这是一种常见的中断源。

首先我们要打开 uart 的中断写入（以 CPU 为参照）：

```C
    uint8_t ier = uart_read_reg(IER);
    uart_write_reg(IER, ier | (1 << 0));
```

这样，我们在键盘输入字符的时候，uart 接受到字符后，设备就会以中断的方式通知 CPU。

接着，我们设置 PLIC：

```C
#include "os.h"

void plic_init(void) {
    int hart = r_tp();

    /*
     * Set priority for UART0.
     *
     * Each PLIC interrupt source can be assigned a priority by writing
     * to its 32-bit memory-mapped priority register.
     * The QEMU-virt (the same as FU540-C000) supports 7 levels of priority.
     * A priority value of 0 is reserved to mean "never interrupt" and
     * effectively disables the interrupt.
     * Priority 1 is the lowest active priority, and priority 7 is the highest.
     * Ties between global interrupts of the same priority are broken by
     * the Interrupt ID; interrupts with the lowest ID have the highest
     * effective priority.
     */
    *(uint32_t*)PLIC_PRIORITY(UART0_IRQ) = 1;

    /*
     * Enable UART0
     *
     * Each global interrupt can be enabled by setting the corresponding
     * bit in the enables registers.
     */
    *(uint32_t*)PLIC_MENABLE(hart, UART0_IRQ) = (1 << (UART0_IRQ % 32));

    /*
     * Set priority threshold for UART0.
     *
     * PLIC will mask all interrupts of a priority less than or equal to
     * threshold. Maximum threshold is 7. For example, a threshold value of zero
     * permits all interrupts with non-zero priority, whereas a value of 7 masks
     * all interrupts. Notice, the threshold is global for PLIC, not for each
     * interrupt source.
     */
    *(uint32_t*)PLIC_MTHRESHOLD(hart) = 0;

    /* enable machine-mode external interrupts. */
    w_mie(r_mie() | MIE_MEIE);

    /* enable machine-mode global interrupts. */
    w_mstatus(r_mstatus() | MSTATUS_MIE);
}

/*
 * DESCRIPTION:
 *	Query the PLIC what interrupt we should serve.
 *	Perform an interrupt claim by reading the claim register, which
 *	returns the ID of the highest-priority pending interrupt or zero if
 *there is no pending interrupt. A successful claim also atomically clears the
 *corresponding pending bit on the interrupt source. RETURN VALUE: the ID of the
 *highest-priority pending interrupt or zero if there is no pending interrupt.
 */
int plic_claim(void) {
    int hart = r_tp();
    int irq = *(uint32_t*)PLIC_MCLAIM(hart);
    return irq;
}

/*
 * DESCRIPTION:
 *	Writing the interrupt ID it received from the claim (irq) to the
 *	complete register would signal the PLIC we've served this IRQ.
 *	The PLIC does not check whether the completion ID is the same as the
 *	last claim ID for that target. If the completion ID does not match an
 *	interrupt source that is currently enabled for the target, the
 *completion is silently ignored. RETURN VALUE: none
 */
void plic_complete(int irq) {
    int hart = r_tp();
    *(uint32_t*)PLIC_MCOMPLETE(hart) = irq;
}

```

当 uart 接受到字符后，会发起中断：

```C
reg_t trap_handler(reg_t epc, reg_t cause) {
    reg_t return_pc = epc;
    reg_t cause_code = cause & MCAUSE_MASK_ECODE;

    if (cause & MCAUSE_MASK_INTERRUPT) {
        /* Asynchronous trap - interrupt */
        switch (cause_code) {
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

CPU 收到中断后，会进入中断处理程序，也就是 trap。由于 uart 是 外部中断，因此 cause 为 11。之后进入到 external_interrupt_handler：

```C

void external_interrupt_handler() {
    int irq = plic_claim();

    if (irq == UART0_IRQ) {
        uart_isr();
    } else if (irq) {
        printf("unexpected interrupt irq = %d\n", irq);
    }

    if (irq) {
        plic_complete(irq);
    }
}
```

这个函数专门处理外部中断，如果是 uart 中断，则：

```C
int uart_getc(void) {
    while (0 == (uart_read_reg(LSR) & LSR_RX_READY))
        ;
    return uart_read_reg(RHR);
}

/*
 * handle a uart interrupt, raised because input has arrived, called from
 * trap.c.
 */
void uart_isr(void) {
    uart_putc((char)uart_getc());
    /* add a new line just to look better */
    uart_putc('\n');
}
```

因此，它接受到字符后，又将字符打印到终端了。

运行程序：
```bash
$ make run
Press Ctrl-A and then X to exit QEMU
------------------------------------
Hello, RVOS!
HEAP_START = 0x800070ec(aligned to 0x80008000), HEAP_SIZE = 0x07ff8f14,
num of reserved pages = 8, num of pages to be allocated for heap = 32752
TEXT:   0x80000000 -> 0x800036c8
RODATA: 0x800036c8 -> 0x80003a02
DATA:   0x80004000 -> 0x80004004
BSS:    0x80004010 -> 0x800070ec
HEAP:   0x80010000 -> 0x88000000
Task 0: Created!
Task 0: Running...
Task 1: Created!
Task 1: Running...
Task 0: Running...
external interruption!
d
Task 1: Running...
external interruption!

```
这就是外部中断的大概流程。核心就是 plic_claim、plic_complete。













