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

ISA 实验必须有一个真值源。否则 LLVM 发出一条指令之后，你很难判断到底是后端错了、汇编错了、模拟器错了，还是测试程序本身错了。

RISC-VI 项目里的 C 模拟器承担三个角色：

1. 命令行执行器：运行 flat binary，输出 JSON。
2. ISA 语义模型：实现 RV32R 和 RISC-VI32R 指令语义。
3. 后续 RTL difftest 的 reference model：提供 shared library API。

![RISC-VI C 模拟器结构](/images/mermaid-svg/riscvi-07-simulator-reference-model/simulator-architecture.svg)

这张图要表达三件事：

1. 模拟器分为 loader、fetch、decode、execute、stats 几个部分。
2. decode 负责从 raw instruction 得到结构化指令。
3. execute 只维护架构状态，不建模真实流水线周期。

## decode 和 execute 分离

`decode.c` 做的事情是把 32 位 raw instruction 拆成 `inst_t`。RISC-VI v0.1 的入口类似：

```c
if (xvi_encoding == SIM_XVI_ENCODING_V0_1 && opcode == OPCODE_CUSTOM0) {
  decode_riscvi_v01(&inst, raw);
  return inst;
}
```

在 v0.1 中，8 条指令都在 `custom-0` 里，用 `funct3` 区分：

- `000`：`lwxs`
- `001`：`swxs`
- `010`：`min`
- `011`：`max`
- `100`：`minu`
- `101`：`maxu`
- `110`：`csel`
- `111`：`bchkltu`

`exec.c` 负责真正修改架构状态。例如 `lwxs/swxs`：

```c
case OP_XVI_LWXS:
  addr = rs1 + (rs2 << inst->scale);
  write_rd(cpu, inst, load_value(cpu, inst, addr));
  cpu->pc = inst->snpc;
  break;

case OP_XVI_SWXS:
  addr = rs1 + (rs2 << inst->scale);
  store_value(cpu, inst, addr, cpu->gpr[inst->rd]);
  cpu->pc = inst->snpc;
  break;
```

## 模拟器不做什么

它不负责预测周期，不负责建模乱序，不负责 cache，也不负责真实分支预测状态。它的职责是提交级正确性。

这点很重要。后面如果看到 C 模拟器里的动态指令数减少，不能直接说 IPC 提升。IPC 必须到 RTL 计数模型里讨论。

## CLI 模式

构建和测试：

```bash
cd /home/luyoung/llvm-project/riscv-vi-research

make sim
make sim-test
```

手动运行一个 cpu-test binary：

```bash
./sim/riscvi32r-sim \
  --isa riscvi32r \
  --bin build/cpu-test/riscvi32r/bounds-check.bin \
  --program bounds-check \
  --max-inst 1000000 \
  --expect-halt ebreak \
  --expect-a0 0
```

## reference model API

模拟器还提供 shared library 入口，后续 RTL difftest 可以这样使用：

```text
riscvi_ref_init
riscvi_ref_load_file
riscvi_ref_step
riscvi_ref_get_regs
```

这个 API 的关键是 `step()`：每次最多提交一条指令。这是提交级对齐口径，不是周期级口径。

检查：

```bash
make sim-difftest-api-check
make sim-trace-smoke
```

## 常见错误

如果模拟器输出 `unimplemented op`，优先检查：

- ISA 模式是否是 `riscvi32r`；
- 编码版本是否一致；
- decode 是否识别 `funct3/opcode`；
- exec 是否实现了目标 op。

如果 trace 里没有目标 RISC-VI 指令，先回到汇编检查，确认 LLVM 或 AM wrapper 真的生成了它。

## 小结

C 模拟器是 RISC-VI 项目的语义锚点。它让 LLVM 发射、AM runtime、cpu-test 和 RTL difftest 都可以围绕同一个 reference model 对齐。下一篇进入 AM/BSP：裸机 C 程序到底是怎么启动和停机的。
