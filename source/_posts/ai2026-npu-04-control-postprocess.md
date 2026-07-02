---
title: "AI2026 NPU 设计笔记（四）：控制状态机、激活和池化如何串成算子链"
date: 2026-05-01 15:03:13
tags:
  - "AI2026"
  - "NPU"
  - "control unit"
  - "post-processing"
categories:
  - "Computer Architecture"
---

## 前言

如果说 PE Array 是肌肉，Unified Buffer 是短期记忆，那么 Control Unit 就是 NPU 的神经系统。它不直接做乘法，却决定什么时候搬数据、什么时候启动阵列、什么时候后处理、什么时候对外输出。

NPU 设计里最容易被低估的部分就是控制。算法描述往往只有一句 `Y = pool(relu(A x B))`，硬件却要把它拆成命令、状态、握手、读写和完成信号。

![控制状态机与后处理链路](/images/mermaid-svg/ai2026-npu-04-control-postprocess/control-postprocess.svg)

## 命令格式：把软件意图压成硬件字段

AI2026 的 Control Unit 使用 32 位命令格式：

![NPU 命令字段与状态机](/images/mermaid-svg/ai2026-npu-04-control-postprocess/command-fsm.svg)

```text
[31:24] opcode
[23:12] addr/base
[19:16] extra
[11:0]  len/rows
```

这个格式写在 `AI2026/npu-project/rtl/control/control_unit.sv:303` 到 `:307`。opcode 表在同文件 `:309` 到 `:317`，覆盖 `LOAD_W`、`LOAD_A`、`MATMUL`、`ACT`、`POOL`、`STORE`、`WAIT`。

命令格式的价值不在于它有多复杂，而在于它建立了软硬件边界：

1. 软件只描述“要做什么”和“数据在哪里”。
2. 控制单元把命令翻译成状态迁移。
3. 顶层数据通路根据控制信号执行。

这比直接从 testbench 拉一堆内部信号更像真实系统。

## 命令 FIFO：为什么不能只接一条线

Control Unit 前面有一个命令 FIFO。FIFO 深度默认 16，定义在 `AI2026/npu-project/rtl/control/control_unit.sv:10` 到 `:12`；实例化在同文件 `:91` 到 `:113`。

FIFO 的存在说明主机和 NPU 不是完全同步的。主机可能连续发命令，NPU 内部却要花多个周期执行一个 `MATMUL` 或 `STORE`。用 FIFO 可以让输入侧和执行侧解耦。

`cmd_ready = !cmd_fifo_full` 写在 `AI2026/npu-project/rtl/control/control_unit.sv:115`。这就是一个很小但很关键的 backpressure 协议：FIFO 满了，主机就不应该继续塞命令。

## 主状态机：从 IDLE 到 DONE

Control Unit 的状态定义在 `AI2026/npu-project/rtl/control/control_unit.sv:81` 到 `:89`：

- `STATE_IDLE`
- `STATE_FETCH`
- `STATE_DECODE`
- `STATE_DMA_LOAD`
- `STATE_MATMUL`
- `STATE_ACT`
- `STATE_POOL`
- `STATE_DMA_STORE`
- `STATE_DONE`

主状态机在 `AI2026/npu-project/rtl/control/control_unit.sv:153` 到 `:212`。它的结构很清楚：

1. `IDLE` 看到 FIFO 非空后进入 `FETCH`。
2. `FETCH` 发起读取，下一拍进入 `DECODE`。
3. `DECODE` 根据 opcode 选择 DMA、MATMUL、ACT、POOL 或 STORE。
4. 具体执行状态等待对应完成信号。
5. `DONE` 回到 `IDLE`。

这里有一个值得注意的 RTL 细节：FIFO 是同步读，所以 `FETCH` 发起读取，`DECODE` 周期才使用读出的命令。这个注释写在 `AI2026/npu-project/rtl/control/control_unit.sv:127`。

这类细节很工程。写状态机时如果忘记同步读延迟，最常见的 bug 就是 decode 到上一条命令、空命令或未稳定命令。

## 控制信号如何驱动数据通路

Control Unit 不直接访问 UB 或 PE 里的每根线，而是产生少量控制信号。

`MATMUL` 命令会拉起 `pe_start`，并设置 A/B/C 地址、行列参数，见 `AI2026/npu-project/rtl/control/control_unit.sv:218` 到 `:237`。

`ACT` 命令会输出 `act_type` 和 `act_enable`，见 `AI2026/npu-project/rtl/control/control_unit.sv:239` 到 `:250`。

`POOL` 命令会输出 `pool_enable` 和 `pool_type`，见 `AI2026/npu-project/rtl/control/control_unit.sv:252` 到 `:263`。

`LOAD` 和 `STORE` 通过 DMA 控制信号表达，见 `AI2026/npu-project/rtl/control/control_unit.sv:265` 到 `:295`。

这就是控制单元的边界：它不关心 PE 里面怎么乘，也不关心 UB 里每个地址怎样实现；它只负责发起动作和等待完成。

