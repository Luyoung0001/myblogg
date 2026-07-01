---
title: "RISC-VI 研究笔记（四）：LLVM 指令选择如何生成 lwxs、csel 和 bchkltu"
date: 2026-04-16 15:03:13
tags:
  - "RISC-VI"
  - "LLVM"
  - "instruction selection"
  - "RISC-V"
categories:
  - "Computer Architecture"
---

## 前言

定义机器指令只是第一步。更关键的问题是：LLVM 什么时候会选择它？

这篇从指令选择的角度看 RISC-VI。也就是说，我们关心的是：

```text
LLVM IR 中的某种语义模式
  -> SelectionDAG 或 RISC-V DAG 节点
  -> TableGen pattern 匹配
  -> RISC-VI MachineInstr
  -> 汇编输出
```

![LLVM 指令选择流程](/images/mermaid-svg/riscvi-04-instruction-selection/selection-flow.svg)

这张图要表达三件事：

1. 指令选择基于语义模式，不是文本替换。
2. 不同指令的来源不同：min/max、indexed memory、select、branch 各有路径。
3. 调试时要先看 IR，再看汇编，最后跑模拟器。

## min/max：最自然的一类

`min/max/minu/maxu` 最适合作为指令选择入门，因为 LLVM 中已经有稳定节点：

```tablegen
def : PatGprGpr<smin, XVI_MIN, i32>;
def : PatGprGpr<smax, XVI_MAX, i32>;
def : PatGprGpr<umin, XVI_MINU, i32>;
def : PatGprGpr<umax, XVI_MAXU, i32>;
```

因此如果 C 程序形成了类似 min/max 的 IR，打开 `+xvi32r` 后就有机会发出 `min/max`。

可以直接看：

```bash
cd /home/luyoung/llvm-project/riscv-vi-research

make -C cpu-test asm CASE=max MODE=riscvi32r
make -C cpu-test asm CASE=min3 MODE=riscvi32r
```

对比 RV32R：

```bash
make -C cpu-test asm CASE=max MODE=rv32r
make -C cpu-test asm CASE=max MODE=riscvi32r
```

## indexed memory：GEP + load/store

`lwxs/swxs` 对应的是数组索引访问：

```llvm
%p = getelementptr i32, ptr %base, i32 %idx
%v = load i32, ptr %p
```

在 RV32I 中，这类访问常见形式是：

```asm
slli t0, idx, 2
add  t0, base, t0
lw   rd, 0(t0)
```

RISC-VI 想把地址生成链折叠成：

```asm
lwxs rd, base, idx, 2
```

TableGen 里使用 indexed address pattern：

```tablegen
def XVIAddrRegRegScale3 : AddrRegRegScale<3>;

def : XVI_LdIdxPat<load, XVI_LWXS>, Requires<[HasVendorXRiscVI, IsRV32]>;
def : XVI_StIdxPat<store, XVI_SWXS>, Requires<[HasVendorXRiscVI, IsRV32]>;
```

这里一定要记住：`lwxs/swxs` 减少的是地址生成链，不是减少真实 load/store 次数。

## csel：减少小分支，但会引入三源

`csel` 的目标是条件选择：

```text
csel rd, true_value, false_value, cond
```

对应 pattern 类似：

```tablegen
def : Pat<(riscv_selectcc (i32 GPR:$cond), 0, SETNE,
                          (i32 GPR:$truev), GPR:$falsev),
          (XVI_CSEL GPR:$truev, GPR:$falsev, GPR:$cond)>;
```

它的好处是可以减少一些小控制流片段。代价也很明确：三源寄存器读取会带来读端口压力。所以后面做 RTL 评测时，不能只看提交数减少，还要看 `three_source_count` 和 `read_port_pressure_count`。

## bchkltu：失败路径分支

`bchkltu` 是边界检查失败分支：

```text
bchkltu index, length, fail
```

语义是：

```c
if ((uint32_t)index >= (uint32_t)length) {
  goto fail;
}
```

它对应的是 `idx < len` 检查的反方向。TableGen pattern 里是：

```tablegen
let AddedComplexity = 10 in
def : Pat<(riscv_brcc (i32 GPR:$idx), GPR:$len, SETUGE, bb:$target),
          (XVI_BCHKLTU GPR:$idx, GPR:$len, bare_simm13_lsb0_bb:$target)>;
```

这里最容易出错的是方向。源码里经常写 `idx < len` 表示通过，而 `bchkltu` 跳的是失败路径。

## 验证命令

看汇编：

```bash
cd /home/luyoung/llvm-project/riscv-vi-research

make -C cpu-test asm CASE=bounds-check MODE=riscvi32r
make -C cpu-test asm CASE=select-sort MODE=riscvi32r
```

运行：

```bash
make -C cpu-test run CASE=max MODE=riscvi32r
make -C cpu-test run CASE=bounds-check MODE=riscvi32r
```

如果怀疑 pattern 没匹配，先看 `.ll`，再看 `.s`。不要一开始就怀疑模拟器。

## 小结

LLVM 指令选择的核心不是“我希望它发什么”，而是“IR 中是否稳定出现这种语义模式”。RISC-VI 的 8 条指令之所以有意义，正是因为它们分别对上了 LLVM 输出中比较常见的模式。下一篇我们从工程角度看：新增一条指令到底要同步改多少层。
