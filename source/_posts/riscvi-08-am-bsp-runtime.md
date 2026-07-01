---
title: "RISC-VI 研究笔记（八）：AM/BSP 最小运行时与裸机程序"
date: 2026-04-20 15:03:13
tags:
  - "RISC-VI"
  - "bare metal"
  - "AM"
  - "RISC-V"
categories:
  - "Computer Architecture"
---

## 前言

LLVM 后端能生成汇编，模拟器能执行指令，这还不足以支撑系统实验。因为真实的 C 程序不是从 `main()` 自己冒出来的，它需要启动代码、栈、`.bss`、链接脚本、平台 I/O 和停机约定。

AM/BSP 最小运行时解决的就是这个问题：让 C 程序以裸机 workload 的形式进入模拟器和 RTL。

![AM/BSP 裸机启动流程](/images/mermaid-svg/riscvi-08-am-bsp-runtime/am-boot-flow.svg)

## 裸机程序为什么不能直接从 `main()` 开始

在普通操作系统里，用户程序启动前已经有 loader、libc runtime 和内核环境。但在 RISC-VI 的模拟器实验里，flat binary 被直接放进内存执行。此时必须有人负责：

1. 设置栈指针。
2. 清零 `.bss`。
3. 调用 `main()`。
4. 把 `main()` 返回值交给停机逻辑。
5. 防止程序返回到未知地址。

这些工作由平台启动代码完成。

## `_start`：最小启动序列

启动代码位于：

```text
riscv-vi-research/am/src/platform/riscvi32r-sim/start.S
```

这段启动代码从 `riscv-vi-research/am/src/platform/riscvi32r-sim/start.S:5` 开始：先设置 `sp`，再清 `.bss`，最后调用 `main` 和 `halt`。

核心逻辑很短：

```asm
_start:
  la sp, _stack_top

  la t0, _bss_start
  la t1, _bss_end
1:
  bgeu t0, t1, 2f
  sw zero, 0(t0)
  addi t0, t0, 4
  jal zero, 1b

2:
  call main
  call halt
```

这段代码解释了裸机 runtime 的本质：它不是一个完整操作系统，而是为 C 程序建立最少的执行前提。

## UART：用 MMIO 表达输出

平台相关代码在：

```text
riscv-vi-research/am/src/platform/riscvi32r-sim/trm.c
```

UART 和停机约定位于 `riscv-vi-research/am/src/platform/riscvi32r-sim/trm.c:6` 和 `:14`。前者固定 MMIO 地址，后者把返回码绑定到 `a0` 并执行 `ebreak`。

UART 输出被建模成一个固定地址的 MMIO 写：

```c
#define UART_TX_ADDR 0x10000000u

void putch(char ch) {
  *(volatile unsigned char *)UART_TX_ADDR = (unsigned char)ch;
}
```

这里的 `volatile` 很关键。它告诉编译器这次内存写有外部可见副作用，不能随意删除或重排成普通无用 store。模拟器则在内存系统里捕获这个地址，把字符累积成 UART 输出字段。

## halt：用 ABI 寄存器传返回值

停机逻辑同样在 `trm.c`：

```c
void halt(int code) {
  register int a0 asm("a0") = code;
  asm volatile("ebreak" : "+r"(a0) : : "memory");
  while (1) {
  }
}
```

这里有两个约定：

1. `a0` 保存返回码，符合 RISC-V 常见 ABI 返回值习惯。
2. `ebreak` 告诉模拟器程序主动停机。

模拟器看到 `ebreak` 后，可以读取 `a0` 并形成 `final_a0`。这就是裸机测试里“函数返回值”如何跨过 runtime 边界，被外部测试系统观察到。

## `.word` wrapper：工程过渡层

在编译器发射完全稳定之前，AM 平台层可以通过 `.word` wrapper 直接构造 RISC-VI 指令。这类 wrapper 的价值不在于替代 LLVM，而在于分层验证。

它可以帮助区分问题：

- wrapper 正确、LLVM 不发，问题多半在后端 pattern 或 feature。
- wrapper 和 LLVM 都能发、模拟器不执行，问题可能在 decode/execute。
- 模拟器通过、RTL 不通过，问题进入硬件实现或 hazard 处理。

所以 wrapper 是早期 bring-up 的脚手架，而不是最终交付形态。最终目标仍然是让普通 C 代码经 LLVM 后端自然生成 RISC-VI 指令。

## 成对 workload 的意义

项目里有不少 `compare_*` 应用，例如：

```text
compare_array_max_rv32r
compare_array_max_riscvi
compare_bounds_sum_rv32r
compare_bounds_sum_riscvi
```

它们的设计思路是：两版程序计算同一个结果，但一版使用普通 RV32R 序列，另一版显式使用或触发 RISC-VI 指令。

这对后续评测很重要。ISA 实验最怕 workload 不同导致归因混乱。同源程序对能把变量压到“指令序列不同”，从而更公平地比较 retired count、cycle count、stall 和分支行为。

## 项目里的验证入口

AM 层的检查入口覆盖单个应用 smoke 和全量 AM 应用。它们背后的检查对象是：启动代码、链接脚本、UART、`halt()`、wrapper、应用返回值和 RISC-VI 指令计数。它们是裸机 workload 能进入模拟器和 RTL 的前置条件。

## 小结

AM/BSP 是软件闭环的地基。它把 C 函数变成真正的裸机程序，让编译器输出不再只是汇编片段，而是可以被模拟器和 RTL 加载执行的 workload。下一篇进入评测口径：什么时候可以说 RISC-VI 有收益，什么时候还不能说。
