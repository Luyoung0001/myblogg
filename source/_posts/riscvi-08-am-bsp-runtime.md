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

LLVM 后端能发汇编，模拟器能执行指令，这还不够。我们还需要让 C 程序像一个真正的裸机程序那样启动、运行、输出、停机。

这就是 AM/BSP 最小运行时的作用。

![AM/BSP 裸机启动流程](/images/mermaid-svg/riscvi-08-am-bsp-runtime/am-boot-flow.svg)

这张图要表达三件事：

1. 裸机程序不是从 `main()` 自动开始的。
2. `_start` 负责设置栈、清 `.bss`，然后调用 `main()`。
3. `halt()` 用 `ebreak` 通知模拟器程序正常结束。

## `_start` 做什么

启动代码在：

```text
am/src/platform/riscvi32r-sim/start.S
```

核心流程是：

```asm
_start:
  la sp, _stack_top

  la t0, _bss_start
  la t1, _bss_end
```

先设置栈，再清零 `.bss`，然后：

```asm
call main
call halt
```

这就是裸机 C 程序能跑起来的前提。

## UART 和 halt

平台相关代码在：

```text
am/src/platform/riscvi32r-sim/trm.c
```

UART 输出：

```c
#define UART_TX_ADDR 0x10000000u

void putch(char ch) {
  *(volatile unsigned char *)UART_TX_ADDR = (unsigned char)ch;
}
```

停机：

```c
void halt(int code) {
  register int a0 asm("a0") = code;
  asm volatile("ebreak" : "+r"(a0) : : "memory");
  while (1) {
  }
}
```

这里 `a0` 保存返回值，`ebreak` 告诉模拟器停机。模拟器再检查 `final_a0` 是否符合预期。

## `.word` wrapper 是过渡方案

在真实 LLVM 发射路径完全稳定之前，AM 里保留了 `.word` wrapper。比如 `xvi.S` 可以直接用编码宏构造 RISC-VI 指令。

这种方式的好处是：即使 LLVM 后端还没完全接好，也可以先验证模拟器和 RTL。

但它不是最终目标。最终目标是让 C 代码通过 LLVM 后端自然发出 RISC-VI 指令，`.word` 只作为 fallback 或低层测试手段。

## 构建和运行

常用命令：

```bash
cd /home/luyoung/llvm-project/riscv-vi-research

make -C am APP=smoke test
make -C am APP=riscvi_ops test
make -C am APP=uart_hello test
```

整体 AM smoke：

```bash
make am-test
```

全量 AM 应用：

```bash
make am-test-all
```

## 成对 workload

项目里有很多 `compare_*` 应用：

```text
compare_array_max_rv32r
compare_array_max_riscvi
compare_bounds_sum_rv32r
compare_bounds_sum_riscvi
...
```

它们的意义是：两版程序计算同一个结果，一版只用 RV32R，一版显式使用 RISC-VI 指令。后续做 RTL 对比时，这种同源 workload 很关键。

## 常见问题

链接失败时，先检查 linker script 和 runtime object。

程序不停机时，检查 `halt()` 是否执行到 `ebreak`。

UART 预期失败时，检查 `putch()` 写的是不是 `0x10000000`，以及模拟器 JSON 里的 `uart_output`。

RISC-VI 动态条数不足时，先看测试程序是否真的触发了 wrapper 或 LLVM 发射路径。

## 小结

AM/BSP 是软件闭环的地基。它让 C 程序不再只是一个孤立函数，而是能作为裸机 workload 进入模拟器和 RTL。下一篇进入评测口径：什么时候可以说“RISC-VI 比 RV32R 有收益”，什么时候还不能说。
