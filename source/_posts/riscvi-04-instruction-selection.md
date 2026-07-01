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

定义机器指令只是第一步。LLVM 后端真正有趣的问题是：它什么时候会主动生成这些指令？

这不是字符串替换。LLVM 不会看到汇编里有 `slli; add; lw` 再粗暴改成 `lwxs`。更真实的路径是：

```text
LLVM IR
  -> DAG combine / legalize
  -> SelectionDAG 节点
  -> target-specific address selection
  -> TableGen pattern
  -> MachineInstr
  -> asm printer / MC encoder
```

![LLVM 指令选择流程](/images/mermaid-svg/riscvi-04-instruction-selection/selection-flow.svg)

这篇不以“跑某个 case”为主线，而是看 RISC-VI 的几类指令分别接在 LLVM 指令选择的哪个位置。

## min/max：最自然的 pattern

`min/max/minu/maxu` 是 RISC-VI 里最容易被 LLVM 接住的一组指令，因为 LLVM 中本来就有对应的语义节点。

在 RISC-V 后端初始化 lowering 行为时，项目把 `SMIN/SMAX/UMIN/UMAX` 标成 legal：

```cpp
if (Subtarget.hasStdExtZbb() || Subtarget.hasVendorXRiscVI() ||
    (Subtarget.hasVendorXCValu() && !Subtarget.is64Bit())) {
  setOperationAction({ISD::SMIN, ISD::SMAX, ISD::UMIN, ISD::UMAX}, XLenVT,
                     Legal);
}
```

这段逻辑对应 `llvm/lib/Target/RISCV/RISCVISelLowering.cpp:427` 附近。它是 `min/max` 能自然落到 TableGen pattern 的前置条件。

这段代码的含义是：如果打开了 RISC-VI feature，后端不必把 min/max 展开成比较和选择，而是允许 SelectionDAG 保留这些节点，交给后面的 pattern 匹配。

TableGen 里的 pattern 就很直接：

```tablegen
def : PatGprGpr<smin, XVI_MIN, i32>;
def : PatGprGpr<smax, XVI_MAX, i32>;
def : PatGprGpr<umin, XVI_MINU, i32>;
def : PatGprGpr<umax, XVI_MAXU, i32>;
```

这就是一个优秀 ISA 扩展最舒服的落点：LLVM 中间表示里已经有稳定语义，目标指令只要接住它。

## indexed memory：不是 TableGen 一行能解决的事

`lwxs/swxs` 对应的是数组索引访问：

```llvm
%p = getelementptr i32, ptr %base, i32 %idx
%v = load i32, ptr %p
```

在普通 RV32I 上，这类地址通常会变成：

```asm
slli t0, idx, 2
add  t0, base, t0
lw   rd, 0(t0)
```

RISC-VI 希望把地址生成链折叠成：

```asm
lwxs rd, base, idx, 2
```

这个选择不是只靠 TableGen pattern。LLVM RISC-V 后端里有专门的地址选择函数 `SelectAddrRegRegScale`。项目把 RISC-VI 接进了这条路径：

```cpp
if (!(VT.isScalarInteger() &&
      (Subtarget.hasVendorXTHeadMemIdx() || Subtarget.hasVendorXqcisls())) &&
    !(VT == MVT::i32 && Subtarget.hasVendorXRiscVI()) &&
    !((VT == MVT::f32 || VT == MVT::f64) &&
      Subtarget.hasVendorXTHeadFMemIdx()))
  return false;
```

源码位置在 `llvm/lib/Target/RISCV/RISCVISelDAGToDAG.cpp:3666` 到 `:3803`。其中 `:3674` 把 `Subtarget.hasVendorXRiscVI()` 纳入 indexed load/store 判断，`:3722` 开始的 `SelectAddrRegRegScale` 负责拆出 base、index 和 scale。

这段代码说明：当 memory value type 是 i32 且打开 `+xvi32r` 时，RISC-VI indexed memory 指令可以参与 reg+reg<<scale 的地址匹配。

`SelectAddrRegRegScale` 继续识别两类地址：

1. `base + (index << scale)`。
2. 没有显式 shift 时的 `base + index`，scale 视为 0。

它还会检查这个 `add` 是否值得折叠进 load/store，避免把被多个非内存用户共享的地址计算过早吞掉。这个细节很重要：指令选择不仅要“能匹配”，还要“值得匹配”。

