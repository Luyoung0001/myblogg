---
title: "AI2026 NPU 设计笔记（五）：用 Verilator 做分层验证和波形调试"
date: 2026-05-02 15:03:13
tags:
  - "AI2026"
  - "NPU"
  - "Verilator"
  - "verification"
categories:
  - "Computer Architecture"
---

## 前言

RTL 项目最容易出现一种错觉：模块写完了，仿真能跑一下，就算完成。真正靠谱的验证应该分层：先锁住单 PE，再锁住阵列数据流，再锁住控制单元，最后看整机算子链。

AI2026 的 `npu-project` 是一个适合教学的例子。它不追求 FPGA 下板，而是在 Verilator 仿真中展示 NPU 常见部件如何协同工作，项目定位见 `AI2026/npu-project/README.md:1` 到 `:3`。

![NPU Verilator 分层验证](/images/mermaid-svg/ai2026-npu-05-verilator-validation/verilator-validation.svg)

## 验证不是“看见输出”这么简单

NPU 的 bug 往往不是“完全没有输出”，而是更隐蔽：

1. 有符号乘法被当成无符号。
2. A/B lane 顺序反了。
3. 同步读多等或少等了一拍。
4. `valid` 和数据错位。
5. ACT/POOL 写回地址覆盖了错误位置。
6. STORE 读出了后处理前的旧数据。

所以验证不能只盯最终一串数字。它要能回答：错了以后在哪一层定位。

AI2026 的 NPU 仿真入口也按这个思路组织。`sim/Makefile` 列出的 RTL 文件包括 PE Array、activation、pooling、Unified Buffer、Control Unit 和 `npu_top`，见 `AI2026/npu-project/sim/Makefile:24` 到 `:35`；testbench 包括 full system、PE array、control unit，见同文件 `:37` 到 `:41`。

## 第一层：单 PE 合约

单 PE 最小验证点包括基本乘法、累加、有符号数和连续累加。`single_pe_tb.sv` 开头直接写明测试目标是 `y = y_in + a * b`，见 `AI2026/npu-project/sim/tb/single_pe_tb.sv:1` 到 `:5`。

它的测试内容很有代表性：

- 简单乘法 `2 * 3 = 6`，见 `AI2026/npu-project/sim/tb/single_pe_tb.sv:75` 到 `:107`。
- 累加 `2 * 3 + 6 = 12`，见同文件 `:109` 到 `:137`。
- 负数乘法 `-2 * 3 = -6`，见同文件 `:139` 到 `:170`。
- 连续累加 `1*1 + 2*2 + 3*3 = 14`，见同文件 `:172` 到 `:212`。

这些测试看似小，但它们把 PE 的数值语义固定住了。后面如果阵列结果错了，至少可以先排除“单个 MAC 算错”这一层。

## 第二层：阵列数据流

阵列验证的核心不是重新证明乘法，而是证明数据 layout、valid 节拍和累加维度正确。

`pe_array_4x4_simple_tb.sv` 的注释非常明确：A 的列和 B 的行连续发送，每个周期一对，让数据在阵列中流动和相遇，见 `AI2026/npu-project/sim/tb/pe_array_4x4_simple_tb.sv:2` 到 `:8`。

测试里连续发送四组 A 列和 B 行，见 `AI2026/npu-project/sim/tb/pe_array_4x4_simple_tb.sv:82` 到 `:110`。发送完以后等待若干周期，让数据流过阵列并完成计算，见同文件 `:117` 到 `:121`。

这层验证最适合看波形。你应该关心的不是“最终矩阵打印得好不好看”，而是：

1. 每一拍 `a_valid/b_valid` 是否和输入数据对齐。
2. `clear_acc` 是否只在新任务开始时发生。
3. `y_reg[row][col]` 是否沿 k 维累加。
4. `c_valid/c_last` 是否在结果稳定后出现。

项目里的阵列 RTL 在 `a_valid_r && b_valid_r` 时累加 `y_reg[row][col]`，见 `AI2026/npu-project/rtl/common/pe_array_16x16.sv:136` 到 `:147`。波形要对的，正是这一段语义。

## 第三层：控制单元

控制单元 testbench 的价值是把命令解码和状态机单独拿出来看。

`control_unit_tb.sv` 里定义了命令通道、状态输出、PE 控制、ACT/POOL 控制、DMA 控制等信号，见 `AI2026/npu-project/sim/tb/control_unit_tb.sv:23` 到 `:53`。它用 `send_cmd` task 按 `cmd_ready` 协议发送命令，见同文件 `:98` 到 `:108`。

