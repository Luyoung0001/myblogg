---
title: 基于 riscv32 的 OS 设计：系统调用
date: 2025-03-29 13:03:13
tags:
    - OS
    - riscv
categories: OS
---

## 前言

我们希望，用户程序仅仅运行在 U 模式，而 OS 代码统统运行在高权限模式下。这样，比如 tasks 的行为将会受到限制，比如它不能随意的修改 csr 寄存器，甚至修改 PLIC、CLINT、UART 寄存器以危害整个系统。

因此，某些可能会威胁到 OS 稳定运行的操作应该运行在高权限模式下，比如 S、M 模式。而 tasks 想要执行某些高权限操作，只能通过系统调用的方式来进行。

## 系统模式：用户态和内核态

在RISC-V架构中，`mstatus`（机器状态寄存器）的`MPP`（机器先前模式）字段起着至关重要的作用。

**`MPP`字段的作用：**

* **保存先前的特权模式：**
    * `MPP`字段用于保存机器模式（M-mode）下发生异常或中断之前的特权模式。
    * 当CPU从M-mode下的异常或中断处理程序返回时，`MPP`字段的值将被用于恢复到先前的特权模式。
* **支持特权模式的切换：**
    * RISC-V架构支持多种特权模式，包括机器模式（M-mode）、超级用户模式（S-mode）和用户模式（U-mode）。
    * `MPP`字段确保了在M-mode下处理异常或中断后，系统能够正确地返回到之前的特权模式，从而保证了特权模式切换的正确性。
* **异常处理和返回：**
    * 当发生异常或中断时，CPU会自动切换到 M-mode，并保存先前的特权模式到`MPP`字段。
    * 在异常或中断处理程序结束时，`mret`（机器模式返回）指令会读取`MPP`字段的值，并将CPU的特权模式恢复到之前保存的模式。

也就是说，我们希望 mret 返回到 task 的时候，CPU 模式恢复到 U_mode，这意味着我们

## 系统模式的切换

设置 msstatus.MPP 以及 mstatus.MPIE：

```asm
#ifdef CONFIG_SYSCALL
	# Set mstatus.MPP as 0, so we will run in User mode after MRET.
	# No need to set mstatus.MPIE to 1 explicitly, because according to ISA
	# specification: interrupts for M-mode, which is higher than U-mode, are
	# always globally enabled regardless of the setting of the global MIE bit.
	li	t0, 0
	csrc	mstatus, t0
#else
	# Set mstatus.MPP to 3, so we still run in Machine mode after MRET.
	# Set mstatus.MPIE to 1, so MRET will enable the interrupt.
	li	t0, 3 << 11 | 1 << 7
	csrs	mstatus, t0
#endif
```

如果我们打开 CONFIG_SYSCALL，那么在 mret 后，将会回到 U-mode，此时如果我们在 task 中访问特权指令，比如 csr 指令：

```C
void user_task0(void) {
    uart_puts("Task 0: Created!\n");

    unsigned int hid = -1;
    hid = r_mhartid(); // 内联汇编，访问 csr 寄存器
    printf("hart id is %d\n", hid);

    while (1) {
        uart_puts("Task 0: Running... \n");
        task_delay(DELAY);
    }
}
```

它就会引发异常：
```bash
Task 0: Created!
Sync exceptions! Code = 2
panic: OOPS! What can I do!
QEMU: Terminated
```

但是，如果我们设置 mstatus.MPP 让它返回之后依然是机器模式，那就不会引发异常：

```asm
#ifdef CONFIG_SYSCALL
	li	t0, 3 << 11 | 1 << 7
	csrs	mstatus, t0
#else
```

```bash
Task 0: Created!
hart id is 0
Task 0: Running...
Task 0: Running...
Task 0: Running...
timer interruption!
tick: 1
Task 1: Created!
```

因为我们要设计系统调用，因此必须得设置成 mstatus.MPP = 00。

## 系统调用的执行流程

系统调用，就是用户态的函数调用一个 syscall，这个 syscall 在执行的过程中需要通过 trap 进入到核心态来完成，之后再次返回到用户态。

可以通过 ecall 指令来引发一个异常。这里需要认识到，操作系统的设计和 ISA 是解耦的，ecall 可以主动引发一个异常，使得 CPU 进入高权限模式。OS 设计人员可以借助 ecall 来实现 syscall，这这不意味着 syscall 只能通过 ecall 来实现。你要是不嫌延迟以及代码架构差，完全可以调用 syscall 之后，通过 CLINT 的软件中断来实现。当然，ecall 是最好的选择。

ecall 命令用于主动触发异常。根据调用 ecall 的权限级别产生不同的 exception code，异常产生时 epc 寄存器的值存放的是 ecall 指令本身的地址。

ecall 引发异常后，可以根据异常号来确定是什么异常。如果是从 U-mode 发起的 ecall，那么 mcause 的值为 8（可以查阅手册）。这时候就会进入 syscall 执行流程：

