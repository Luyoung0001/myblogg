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

做一个 ISA 实验，最容易犯的错误是先盯着指令本身看：我加了什么指令，编码长什么样，汇编怎么写。可真正难的地方其实不是“写出几条新指令”，而是让这些指令进入一条完整的工程流水线。

对 RISC-VI 这个项目来说，我最想先确认的是：

- C 程序能不能先变成 LLVM IR；
- 修改后的 LLVM 后端能不能发出 RISC-VI 汇编；
- `llvm-mc` 能不能识别这些汇编并生成 object；
- AM/BSP runtime 能不能把程序链接成裸机 ELF；
- flat binary 能不能被 C 模拟器执行；
- 最后能不能用 JSON 和 trace 自动验证结果。

这篇先不急着讲 `lwxs`、`csel`、`bchkltu` 这些指令的细节，而是先把整个链路跑通。

![C 到 RISC-VI 模拟器的端到端流水线](/images/mermaid-svg/riscvi-01-toolchain-pipeline/toolchain-flow.svg)

这张图要表达三件事：

1. `clang` 在这里主要负责把 C 变成 LLVM IR。
2. 真正把 LLVM IR 降到 RISC-VI 汇编的是修改后的 `llc`。
3. 模拟器输出 JSON，不只是为了好看，而是为了后续自动化回归。

## 先区分几个工具

很多刚接触 LLVM 后端的人会把 `clang`、`llc`、`llvm-mc` 混在一起。它们在这条链路里分工很明确。

`clang` 是前端，负责读 C 代码，做语义分析和优化，然后生成 LLVM IR。当前项目里 `cpu-test/Makefile` 里的规则就是：

```makefile
$(BUILD_DIR)/riscvi32r/%.ll: tests/%.c
	$(CLANG) $(CLANG_FLAGS) -S -emit-llvm "$<" -o "$@"
```

`llc` 是后端，负责把 LLVM IR 降到目标机器汇编。RISC-VI 的关键开关就在这里：

```makefile
LLC_FLAGS_riscvi32r := -mtriple=riscv32 -mattr=+xvi32r
```

注意，这里仍然是 `-mtriple=riscv32`，不是 `riscvi32r`。当前阶段 RISC-VI 作为 RISC-V 后端的实验扩展接入，使用 `+xvi32r` 打开新指令。

`llvm-mc` 是汇编器，把汇编文本变成 object：

```makefile
$(LLVM_MC) $(ASM_FLAGS_riscvi32r) -filetype=obj "$<" -o "$@"
```

然后 `riscv32-unknown-elf-ld` 链接 AM runtime，`llvm-objcopy` 生成 flat binary，最后交给 `sim/riscvi32r-sim`。

## 一条最小运行命令

先跑一个最简单的 Hello：

```bash
cd /home/luyoung/llvm-project/riscv-vi-research

make -C cpu-test run CASE=hello MODE=riscvi32r
```

正常情况下，输出 JSON 中应该看到类似字段：

```json
{
  "halted": true,
  "halt_reason": "ebreak",
  "final_a0": 0,
  "uart_output": "Hello RISC-VI!"
}
```

这里有一个小细节：模拟器不会直接在终端打印 `Hello RISC-VI!`，它会把 UART 输出收集到 JSON 的 `uart_output` 字段。这样做是为了方便脚本校验。

## 查看中间产物

可以把生成物都摊开看：

```bash
ls build/cpu-test/riscvi32r/hello.*
sed -n '1,120p' build/cpu-test/riscvi32r/hello.s
cat build/cpu-test/riscvi32r/hello.json
```

如果只想看汇编：

```bash
make -C cpu-test asm CASE=hello MODE=riscvi32r
```

`hello` 通常不会出现 `lwxs`、`min`、`bchkltu` 这类自定义指令，因为它只是循环输出字符串。它验证的是基础闭环，不是验证 RISC-VI 指令收益。

要看真实触发 RISC-VI 指令，可以换几个 case：

```bash
make -C cpu-test asm CASE=max MODE=riscvi32r
make -C cpu-test asm CASE=min3 MODE=riscvi32r
make -C cpu-test asm CASE=bounds-check MODE=riscvi32r
```

也可以直接 grep：

```bash
grep -nE '\b(min|max|lwxs|swxs|bchkltu|csel)\b' build/cpu-test/riscvi32r/*.s
```

## 这条链路为什么重要

如果只有手写汇编，最多说明“模拟器能执行某条指令”。如果只有 LLVM 后端样例，最多说明“后端能打印某条指令”。真正有价值的是：

```text
C 程序 -> LLVM 后端 -> 汇编器 -> 裸机程序 -> 模拟器 -> 自动化结果
```

这意味着后续讨论 `min/max` 是否能减少分支、`lwxs/swxs` 是否能压缩地址生成链、`bchkltu` 是否能覆盖边界检查，都有了统一的实验入口。

## 常见问题

如果找不到 `llc`，先检查：

```bash
ls /home/luyoung/llvm-project/build-riscvi-llvm/bin/llc
```

如果汇编器不认识 RISC-VI 指令，要确认 `llvm-mc` 也是同一个修改后的 LLVM 构建产物。

如果 `hello` 里没有 RISC-VI 指令，不是失败。它只是没有形成 `min/max`、indexed memory 或 bounds check 这类模式。

## 小结

这一篇建立的是工程地基：RISC-VI 不是孤立的汇编实验，而是一条从 C 到模拟器的完整流水线。下一篇开始，我们进入 LLVM 后端本身，看为什么这个项目第一阶段选择 `+xvi32r`，而不是一开始就新建一个 `riscvi32r` target。
