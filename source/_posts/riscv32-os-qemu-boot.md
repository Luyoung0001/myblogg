---
title: "RISC-V32 OS 实验（一）：用 QEMU 启动裸机程序"
date: 2025-03-15 13:03:13
tags:
  - "RISC-V"
  - "QEMU"
  - "bare metal"
categories:
  - "Operating Systems"
---

## 写的 OS 如何运行？

如果写过 CPU，那就会明白，OS 和其它裸机程序几乎没有什么区别。当然，复杂的 OS 可能引入各种模式，比如 S-mode，但这不影响 OS 是一个比较复杂的裸机程序的事实。

因此，写好的 OS 加载到内存中，让 CPU 的 PC 指向 OS 的第一条指令，然后开始运行，OS 就成功运行了。

根据以上判断，OS 想要跑起来，需要：
- 一个 CPU；
- 一个 RAM。

有一个模拟器 qemu，它提供了上述条件的超集。

## 如何利用 qemu 运行 一个裸机程序？

要问 qemu 如何运行一个 OS，倒不如问 qemu 如何运行一个裸机程序，毕竟前面说了，OS 和裸机程序并没有多大区别。

先看一个裸机程序：

```asm
	.text			# Define beginning of text section
	.global	_start		# Define entry _start

_start:
	li x6, 1		# x6 = 1
	li x7, 2		# x7 = 2
	add x5, x6, x7		# x5 = x6 + x7

stop:
	j stop			# Infinite loop to stop execution

	.end			# End of file

```

上面这个程序足够简单，它的作用是计算 1+2 的值，并且将值放在 x5 中。

### 编译
程序编译成指令流才能运行在 CPU 上，上面的程序也要编译。但是假设现在的环境是 linux，那么需要进行交叉编译：在 linux 上编译目标为 riscv 平台的二进制文件。

```bash
EXEC = test
SRC = ${EXEC}.s

CROSS_COMPILE = riscv64-unknown-elf-
CFLAGS = -nostdlib -fno-builtin -march=rv32g -mabi=ilp32 -g -Wall

QEMU = qemu-system-riscv32
QFLAGS = -nographic -smp 1 -machine virt -bios none

GDB = gdb-multiarch
CC = ${CROSS_COMPILE}gcc
OBJCOPY = ${CROSS_COMPILE}objcopy
OBJDUMP = ${CROSS_COMPILE}objdump

all:
	@${CC} ${CFLAGS} ${SRC} -Ttext=0x80000000 -o ${EXEC}.elf
	@${OBJCOPY} -O binary ${EXEC}.elf ${EXEC}.bin
```

上面的 `-Ttext=0x80000000`取代了链接脚本的作用，CFLAGS 的含义也很简单，后面规定了输出格式是 32 位的通用指令集。

## 运行

```bash
run: all
	@echo "Press Ctrl-A and then X to exit QEMU"
	@echo "------------------------------------"
	@echo "No output, please run 'make debug' to see details"
	@${QEMU} ${QFLAGS} -kernel ./${EXEC}.elf
```

展开之后，命令如下：
```bash
qemu-system-riscv32 -nographic -smp 1 -machine virt -bios none -kernel ./test.elf
```

这个命令含义是使用 QEMU 模拟器运行一个针对 RISC-V 32位架构的程序：

* `qemu-system-riscv32`:
    * 这是 QEMU 模拟器的命令，用于模拟 RISC-V 32位架构的系统。
* `-nographic`:
    * 这个选项告诉 QEMU 禁用图形界面，使用文本控制台进行输入和输出。这意味着程序将在终端中运行，而不是在一个独立的图形窗口中。
* `-smp 1`:
    * 这个选项设置模拟器的 CPU 数量为 1，即单核 CPU。
* `-machine virt`:
    * 这个选项指定模拟的机器类型为 `virt`，这是一个通用的 RISC-V 虚拟机，适合运行裸机程序。
* `-bios none`:
    * 这个选项告诉 QEMU 禁用 BIOS。因为我们运行的是裸机程序，不需要 BIOS。
* `-kernel ./test.elf`:
    * 这个选项告诉 QEMU 加载并运行 `test.elf` 文件作为内核。这意味着 `test.elf` 文件被当作操作系统内核加载运行。

**综合来看，这个命令的完整含义是：**

使用 QEMU 模拟一个单核 RISC-V 32位虚拟机，禁用图形界面和 BIOS，并加载 `test.elf` 文件作为内核运行。

**具体来说，这个命令用于：**

* 在没有实际硬件的情况下，模拟 RISC-V 32位系统的运行环境。
* 运行编译好的裸机程序（`test.elf`），并查看其输出。
* 调试裸机程序，因为 QEMU 可以与 GDB 配合使用。

QEMU能够正确加载程序到内存地址，是因为它能够读取并解释ELF文件中的加载地址信息。


这样，我们就知道 qemu 怎么用了，那么运行一个 OS 就很简单了。
















