---
title: "Cortex-M3 实验：用 QEMU 运行汇编程序"
date: 2024-04-27 13:14:57
tags:
  - "Cortex-M3"
  - "assembly"
  - "QEMU"
categories:
  - "Embedded Systems"
---

<!--more-->

## 正文
环境：macOS M1。
[前文](https://blog.csdn.net/m0_73651896/article/details/138233484?spm=1001.2014.3001.5501)讨论了 qemu 模拟8086 的平台运行8086 汇编代码，本文将讨论 qemu 模拟arm 平台运行 CM3 代码。代码：

```c
.syntax unified
.cpu cortex-m3

.global _start

.equ UART0_BASE, 0x4000C000
.equ UART0_DR, UART0_BASE + 0x00
.equ UART0_FR, UART0_BASE + 0x18
.equ TXFE, 0x80

.section .text
.thumb_func
_start:
    // 设置堆栈指针
    ldr sp, =stack_top

    // 复位向量表
    .word _start

    // 复位向量（Reset Handler）
    _reset_handler:
        // 初始化 UART0
        ldr r0, =UART0_DR
        ldr r1, =0x48 // 'H'
        str r1, [r0]
        ldr r1, =0x65 // 'e'
        str r1, [r0]
         ldr r1, =0x6C // 'l'
        str r1, [r0]
        ldr r1, =0x6C // 'l'
        str r1, [r0]
        ldr r1, =0x6F // 'o'
        str r1, [r0]
        ldr r1, =0x2C // ','
        str r1, [r0]
        ldr r1, =0x20 // ' '
        str r1, [r0]
        ldr r1, =0x57 // 'W'
        str r1, [r0]
        ldr r1, =0x6F // 'o'
        str r1, [r0]
        ldr r1, =0x72 // 'r'
        str r1, [r0]
        ldr r1, =0x6C // 'l'
        str r1, [r0]
        ldr r1, =0x64 // 'd'
        str r1, [r0]
        ldr r1, =0x21 // '!'
        str r1, [r0]
        // 其他字符...

    // 等待 UART0 发送完成
    wait_for_tx:
        ldr r0, =UART0_FR
        ldrb r1, [r0]
        tst r1, #TXFE
        bne wait_for_tx

    // 程序结束
    b .

stack_top: .word 0x20001000
```
编译以及链接：

```bash
arm-none-eabi-as -o hello.o hello.s
arm-none-eabi-ld -Ttext=0x0 -o hello.elf hello.o
arm-none-eabi-objcopy -O binary hello.elf hello.bin
```

如果mac 上没有这些命令，直接安装以下：
### 安装：

```bash
brew tap ArmMbed/homebrew-formulae
brew install arm-none-eabi-gcc
```
这两个命令是用于在 macOS 系统上安装 ARM 嵌入式开发工具链的：

1. `brew tap ArmMbed/homebrew-formulae`：
   - 这个命令通过 Homebrew 向您的系统添加了一个额外的存储库（tap），这个存储库包含了 ARM Mbed 的一些软件包。Homebrew 是 macOS 上一个流行的包管理器，用于安装和管理软件包。
   - `tap` 命令允许您在 Homebrew 中添加非官方的软件源，以便于安装那些不在官方存储库中的软件包。

2. `brew install arm-none-eabi-gcc`：
   - 这个命令使用 Homebrew 从 ARM Mbed 的存储库中安装 ARM 嵌入式设备的 GCC 工具链。
   - `arm-none-eabi-gcc` 是一个用于编译嵌入式 C/C++ 代码的工具链，针对 ARM Cortex-M 和 Cortex-R 处理器系列。这个工具链包括了编译器、链接器等工具，用于将源代码编译为可在 ARM 嵌入式设备上运行的机器代码。


### 验证安装：
   安装完成后，可以在终端中运行以下命令来验证 ARM Cortex-M 工具链的安装：
   ```
   arm-none-eabi-gcc --version
   arm-none-eabi-as --version
   arm-none-eabi-ld --version
   arm-none-eabi-objcopy --version
   ```


我们接着就用 qemu 来运行我们的二进制文件：

```bash
qemu-system-arm -M lm3s6965evb -nographic -kernel hello.bin
Timer with period zero, disabling
Hello, World!
```

可以看到，成功得运行了。其中的 **lm3s6965evb** 是 Luminary Micro 的 Cortex-M3 处理器（LM3S6965）的一种开发板型号，这个命令告诉 QEMU 使用 LM3S6965 Cortex-M3 处理器模拟器，在终端中以无图形界面模式运行，并加载名为 **hello.bin** 的内核镜像文件。





