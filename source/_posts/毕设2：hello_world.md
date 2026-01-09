---
title: 毕设(2)：Hello World!
date: 2026-01-10 10:03:13
tags: 毕设
categories: loongarch32r
mermaid: true
---

## 基础工作

### type.h
利用编译器内置的宏，可以精确的生成有无符号的 8、16、32等数据类型，这样就不用包含别人的库了：

```C
#ifndef __TYPES_H__
#define __TYPES_H__

typedef __INT8_TYPE__ int8_t;
typedef __INT16_TYPE__ int16_t;
typedef __INT32_TYPE__ int32_t;
typedef __INT64_TYPE__ int64_t;

typedef __UINT8_TYPE__ uint8_t;
typedef __UINT16_TYPE__ uint16_t;
typedef __UINT32_TYPE__ uint32_t;
typedef __UINT64_TYPE__ uint64_t;

typedef __SIZE_TYPE__ size_t;
typedef __UINTPTR_TYPE__ uintptr_t;
typedef __INTPTR_TYPE__ intptr_t;

typedef int8_t boolean;
#define true 1
#define false 0

#define NULL ((void *)0)

#endif
```

### uart.h
由于打印字符在指令层面上就是给某一个地址写数据。如果要是用 uart 来实现字符的输出，那么其实就是给 uart 控制器的接受缓冲地址写数据。

当时 uart 在使用之前，我们需要初始化它，这样才能使用它。于是我们对 uart 的一系列地址进行读写，这就是所谓的初始化。

### FCR - FIFO 控制寄存器 (偏移 0x02，只写)

```
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ 7 │ 6 │ 5 │ 4 │ 3 │ 2 │ 1 │ 0 │
  ├───┴───┼───┴───┼───┼───┼───┼───┤
  │ RCVR  │ saved │DMA│TXSR│RXSR│FIFO│
  │Trigger│       │Mode│   │   │ EN │
  └───────┴───────┴───┴───┴───┴───┘

  ┌─────┬──────────────┬────────────────────────────────────────────┐
  │ bit │     name     │                    info                    │
  ├─────┼──────────────┼────────────────────────────────────────────┤
  │ 0   │ FIFO_EN      │ 1=enable FIFO                              │
  ├─────┼──────────────┼────────────────────────────────────────────┤
  │ 1   │ RXSR         │ 1=reset recieve FIFO                       │
  ├─────┼──────────────┼────────────────────────────────────────────┤
  │ 2   │ TXSR         │ 1=reset trans FIFO                         │
  ├─────┼──────────────┼────────────────────────────────────────────┤
  │ 7:6 │ RCVR Trigger │ trigger level (00=1byte, 01=4, 10=8, 11=14)│
  └─────┴──────────────┴────────────────────────────────────────────┘
```

启动的时候，需要使能 FIFO，以及复位发送 FIFO、接收 FIFO。

### LCR - 线控制寄存器 (偏移 0x03)

```
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ 7 │ 6 │ 5 │ 4 │ 3 │ 2 │ 1 │ 0 │
  ├───┼───┼───┼───┼───┼───┴───┴───┤
  │DLAB│Break│Stick│EPS│PEN│  WLS  │
  └───┴───┴───┴───┴───┴───────────┘
  ┌─────┬──────┬───────────────────────────────────────┐
  │ 位  │ 名称 │                 说明                  │
  ├─────┼──────┼───────────────────────────────────────┤
  │ 1:0 │ WLS  │ 字长 (00=5位, 01=6位, 10=7位, 11=8位) │
  ├─────┼──────┼───────────────────────────────────────┤
  │ 2   │ STB  │ 停止位 (0=1位, 1=2位)                 │
  ├─────┼──────┼───────────────────────────────────────┤
  │ 3   │ PEN  │ 校验使能                              │
  ├─────┼──────┼───────────────────────────────────────┤
  │ 7   │ DLAB │ 除数锁存访问位 (1=访问波特率寄存器)   │
  └─────┴──────┴───────────────────────────────────────┘
```

设置波特率的时候，需要将 LCR 的 DLAB 位拉高，拉高后才能写入 divisor。

难点机制: 16550 UART 存在寄存器地址复用。
- DLL (低8位除数) 和 RBR/THR (数据寄存器) 共用同一个地址 0x00。
- DLM (高8位除数) 和 IER (中断使能) 共用同一个地址 0x01。

DLAB (Divisor Latch Access Bit): UART_LCR 的第 7 位。

- 当 DLAB = 1 时：访问 0x00 和 0x01 是在读写除数；
- 当 DLAB = 0 时：访问 0x00 和 0x01 是在读写数据/中断。