## Activation Unit：非线性从哪里来

神经网络不能只有线性矩阵乘法。否则多层线性层叠起来仍然是线性变换，表达能力不够。

Activation Unit 支持 ReLU、LeakyReLU、Sigmoid、Tanh，类型定义在 `AI2026/npu-project/rtl/control/activation_unit.sv:35` 到 `:40`。

最常用的 ReLU 很简单：如果输入是负数，输出 0；否则输出原值。RTL 写在 `AI2026/npu-project/rtl/control/activation_unit.sv:52` 到 `:55`。LeakyReLU 做带斜率的负半轴处理，见同文件 `:56` 到 `:62`。Sigmoid 和 Tanh 在这个 demo 里采用 LUT 风格，查表数组定义在 `:67` 到 `:83`。

真正的选择逻辑在 `AI2026/npu-project/rtl/control/activation_unit.sv:98` 到 `:107`。输出寄存器让 `valid_out` 跟随 `valid_in`，见同文件 `:113` 到 `:120`。

这里的工程取舍也很典型：ReLU 可以直接组合逻辑实现；Sigmoid/Tanh 如果精确实现会很贵，所以教学 demo 用 LUT mental model 表达它们。

## Pooling Unit：降采样不是随手取个数

Pooling Unit 支持 Max Pooling 和 Average Pooling，注释在 `AI2026/npu-project/rtl/control/pooling_unit.sv:4` 到 `:6`。默认窗口是 2x2，步长是 2，见同文件 `:9` 到 `:14`。

Pooling 的本质是把一个局部窗口压成一个输出值。项目里用 `window_buf` 收集窗口数据，见 `AI2026/npu-project/rtl/control/pooling_unit.sv:41` 到 `:58`；窗口数据写入逻辑在 `:63` 到 `:80`。

Max 和 Average 的选择写在 `AI2026/npu-project/rtl/control/pooling_unit.sv:125` 到 `:133`。输出有效信号要等窗口完成，`window_done` 定义在同文件 `:139` 到 `:151`。

这提醒我们，pooling 不是简单“每两个点取一个点”。它需要窗口边界、计数、valid 和输出节拍配合，否则容易出现错位。

## 顶层如何把 ACT/POOL 落到 UB 上

`npu_top.sv` 里没有把后处理写成抽象函数调用，而是把它拆成状态机。

后处理状态包括 `POST_ACT_READ_REQ`、`POST_ACT_READ_WAIT`、`POST_ACT_PIPE`、`POST_ACT_WRITE`、`POST_POOL_READ_REQ`、`POST_POOL_READ_WAIT`、`POST_POOL_WRITE`，定义在 `AI2026/npu-project/rtl/top/npu_top.sv:157` 到 `:164`。

对于 ACT，顶层从 UB 读一个结果，等待同步读返回，把数据送进 activation pipeline，再写回同一结果区域。这个流程在 `AI2026/npu-project/rtl/top/npu_top.sv:609` 到 `:654`。

对于 POOL，顶层要读 2x2 窗口的四个值，调用 `apply_pool_window` 得到最大值或平均值，再写回压缩后的结果。函数定义在 `AI2026/npu-project/rtl/top/npu_top.sv:206` 到 `:235`，POOL 状态机在同文件 `:656` 到 `:694`。

这里有一个很漂亮的教学点：单独的 `pooling_unit.sv` 讲模块原理，`npu_top.sv` 则展示“在真实结果 buffer 上如何调度后处理”。前者是算法单元，后者是系统集成。

## 整机 testbench 检查的算子链

`npu_tb.sv` 不是只测矩阵乘法。它的测试内容注释里列了 PE Array 矩阵乘法、激活函数和数据流控制，见 `AI2026/npu-project/sim/tb/npu_tb.sv:4` 到 `:7`。

更具体地说，它覆盖了三类链路：

1. `MATMUL + STORE`：B 是单位矩阵，结果应等于 A，见 `AI2026/npu-project/sim/tb/npu_tb.sv:215` 到 `:287`。
2. `MATMUL + ACT(ReLU) + STORE`：带符号输入经过 ReLU 后输出为 0，见同文件 `:289` 到 `:336`。
3. `MATMUL + POOL(Max) + STORE`：矩阵结果经过 2x2 max pooling，见同文件 `:338` 到 `:384`。

这比单独测一个模块更接近真实使用方式。NPU 的价值不只是某个模块能工作，而是算子链在命令、buffer、状态机和输出协议之间能闭合。

## 这篇的核心结论

控制单元把 NPU 从“计算电路”变成“可被主机驱动的协处理器”。

AI2026 这个实现里，值得抓住三点：

1. 32 位命令格式把软件意图压成 opcode、地址和长度字段。
2. Control Unit 用 FIFO 和 FSM 解耦主机输入与内部执行。
3. 顶层把 ACT/POOL 实现成对 UB 的原地读改写，而不是纸面上的函数调用。

下一篇我们收束到验证：一个 NPU demo 怎样用 Verilator、模块级 testbench 和波形把这些层次验证起来。
