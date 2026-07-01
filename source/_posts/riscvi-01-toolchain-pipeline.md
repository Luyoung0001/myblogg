---
title: "RISC-VI 研究笔记（一）：从 C 程序到 RISC-VI 模拟器"
date: 2026-04-13 15:03:13
tags:
  - "RISC-VI"
  - "LLVM"
  - "RISC-V"
  - "simulator"
categories:
  - "Computer Architecture"
---

## 前言

做 ISA 实验时，最容易把注意力放在“我加了几条指令”上。但从工程角度看，一条新指令真正成立，不是因为它能被手写汇编跑一次，而是因为它能穿过完整的软件栈：

```text
C 源码
  -> LLVM IR
  -> 目标相关指令选择
  -> 汇编文本
  -> object
  -> 裸机 ELF
  -> flat binary
  -> 模拟器执行结果
```

RISC-VI 这个项目的第一条主线，就是把这条链路做成可以反复验证的闭环。读者不需要有本地仓库也能理解这件事：它本质上是在回答“一个 LLVM 后端改动，如何一路变成可执行的机器行为”。

![C 到 RISC-VI 模拟器的端到端流水线](/images/mermaid-svg/riscvi-01-toolchain-pipeline/toolchain-flow.svg)

这张图里最重要的不是某个命令，而是职责分层：

1. `clang` 把 C 语言降到 LLVM IR，它还没有真正决定要用哪条 RISC-VI 指令。
2. `llc` 进入 RISC-V 后端，根据 subtarget feature 和 pattern 做目标相关降级。
3. `llvm-mc` 负责把汇编文本编码成 object，检验 MC 层是否认识新指令。
4. AM/BSP runtime 提供裸机启动、UART 和停机约定。
5. C 模拟器把 flat binary 当作机器码执行，并给出结构化结果。

## 这不是“跑通 Hello”那么简单

很多工程报告会从一个 Hello 程序开始，然后展示一串终端输出。问题在于，读者如果没有项目、没有构建环境、没有本地终端，这种写法只会让人一头雾水。

更合适的讲法是：Hello 程序在这里不是为了展示 RISC-VI 指令收益，而是为了验证最小裸机闭环。

一个最小 Hello 能检查四件事：

1. 启动代码能否把栈和 `.bss` 初始化好。
2. C 程序能否调用到平台层的 `putch()`。
3. UART MMIO 写入能否被模拟器捕获。
4. `halt()` 能否用 `ebreak` 把程序返回值交给模拟器。

所以 Hello 的价值不是“输出了一行字符串”，而是证明编译、链接、加载、执行、停机这几层已经对齐。只有这个地基稳定，后面讨论 `lwxs`、`csel`、`bchkltu` 才有意义。

## 从 Makefile 看工具链分工

本项目的 `cpu-test/Makefile` 是理解端到端链路的好入口。它不是博客读者必须执行的脚本，而是一份工程分层说明。

在源码里，这条链路的总说明写在 `riscv-vi-research/cpu-test/Makefile:3`，工具路径和目标模式从 `riscv-vi-research/cpu-test/Makefile:17` 开始定义，真正的 C 到 IR、IR 到汇编、ASM 到 OBJ、ELF 到 BIN、BIN 到 JSON 规则分别出现在 `riscv-vi-research/cpu-test/Makefile:137`、`:147`、`:155`、`:163` 和 `:179` 附近。

第一步是 C 到 LLVM IR。规则里使用 `clang -S -emit-llvm` 生成 `.ll`：

```makefile
$(CLANG) $(CLANG_FLAGS) -S -emit-llvm "$<" -o "$@"
```

这里的 flags 仍然是普通 RV32I 裸机口径：

```makefile
-target riscv32 -march=rv32i -mabi=ilp32 -O2 -ffreestanding
```

这说明前端阶段并没有把世界切换成一个全新的架构。它先把 C 程序变成可被 LLVM 后端消费的中间表示。

第二步是 LLVM IR 到汇编。RISC-VI 的关键开关出现在 `llc` 阶段：

```makefile
LLC_FLAGS_riscvi32r := -mtriple=riscv32 -mattr=+xvi32r
```

注意这里仍然是 `-mtriple=riscv32`。RISC-VI 在当前阶段不是新建一个完整 target，而是作为 RISC-V 后端里的 vendor extension 被打开。这个设计选择会在下一篇展开。

第三步是 MC 编码。`llvm-mc` 同样需要 `+xvi32r`：

```makefile
ASM_FLAGS_riscvi32r := -triple=riscv32 -mattr=+xvi32r
```

这一步很关键。LLVM 后端能打印 `min` 只说明 asm printer 知道这个名字；MC 层能把它编码成 object，才说明汇编器、编码描述和 feature predicate 也同步了。

第四步是链接 AM runtime，生成裸机 ELF，再通过 `llvm-objcopy` 得到 flat binary。最后模拟器读取 binary，按 ISA 语义逐条提交。

## JSON 是测试接口，不是读者观看点

项目里模拟器输出 JSON，例如 `halted`、`halt_reason`、`final_a0`、`uart_output`、动态指令统计等字段。它们不是为了让读者手动盯终端，而是为了让测试系统有一个稳定接口。

这类字段背后的工程含义是：

- `halted` 和 `halt_reason` 描述程序是否按约定停机。
- `final_a0` 对应裸机程序返回值。
- `uart_output` 捕获 MMIO 输出，避免把 UART 行为混进非结构化终端文本。
- 指令计数字段用于后续比较 RV32R 和 RISC-VI32R 的动态行为。

换句话说，JSON 的意义是把“程序似乎跑了”变成“程序以可检查的方式结束了”。

## 为什么要先建立端到端闭环

如果只有 TableGen 里的一条 `def`，我们只能说 LLVM 认识了某条机器指令。如果只有模拟器里的一个 `case`，我们只能说 reference model 能执行某个编码。如果只有手写 `.word`，我们甚至无法证明编译器真的会生成它。

RISC-VI 的完整闭环把这些层连在一起：

```text
LLVM 后端能选择指令
  -> MC 层能编码指令
  -> 裸机 runtime 能承载程序
  -> 模拟器能执行语义
  -> 报告系统能记录结果
```

这就是后面所有性能讨论的地基。没有这条链路，`min/max` 是否减少分支、`lwxs/swxs` 是否压缩地址生成链、`bchkltu` 是否改善边界检查，都只能停留在猜想。

## 项目里的验证入口

项目里确实有用于复现这条链路的 `cpu-test` 入口，例如生成汇编、运行单个用例、汇总 RV32R/RISC-VI32R 对比报告。但这些入口只是工程验证手段，不是理解本文的前提。

更重要的是看入口背后的分层：C 到 IR、IR 到汇编、MC 编码、链接 runtime、模拟器运行、报告汇总。理解了这条链路，才算真正理解一个 LLVM 后端实验是如何落地的。

## 小结

这一篇建立的是工程地基：RISC-VI 不是孤立的汇编实验，而是一条从 C 程序到模拟器结果的完整工具链。下一篇进入 LLVM 后端内部，看为什么项目第一阶段选择 `+xvi32r` 这种 extension 路线，而不是一开始就新建 `riscvi32r` target。