```C

reg_t trap_handler(reg_t epc, reg_t cause, struct context* cxt) {
    reg_t return_pc = epc;
    reg_t cause_code = cause & MCAUSE_MASK_ECODE;

    if (cause & MCAUSE_MASK_INTERRUPT) {
        /* Asynchronous trap - interrupt */
        switch (cause_code) {
            case 3:
                uart_puts("software interruption!\n");
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
        switch (cause_code) {
            case 8:
                uart_puts("System call from U-mode!\n");
                do_syscall(cxt);
                return_pc += 4;
                break;
            default:
                panic("OOPS! What can I do!");
                // return_pc += 4;
        }
    }

    return return_pc;
}
```

上面的 do_syscall 定义为：

```C
int sys_gethid(unsigned int* ptr_hid) {
    printf("--> sys_gethid, arg0 = %p\n", ptr_hid);
    if (ptr_hid == NULL) {
        return -1;
    } else {
        *ptr_hid = r_mhartid();
        return 0;
    }
}

void do_syscall(struct context* cxt) {
    uint32_t syscall_num = cxt->a7;
    switch (syscall_num) {
        case SYS_gethid:
            cxt->a0 = sys_gethid((unsigned int*)(cxt->a0));
            break;
        default:
            printf("Unknown syscall no: %d\n", syscall_num);
            cxt->a0 = -1;
    }
    return;
}
```

这样根据系统调用号就完成了系统调用的过程，之后返回。这里需要注意的是，异常有 retry 机制，因此必须手动把 return_pc+4 后返回，不然死循环。

这里有一个问题，系统调用号码是怎么传递的？这就涉及到系统调用的传参了。

## 系统调用的传参

如果整个系统中只有一个系统调用，那就不需传递系统调用号码了，直接进来 do_syscall，传递的函数参数就用 a* 来传递。

我们知道，syscall 是在 trap 过程中执行的，这意味着要保存上下文。进了 trap 之后，所有的函数调用和 u_mode 下的调用没有任何区别。

于是，当进入到 trap 之后，可以将多个参数转递给 trap_handler。

系统调用作为操作系统的对外接口，**由操作系统的实现**负责定义。参考 Linux 的系统调用，RVOS 定义系统调用的传参规则如下：
- 系统调用号放在 a7 中
- 系统调用参数使用 a0 ~ a5
- 返回值使用 a0

其中就包含了上下文的地址 mscratch：

```asm
	# call the C trap handler in trap.c
	csrr	a0, mepc
	csrr	a1, mcause
	csrr	a2, mscratch
	call	trap_handler
```
这下，trap_handler 就可以任意访问 context 以及修改 context 了。处理完后，返回到 trap_vector，恢复上下文之后就完成了整个过程。


## print

我们希望，print 输出字符的过程中不要被异常打断。换句话说，当 task0 执行 print 代码的时候，print 是运行在 trap 过程中的。我们可以让 print 整体作为一个系统调用。尽管它的底层由 printf 实现：

```C
void sys_print(char* str) {
    printf(str);
}
```
事实上，我们应该保证，应该将哪些函数暴露给软件开发者。比如实现了系统调用 print 之后，虽然用户调用 print 之后进入内核执行。但是如果同时也将 printf 暴露给用户，那么用户直接调用 printf 并不会进入内核执行，这应该避免。这里只是一个简单的例子，并没有考虑这么多。

首先定义 print 系统调用号：

```C
#define SYS_gethid	1
#define SYS_print  2
```

系统调用入口：

```asm

#include "syscall.h"
.global gethid
.global print
gethid:
    li a7, SYS_gethid
    ecall
    ret

print:
    li a7, SYS_print
    ecall
    ret
```

进入 trap 之后，进入 do_syacall，执行 print 的底层代码就行了：

```C
void sys_print(char* str) {
    printf(str);
}

void do_syscall(struct context* cxt) {
    uint32_t syscall_num = cxt->a7;
    switch (syscall_num) {
        case SYS_gethid:
            cxt->a0 = sys_gethid((unsigned int*)(cxt->a0));
            break;
        case SYS_print:
            sys_print((char *)cxt->a0);
            break;

        default:
            printf("Unknown syscall no: %d\n", syscall_num);
            cxt->a0 = -1;
    }
    return;
}
```

注意，不能在 trap 中调用 syscall。运行效果：

```bash
Sync exceptions! Code = 8 <------- 系统调用 print
System call from U-mode!
Task 0: Created!
Sync exceptions! Code = 8 <------- 系统调用 gethid
System call from U-mode!
--> sys_gethid, arg0 = 0x800057f8
system call returned!, hart id is 0
Sync exceptions! Code = 8 <------- 系统调用 print
System call from U-mode!
Task 0: Running...
```

## 系统调用的封装

OS 提供者还得提供一个C库（对 OS 来说，这就是标准库），这个 C 库 和 OS 的系统调用对应。

当软件开发者开发 OS 上层应用的时候，就可以调用这些头文件中的系统调用或者基于系统调用的函数了，也就是说，要对系统调用做一些基本的包装或者功能提升。