测试内容覆盖：

1. IDLE 状态检查，见 `AI2026/npu-project/sim/tb/control_unit_tb.sv:152` 到 `:161`。
2. `LOAD_W` 是否触发 DMA 写，见同文件 `:163` 到 `:189`。
3. `MATMUL` 是否触发 `pe_start`，见同文件 `:191` 到 `:209`。
4. `pe_done` 后能否回到 IDLE，见同文件 `:211` 到 `:223`。
5. `ACT` 是否打开激活单元，见同文件 `:225` 到 `:243`。

这类测试的重点是“控制信号是否符合协议”。它不会证明矩阵算对，但能证明 opcode 到状态/控制信号的路径没有断。

## 第四层：整机算子链

整机 testbench 用 `npu_top` 串起命令、数据输入、输出捕获、期望值比较。它实例化顶层的地方在 `AI2026/npu-project/sim/tb/npu_tb.sv:56` 到 `:84`。

这里的目标不是让读者在终端复现，而是理解整机验证的检查点：

1. `LOAD_W` 和 `LOAD_A` 把矩阵写入 UB。
2. `MATMUL` 触发阵列计算并提交结果。
3. `ACT` 或 `POOL` 在 UB 结果区上做后处理。
4. `STORE` 把结果流出。
5. testbench 捕获输出并和期望矩阵比较。

`npu_tb.sv` 的 `check_outputs` task 在输出数组和期望数组之间逐项比较，见 `AI2026/npu-project/sim/tb/npu_tb.sv:171` 到 `:191`。波形导出则覆盖顶层、PE Array 和 Control Unit，见同文件 `:404` 到 `:413`。

这就是分层验证的最后一层：既看最终结果，又能在波形里追到内部模块。

## Verilator 在这个项目里的角色

`npu-project/sim/Makefile` 采用 Verilator-only 的仿真流。顶部注释列出特性：16x16 INT8 PE array、on-chip buffer、激活、池化、Verilator simulation，见 `AI2026/npu-project/sim/Makefile:1` 到 `:12`。

Verilator 配置启用了 trace、trace depth、struct trace、timing 和 timescale，见 `AI2026/npu-project/sim/Makefile:43` 到 `:54`。这说明项目不是只追求“编译成 C++ 跑完”，还希望能保留足够波形细节用于调试。

Makefile 里 full system、PE array、single PE、control unit 都有独立构建入口。比如 full system 目标把 RTL 文件和 `npu_tb.sv` 一起交给 Verilator，并指定 `npu_tb` 为 top module，见 `AI2026/npu-project/sim/Makefile:88` 到 `:103`。单 PE 和控制单元的入口分别在同文件 `:141` 到 `:163`、`:169` 到 `:196`。

文章里不需要把这些写成读者操作步骤，但它们作为工程证据很重要：项目不是靠一个“大仿真”硬撑，而是把验证入口拆到了模块层。

## 好的 NPU 波形应该看什么

调 NPU 波形时，不要一打开就盯满屏信号。可以按数据生命周期看：

1. 命令阶段：`cmd_valid/cmd_ready`、opcode、state。
2. LOAD 阶段：DMA 写入 UB 的地址和数据。
3. MATMUL 阶段：`pe_start`、`clear_acc`、A/B 发送、`pe_a_valid/pe_b_valid`。
4. COMMIT 阶段：`pe_c_data` 被逐项写回 UB。
5. ACT/POOL 阶段：读地址、等待拍、写回地址。
6. STORE 阶段：输出 valid/data 是否和期望数量一致。

`npu_top.sv` 里这些阶段都有对应状态或控制信号。比如 MATMUL 状态在 `AI2026/npu-project/rtl/top/npu_top.sv:151` 到 `:156`，后处理状态在 `:157` 到 `:164`，结果 IO 状态在 `:165` 到 `:168`。

这种观察顺序比“全局搜索哪个信号不对”更可靠。你是在跟踪数据生命周期，而不是在波形海里摸索。

## 这篇的核心结论

AI2026 的 NPU 验证体系有三个优点：

1. 从单 PE、阵列、控制单元到整机算子链逐层收敛。
2. Verilator 配置保留了波形调试所需的结构信息。
3. testbench 不只看有没有输出，还比较结果、观察状态和导出内部波形。

这组 NPU 文章到这里形成了一个闭环：原理、PE Array、Unified Buffer、控制后处理、分层验证。接下来可以进入 AI2026 的另一个更偏系统平台的系列：CoolDA 如何把 CPU、BSP、runtime、APB NPU 和 Verilator shell 串成一个最小加速平台。