## csel：selectcc 的目标化

`csel` 的目标是消除一些很小的控制流片段：

```text
cond ? true_value : false_value
```

在 RISC-V 后端里，这类语义会以 `riscv_selectcc` 的形式出现。RISC-VI 通过 pattern 接住两种条件方向：

```tablegen
def : Pat<(riscv_selectcc (i32 GPR:$cond), 0, SETNE,
                          (i32 GPR:$truev), GPR:$falsev),
          (XVI_CSEL GPR:$truev, GPR:$falsev, GPR:$cond)>;
def : Pat<(riscv_selectcc (i32 GPR:$cond), 0, SETEQ,
                          (i32 GPR:$falsev), GPR:$truev),
          (XVI_CSEL GPR:$truev, GPR:$falsev, GPR:$cond)>;
```

这里的核心是方向统一：不管 IR 里是 `cond != 0` 还是 `cond == 0`，最终都要映射到 `csel truev, falsev, cond` 的语义。

不过 `csel` 不是免费午餐。它可以减少小分支，但会引入三源寄存器读取。一个成熟的博客不能只写“指令数变少”，还要保留这个硬件代价。

## bchkltu：失败路径分支

`bchkltu` 最容易讲错，因为名字里有 `ltu`，但它跳转的是失败路径：

```text
bchkltu index, length, fail
```

语义是：

```c
if ((uint32_t)index >= (uint32_t)length) goto fail;
```

因此它匹配的是 `SETUGE`：

```tablegen
let AddedComplexity = 10 in
def : Pat<(riscv_brcc (i32 GPR:$idx), GPR:$len, SETUGE, bb:$target),
          (XVI_BCHKLTU GPR:$idx, GPR:$len, bare_simm13_lsb0_bb:$target)>;
```

`AddedComplexity = 10` 的作用是提高这个 pattern 的选择优先级，让它在合适的情况下胜过更普通的分支展开。

更深一层，`bchkltu` 还必须进入 RISC-V 后端的分支分析逻辑。项目把它归到 `COND_GEU`：

```cpp
case RISCV::BGEU:
case RISCV::XVI_BCHKLTU:
  return RISCVCC::COND_GEU;
```

相关 C++ 落点在 `llvm/lib/Target/RISCV/RISCVInstrInfo.cpp:1106`、`:1591` 和 `:1785`：它们分别涉及分支条件识别、分支反转约束和 branch offset 范围判断。

同时，分支范围判断里也把它和普通 13-bit B-type 分支放在同一类：

```cpp
case RISCV::XVI_BCHKLTU:
  return isInt<13>(BrOffset);
```

这说明一条分支指令不能只在 TableGen 里声明 `isBranch`。后端里所有“认识分支”的 C++ 逻辑，也要知道它。

## 如何判断 pattern 没生效

如果读者在自己的 LLVM 后端项目里遇到“定义了指令但生成不了”，一般按这个顺序排查：

1. IR 是否真的形成了目标语义，例如 `smin`、`select`、`getelementptr + load`。
2. lowering 阶段是否把目标节点合法化或保留下来。
3. TableGen pattern 是否带了正确 predicate。
4. 地址选择这类 ComplexPattern 是否真的匹配。
5. 指令是否被后续 pass 合法化、展开或替换掉。
6. asm printer 和 MC encoder 是否也认识该指令。

这个顺序比直接盯模拟器更有效。模拟器只能告诉你机器码执行结果，不能告诉你 LLVM 为什么没生成那条机器码。

## 项目里的验证入口

项目中用于观察指令选择结果的入口主要是 codegen case 和 cpu-test 汇编生成。它们适合有仓库的人复现；对博客读者来说，更重要的是理解它们对应的检查对象：IR 语义、DAG pattern、地址选择、MachineInstr、汇编输出。

## 小结

LLVM 指令选择的核心不是“我希望它发什么”，而是“中间表示中是否稳定出现这种语义，后端是否在正确层次接住它”。RISC-VI 的几类指令分别落在不同位置：min/max 走标准 DAG 节点，lwxs/swxs 依赖地址选择，csel 走 selectcc，bchkltu 还要进入分支分析。下一篇从工程角度看：新增一条指令到底要同步改多少层。
