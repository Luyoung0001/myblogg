---
title: "RISC-V32 OS 实验（八）：硬件定时器"
date: 2025-03-24 13:03:13
tags:
  - "RISC-V"
  - "timer"
  - "CLINT"
categories:
  - "Operating Systems"
---

## CLINT

RISC-V 规范规定，CLINT 的寄存器编址采用内存映射（memory map）方式。具体寄存器编址采用 base + offset 的格式，且 base 由各个特定 platform 自己定义。针对QEMU-virt，其 CLINT 的设计参考了 SFIVE，base 为 0x2000000。

## 相关寄存器

### mtime

系统全局唯一，在 RV32 和 RV64 上都是 64-bit。系统必须保证该计数器的值始终按照一个固定的频率递增。上电复位时，硬件负责将 mtime 的值恢复为 0。

### mtimecmp

每个 hart 一个 mtimecmp 寄存器，64-bit。上电复位时，系统不负责设置 mtimecmp 的初值。
当 mtime >= mtimecmp 时，CLINT 会产生一个 timer 中断。如果要使能该中断需要保证全局中断打开并且 mie.MTIE 标志位置 1。当 timer 中断发生时，hart 会设置 mip.MTIP，程序可以在 mtimecmp 中写入新的值清除 mip.MTIP。

因此，为了持续得获取时钟中断，我们在处理中断的时候，新设置一个 mtimecmp，这样就能不断得发生时钟中断。

## 硬件定时器的应用

利用时钟中断来做一个电子钟。

### 首先进行基本配置
```C

void timer_load(int interval) {
    int id = r_mhartid();
    *(uint64_t*)CLINT_MTIMECMP(id) = *(uint64_t*)CLINT_MTIME + interval;
}


void timer_init() {
    // 设置中断间隔
    timer_load(TIMER_INTERVAL);

    // 开启机器模式的时钟中断
    w_mie(r_mie() | MIE_MTIE);

    // 开启机器模式全局中断
    w_mstatus(r_mstatus() | MSTATUS_MIE);
}
```

### 处理时间中断

这样，当 mtime 的值大于 mtimecmp 的时候，就会发起中断。然后进入中断处理函数：

```C
reg_t trap_handler(reg_t epc, reg_t cause) {
    reg_t return_pc = epc;
    reg_t cause_code = cause & MCAUSE_MASK_ECODE;

    if (cause & MCAUSE_MASK_INTERRUPT) {
        /* Asynchronous trap - interrupt */
        switch (cause_code) {
            case 3:
                uart_puts("software interruption!\n");
                break;
            case 7:
                // uart_puts("timer interruption!\n");
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

为了持续得发生时钟中断，我们可以这样修改 handler：

```C
void print_time(uint32_t tick) {
    uint32_t sec = tick % 60;
    uint32_t minute = tick / 60;
    uint32_t hour = tick / 3600;

	printf("%02d:%02d:%02d\n", hour, minute, sec);

}

void timer_handler() {
    _tick++;
    // printf("tick: %d\n", _tick);
    print_time(_tick);

    timer_load(TIMER_INTERVAL);
}
```

打印当前的时间后，将会重新设置 mtimecmp 后，将会再次引发时钟中断。因此执行结果大致如下：

```bash
Hello, RVOS!
HEAP_START = 0x800070f0(aligned to 0x80008000), HEAP_SIZE = 0x07ff8f10,
num of reserved pages = 8, num of pages to be allocated for heap = 32752
TEXT:   0x80000000 -> 0x80003938
RODATA: 0x80003938 -> 0x80003c44
DATA:   0x80004000 -> 0x80004004
BSS:    0x80004010 -> 0x800070f0
HEAP:   0x80010000 -> 0x88000000
Task 0: Created!
Task 1: Created!
0:0:1
0:0:2
0:0:3
```

这就是一个简单的时间显示了，为了让它现实得很精准，就需要了解 CPU 的时钟频率，通过计算来定义 TIMER_INTERVAL 的大小。