操作流程:
- 先写 UART_LCR = 0x80，把开关拨到“分频”档；
- 写 DLM (高位)；
- 写 DLL (低位)。

设置完 divisor 后，就可以把 UART_LCR 的 DLAB 关了，并且可以设置数据传送模式，比如 8N1。

### 关闭 Modem 控制以及中断控制

```C
#define UART_MCR 0x04 /* Modem控制 */
#define UART_IER 0x01 /* 中断使能（默认关闭） */
```

### 正式初始化
这是一个非常规范的 UART 初始化流程：
```C
#define UART_REG(offset) (*(volatile uint8_t *)(UART_BASE + (offset)))
int bsp_uart_init(uint8_t uart_id, uint32_t baudrate) {
    uint16_t divisor;

    /* 计算波特率除数: divisor = CLK / (16 * baudrate) */
    divisor = UART_CLK / (16 * baudrate);

    /* 1. 使能并复位FIFO */
    UART_REG(UART_FCR) = FCR_FIFO_EN | FCR_RXSR | FCR_TXSR; /* 0x07 */

    /* 2. 设置DLAB=1，准备写入除数 */
    UART_REG(UART_LCR) = LCR_DLAB; /* 0x80 */

    /* 3. 写入除数 */
    UART_REG(UART_DLM) = (divisor >> 8) & 0xFF; /* 高字节 */
    UART_REG(UART_DLL) = divisor & 0xFF;        /* 低字节 */

    /* 4. 设置数据格式 8N1，同时清除DLAB */
    UART_REG(UART_LCR) = LCR_8N1; /* 0x03 */

    /* 5. 关闭Modem控制 */
    UART_REG(UART_MCR) = 0x00;

    return 0;
}
```

## 仿真系统打印字符

打印出信息，是用来 debug 的重要途径。因此得准备两套打印机制：
- 写到某一个缓冲区，这个缓冲区未来可供某些算法处理后发送到 framebuffer 显示到 HDMI；
- 直接写到 uart，通过 uart 串口屏直接显示信息。

由于现在在初始阶段，目前只能先在 uart 上输出，而且在仿真系统中，只能在通过数据截获来输出：

```C
void uart_read_handling() {
    // APB UART write handling - monitor writes to UART THR (0x1fe001e0)
    if (top->apb_uart_wvalid && !last_apb_uart_wvalid) {
        uint32_t uart_addr = top->apb_uart_awaddr;
        uint8_t uart_char = top->apb_uart_wdata & 0xFF;
        if (uart_addr == 0x1fe00000) {
            if ((uart_char >= 32 && uart_char <= 126) ||
            uart_char == '\n' ||
            uart_char == '\r' ||
            uart_char == '\t') {

            putchar(uart_char);
            fflush(stdout);
        }
    }
}

```

写一个测试函数：
```C
#include <myprintf.h>
#include <types.h>
#include <uart.h>

int main(void) {
    /* 初始化 UART */
    bsp_uart_init(0, BAUDRATE);
    /* 测试输出 */

    myprintf("Hello, LoongArch!\n");
    myprintf("Number: %d\n", 123456789);
    myprintf("Negative: %d\n", -987654321);
    myprintf("Hex: 0x%08x\n", 0xDEADBEEF);
    myprintf("Pointer: %p\n", (void *)0x1FE001E0);
    myprintf("String: %s\n", "LoongArch32");
    myprintf("Char: %c\n", 'X');
    myprintf("Done!\n");
    /* 死循环 */
    while (1)
        ;
    return 0;
}
```
`myprintf()`其实是包装了 `bsp_uart_puts()` 的一个模仿 `printf()` 的函数，其中：

```C
void bsp_uart_putc(uint8_t uart_id, char ch) {
    #ifndef SIMULATION
        /* 等待发送缓冲区空 (真实硬件需要等待) */
        while (!(UART_REG(UART_LSR) & LSR_THRE))
            ;
    #endif
    UART_REG(UART_THR) = ch;
}
char bsp_uart_getc(uint8_t uart_id) {
    /* 等待数据就绪 */
    while (!(UART_REG(UART_LSR) & LSR_DR))
        ;
    return UART_REG(UART_RBR);
}
void bsp_uart_puts(uint8_t uart_id, const char *str) {
    while (*str) {
        bsp_uart_putc(uart_id, *str++);
    }
}
```
测试一下：

```txt
Hello, LoongArch!
Number: 123456789
Negative: -987654321
Hex: 0xdeadbeef
Pointer: 0x1fe001e0
String: LoongArch32
Char: X
Done!

[SIM] Received signal 2, stopping...
```
嗯嗯，非常不错，经历了漫长的调 Bug，总算测试成功了！
