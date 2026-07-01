---
title: "RISC-VI 研究笔记（七）：C 模拟器与 reference model"
date: 2026-04-19 15:03:13
tags:
  - "RISC-VI"
  - "simulator"
  - "difftest"
  - "RISC-V"
categories:
  - "Computer Architecture"
---

## 前言

ISA 实验必须有一个真值源。否则 LLVM 发出一条新指令后，很难判断错误来自哪里：是后端 pattern 选错了、MC 编码错了、模拟器语义错了，还是 RTL 实现错了。

RISC-VI 项目里的 C 模拟器承担的不是“跑个 demo”的角色，而是 reference model：

1. 它定义 RV32R 和 RISC-VI32R 的提交级 ISA 语义。
2. 它把程序运行结果变成结构化 JSON，服务自动化回归。
3. 它提供后续 RTL difftest 可以对齐的 step 接口。

![RISC-VI C 模拟器结构](/images/mermaid-svg/riscvi-07-simulator-reference-model/simulator-architecture.svg)

## reference model 的边界

一个好的 ISA reference model 应该清楚自己做什么，也清楚自己不做什么。

它应该做：

- 加载 flat binary；
- fetch 32 位指令；
- decode 出 op、寄存器号、立即数、目标地址；
- execute 并更新架构状态；
- 记录提交级 trace 和统计；
- 在 `ebreak` 或错误条件下停机。

它不应该假装自己是性能模型：

- 不建模真实 cache；
- 不建模分支预测器；
- 不建模乱序执行；
- 不直接给出 IPC 结论；
- 不把“动态指令数减少”等同于“最终性能提升”。

这个边界非常重要。C 模拟器证明的是语义正确性和提交级行为，RTL 才能讨论周期、阻塞、预测和读端口压力。

## decode：从 raw instruction 到结构化 inst

模拟器的 decode 入口在：

```text
riscv-vi-research/sim/common/decode.c
```

RISC-VI v0.1 的 decode 从 `riscv-vi-research/sim/common/decode.c:59` 开始，主入口 `decode_inst()` 在 `:159` 根据 ISA 模式和编码版本决定是否进入 RISC-VI decode。

RISC-VI v0.1 使用 `custom-0` opcode，并通过 `funct3` 区分 8 条指令。decode 逻辑大致是：

```c
if (isa == SIM_ISA_RISCVI32R) {
  if (xvi_encoding == SIM_XVI_ENCODING_V0_1 && opcode == OPCODE_CUSTOM0) {
    decode_riscvi_v01(&inst, raw);
    return inst;
  }
}
```

`decode_riscvi_v01()` 里会先拆出字段：

```c
inst->rd = bits(raw, 11, 7);
inst->rs1 = bits(raw, 19, 15);
inst->rs2 = bits(raw, 24, 20);
inst->scale = bits(raw, 26, 25);
inst->is_riscvi = true;
```

然后按 `funct3` 分发：

```c
case 0: inst->op = OP_XVI_LWXS; break;
case 1: inst->op = OP_XVI_SWXS; break;
case 2: inst->op = OP_XVI_MIN;  break;
case 6:
  inst->op = OP_XVI_CSEL;
  inst->rs3 = funct7 & 0x1fu;
  break;
case 7:
  inst->op = OP_XVI_BCHKLTU;
  inst->imm = imm_b(raw);
  inst->target = inst->pc + inst->imm;
  break;
```

这段代码体现了 decode 层的职责：它不执行语义，只把 raw bits 翻译成后续 execute 层能理解的结构化指令。

## execute：只维护架构状态

执行语义在：

```text
riscv-vi-research/sim/common/exec.c
```

RISC-VI 指令的 execute case 集中在 `riscv-vi-research/sim/common/exec.c:197` 到 `:230`。这个集中度有利于检查 LLVM TableGen 语义和 reference model 语义是否一致。

`lwxs` 的语义是地址折叠：

```c
case OP_XVI_LWXS:
  addr = rs1 + (rs2 << inst->scale);
  write_rd(cpu, inst, load_value(cpu, inst, addr));
  cpu->pc = inst->snpc;
  break;
```

`swxs` 类似，但存储值来自 `rd` 字段对应的寄存器：

```c
case OP_XVI_SWXS:
  addr = rs1 + (rs2 << inst->scale);
  store_value(cpu, inst, addr, cpu->gpr[inst->rd]);
  cpu->pc = inst->snpc;
  break;
```

`csel` 是三源选择：

```c
case OP_XVI_CSEL:
  write_rd(cpu, inst, rs3 != 0 ? rs1 : rs2);
  cpu->pc = inst->snpc;
  break;
```

`bchkltu` 保留为普通条件分支：

```c
case OP_XVI_BCHKLTU:
  taken = rs1 >= rs2;
  execute_branch(cpu, inst, taken);
  break;
```

这几段代码和 LLVM TableGen 的语义必须严格一致。比如 `bchkltu` 如果在编译器里按 `idx >= len` 生成，在模拟器里却按 `idx < len` 跳转，端到端测试可能直接跑飞。

## 统计字段为什么重要

模拟器在执行时记录 load/store/branch/RISC-VI 指令数量、UART 输出、最终寄存器状态等信息。这些字段有两个作用。

第一，它们让功能测试自动化。程序不是“看起来跑完了”，而是以 `ebreak`、`final_a0`、UART 缓冲和退出原因形成可检查结果。

第二，它们为 RTL 评测建立命名口径。比如模拟器里 `bchkltu` 仍然算 branch，这会提醒后续 RTL 不能把它当作“无控制代价”的魔法指令。

## shared library API 的意义

项目还冻结了 reference model API，例如：

```text
riscvi_ref_init
riscvi_ref_load_file
riscvi_ref_step
riscvi_ref_get_regs
```

这些入口在 `riscv-vi-research/sim/common/ref_api.c:111`、`:163`、`:183` 和 `:222` 实现。`riscvi_ref_step()` 每次最多推进一条提交级指令，是后续 RTL 对拍的核心边界。

核心是 `step()`：每次推进一条提交级指令。RTL difftest 可以按提交顺序比较寄存器、PC、内存事件和分支结果。

这里再次体现边界：API 对齐的是提交级语义，不要求每个 cycle 和 RTL 完全一致。RTL 可以有流水线、旁路、stall、flush；reference model 只关心最终提交的架构状态。

## 项目里的验证入口

项目里模拟器相关检查覆盖功能测试、difftest API 检查和 trace smoke。它们不是文章阅读前提，而是说明 reference model 需要同时覆盖功能、API 和 trace 三个面向。只跑一个小程序，不能证明模拟器已经适合作为后续 RTL 对拍基准。

## 小结

C 模拟器是 RISC-VI 项目的语义锚点。LLVM 负责生成机器码，AM/BSP 负责承载裸机程序，RTL 负责周期级实现，而模拟器负责回答最根本的问题：这条指令提交后，架构状态应该是什么。下一篇进入 AM/BSP，看裸机 C 程序如何启动、输出和停机。
