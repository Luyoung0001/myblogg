---
title: "RISC-VI 研究笔记（三）：用 TableGen 定义一条真实机器指令"
date: 2026-04-15 15:03:13
tags:
  - "RISC-VI"
  - "LLVM"
  - "TableGen"
  - "backend"
categories:
  - "Computer Architecture"
---

## 前言

LLVM 后端里，一条机器指令不是一个简单的字符串。它至少要告诉 LLVM：

- 这条指令怎么编码；
- 有哪些输入输出操作数；
- 是否读写内存；
- 是否是分支或 terminator；
- 什么时候可以被选择；
- 对调度模型来说需要哪些读写资源；
- 哪些 LLVM IR 或 DAG 模式可以降成它。

这些信息大部分写在 TableGen 里。RISC-VI 的核心文件是：

```text
llvm/lib/Target/RISCV/RISCVInstrInfoXVI.td
```

![TableGen 机器指令的组成](/images/mermaid-svg/riscvi-03-tablegen-instruction/tablegen-anatomy.svg)

这张图要表达三件事：

1. 指令定义不仅有名字和汇编格式。
2. `mayLoad/mayStore/isBranch/isTerminator` 这类属性会影响后端 pass。
3. IR pattern 决定 LLVM 什么时候会主动发出这条指令。

## 从 `min` 开始

`min` 是最适合入门的一条 RISC-VI 指令。它是普通的双源一目的 ALU 指令：

```text
min rd, rs1, rs2
```

语义是：

```c
rd = (int32_t)rs1 < (int32_t)rs2 ? rs1 : rs2;
```

它没有内存访问，不是分支，也没有副作用。所以可以定义一个通用 ALU 模板：

```tablegen
let hasSideEffects = 0, mayLoad = 0, mayStore = 0 in
class XVI_ALU_rr<bits<7> funct7, bits<3> funct3, string opcodestr,
                 bit Commutable = 0>
    : RVInstR<funct7, funct3, OPC_CUSTOM_0, (outs GPR:$rd),
              (ins GPR:$rs1, GPR:$rs2), opcodestr, "$rd, $rs1, $rs2"> {
  let isCommutable = Commutable;
}
```

然后定义真实指令：

```tablegen
let Predicates = [HasVendorXRiscVI] in {
def XVI_MIN  : XVI_ALU_rr<0b0000000, 0b010, "min", Commutable=1>,
               Sched<[WriteIMinMax, ReadIMinMax, ReadIMinMax]>;
}
```

这里 `HasVendorXRiscVI` 的意思是：只有打开 `+xvi32r` 时，这条指令才可用。

## pattern：让 LLVM 主动选择它

光定义指令还不够。你还要告诉 LLVM：看到什么 IR/DAG 节点时，可以选它。

对于 `min/max/minu/maxu`，LLVM 已经有比较稳定的节点：

```tablegen
let Predicates = [HasVendorXRiscVI, IsRV32] in {
def : PatGprGpr<smin, XVI_MIN, i32>;
def : PatGprGpr<smax, XVI_MAX, i32>;
def : PatGprGpr<umin, XVI_MINU, i32>;
def : PatGprGpr<umax, XVI_MAXU, i32>;
}
```

这就是为什么 `min/max` 是第一批比较容易落地的指令：LLVM 的中间表示里本来就有稳定模式。

## load/store 和 branch 要更小心

`lwxs/swxs` 不是普通 ALU 指令。它们是真正访问内存的指令，因此 TableGen 里必须设置：

```tablegen
mayLoad = 1
mayStore = 1
```

如果把真实内存操作伪装成普通 ALU，后续优化 pass 可能会错误地重排它。

`bchkltu` 也不是普通指令。它是条件分支，所以要有：

```tablegen
let isBranch = 1;
let isTerminator = 1;
```

这些属性直接影响基本块、控制流和 branch analyze。

## 编译和验证

先跑集中检查：

```bash
cd /home/luyoung/llvm-project/riscv-vi-research

make llvm-codegen-cases
make llvm-codegen-real-check
make llvm-mc-smoke
```

也可以手动检查：

```bash
/home/luyoung/llvm-project/build-riscvi-llvm/bin/llc \
  -mtriple=riscv32 -mattr=+xvi32r \
  cases/llvm_codegen/xvi_minmax.ll -o -
```

负向检查：

```bash
/home/luyoung/llvm-project/build-riscvi-llvm/bin/llc \
  -mtriple=riscv32 \
  cases/llvm_codegen/xvi_minmax.ll -o -
```

## 常见错误

第一类是 operand 顺序错误。TableGen 里 asm string 写错，输出汇编就会看着很怪。

第二类是 encoding 对不上。比如 `funct3` 和模拟器 decode 里的值不一致，LLVM 能发、MC 能汇编，但模拟器执行成另一条指令。

第三类是属性漏写。`mayLoad/mayStore/isBranch` 这类字段看起来像标记，实际上会影响后端行为。

## 小结

TableGen 是 LLVM 后端工程里非常关键的一层。它不是简单的“指令表”，而是机器指令语义、编码、调度和选择规则的集中描述。下一篇继续看：这些 TableGen pattern 到底如何把 LLVM IR 变成 `lwxs/min/csel/bchkltu`。
