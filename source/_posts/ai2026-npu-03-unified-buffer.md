---
title: "AI2026 NPU 设计笔记（三）：Unified Buffer 为什么是本地性边界"
date: 2026-04-30 15:03:13
tags:
  - "AI2026"
  - "NPU"
  - "memory hierarchy"
  - "RTL"
categories:
  - "Computer Architecture"
---

## 前言

如果只看 PE Array，NPU 会显得很热闹：每个 PE 都在乘加，阵列里到处是并行度。但真正限制一个加速器的，往往不是“能不能算”，而是“数据能不能及时、低成本地到达计算单元”。

这就是 Unified Buffer 的位置。它不是普通 SRAM 的别名，而是 NPU 内部最重要的本地性边界：哪些数据已经在片上、哪些数据需要外部搬运、哪些中间结果可以原地后处理，都由它决定。

![Unified Buffer 与本地性边界](/images/mermaid-svg/ai2026-npu-03-unified-buffer/unified-buffer.svg)

## 为什么 NPU 必须关心本地性

矩阵乘法看起来只是三重循环：

```text
for i
  for j
    for k
      C[i][j] += A[i][k] * B[k][j]
```

但硬件实现时，真正昂贵的不是这几层循环本身，而是 A、B、C 数据被搬动的次数。

如果每次乘加都从外部存储读取 A 和 B，再把 partial sum 写回外部存储，阵列再大也会被带宽喂不饱。NPU 设计的一个核心目标，就是尽量让数据进入片上以后被多次复用。

Unified Buffer 承担的就是这件事：它把权重、激活、中间结果和输出放在 NPU 可控的片上空间里。项目文件的注释直接列出这四类数据，见 `AI2026/npu-project/rtl/control/unified_buffer.sv:4` 到 `:8`。

## 这个 UB 的结构

`unified_buffer.sv` 的参数很朴素：地址宽度 12，数据宽度 32，深度 4096，见 `AI2026/npu-project/rtl/control/unified_buffer.sv:11` 到 `:15`。

这意味着它用 4096 个 32 位 entry 作为教学尺度下的片上存储。项目注释把地址空间划成四段：

![Unified Buffer 地址分区](/images/mermaid-svg/ai2026-npu-03-unified-buffer/ub-address-map.svg)

- `0x000 - 0x3FF`：Weight Buffer。
- `0x400 - 0x7FF`：Activation Buffer。
- `0x800 - 0xBFF`：Partial Sum。
- `0xC00 - 0xFFF`：Output Buffer。

这张表在 `AI2026/npu-project/rtl/control/unified_buffer.sv:136` 到 `:143`。

注意这个映射的意义。它不是为了把地址写得漂亮，而是把 NPU 的数据生命周期显式分区：

1. 权重和激活是输入。
2. partial sum 是计算中的状态。
3. output 是对外可见的结果。

当你开始讨论 DMA、tile、double buffering 或多 batch 时，这种分区会变得更重要。

## 三个端口对应三类访问者

这个 UB 有三个逻辑端口：

1. 端口 A 给 PE Array 使用。
2. 端口 B 给 DMA 读写。
3. 端口 C 给控制单元访问。

端口定义写在 `AI2026/npu-project/rtl/control/unified_buffer.sv:19` 到 `:44`。内部存储体是一个 `mem` 数组，定义在同文件 `:47` 到 `:53`。

端口 A、B、C 都采用同步读写方式。比如端口 A 在 `a_en && !a_we` 时从 `mem[a_addr]` 读出，见 `AI2026/npu-project/rtl/control/unified_buffer.sv:60` 到 `:67`；写操作则在 `a_en && a_we` 时发生，见同文件 `:70` 到 `:74`。端口 B 和端口 C 的读写结构分别在 `:77` 到 `:109`。

这类同步 SRAM 风格很贴近真实 RTL。它也带来一个工程事实：读数据通常不是组合逻辑立刻出现，而是需要一个时钟边界。这一点会影响顶层状态机。

## 顶层如何处理同步读

`npu_top.sv` 明确把 Unified Buffer 实例化出来，端口 A 连接 PE Array 读路径，端口 B 连接数据写入/STORE，端口 C 用于结果提交和后处理，见 `AI2026/npu-project/rtl/top/npu_top.sv:340` 到 `:368`。

真正值得看的，是顶层对同步读的处理。`npu_top.sv` 在 MATMUL 数据流控制前写了一段注释：Unified Buffer 是同步读，所以要显式拆成地址稳定、捕获 `dout`、发送到 PE Array 三个阶段，见 `AI2026/npu-project/rtl/top/npu_top.sv:496` 到 `:500`。

这就是 RTL 和抽象算法的差别。算法可以写：

```text
read A[k], read B[k], send to array
```

RTL 里却要关心：

1. 哪一拍给地址。
2. 哪一拍 `dout` 有效。
3. 哪一拍把数据送进阵列。
4. valid 信号在哪一拍拉高。

项目把 MATMUL 状态拆成 `MATMUL_CLEAR`、`MATMUL_CAPTURE`、`MATMUL_SEND`、`MATMUL_DRAIN`、`MATMUL_COMMIT`，定义在 `AI2026/npu-project/rtl/top/npu_top.sv:151` 到 `:156`。这正是同步存储带来的工程结构。

## MATMUL 读流：从 UB 到 PE Array

当控制单元发出 `pe_start` 后，顶层会锁存 A、B、C 的基地址，限制本次 matmul 长度，并拉起 `clear_acc_r` 清空阵列累加，见 `AI2026/npu-project/rtl/top/npu_top.sv:538` 到 `:557`。

随后在 `MATMUL_CAPTURE` 状态里，顶层把 `pe_a_word` 和 `pe_b_word` 捕获到发送寄存器，并预取下一组地址，见 `AI2026/npu-project/rtl/top/npu_top.sv:564` 到 `:576`。在 `MATMUL_SEND` 状态，`pe_a_valid` 和 `pe_b_valid` 被拉高，见 `AI2026/npu-project/rtl/top/npu_top.sv:423` 到 `:432`。

这个过程说明 UB 不只是静态存储，它直接参与了流水节拍设计。PE Array 的利用率，很大程度取决于 UB 读路径能不能按节奏供应数据。

## 结果为什么要再写回 UB

很多教学图会画成：

```text
PE Array -> output
```

但实际项目里结果不会马上丢到外部。PE Array 输出 `pe_c_data` 后，顶层在 `MATMUL_COMMIT` 阶段按顺序把结果写回 buffer C 口。写回发生在 `AI2026/npu-project/rtl/top/npu_top.sv:749` 到 `:753`。

为什么要这么做？

因为后面还有 `ACT` 和 `POOL`。如果 MATMUL 结果直接流走，后处理就要重新从外部读回，数据路径会变复杂。写回 UB 后，激活和池化可以在片上进行原地读改写。

顶层注释也把 C 口仲裁的三个用途写清楚了：MATMUL 完成后顺序写回结果，ACT/POOL 原地读改写，STORE 按地址顺序读出，见 `AI2026/npu-project/rtl/top/npu_top.sv:722` 到 `:725`。

这就是 Unified Buffer 的第二层价值：它不只服务计算，也服务后处理和输出。

## 当前 UB 设计的边界

这个 UB 是教学尺度，不是商业 NPU 的完整片上 memory subsystem。

它没有做：

1. 多 bank 冲突处理。
2. 复杂仲裁和 QoS。
3. 双缓冲隐藏外部带宽。
4. DMA descriptor。
5. cache coherency。

但它已经足够讲清楚最核心的概念：数据一旦进入片上，就应该尽可能在 NPU 内部完成计算和后处理，减少来回搬运。

这一点比“buffer 有多大”更重要。很多初学者会把片上存储理解成容量参数，工程上更准确的理解是：片上存储定义了数据复用协议。

## 这篇的核心结论

Unified Buffer 是 NPU 数据通路的本地性边界。

AI2026 的 UB demo 给出三个值得学习的工程点：

1. 权重、激活、partial sum、输出被明确分区。
2. 三个端口对应 PE、DMA、控制/后处理三类访问者。
3. 同步读特性直接塑造了顶层 MATMUL 状态机。

下一篇我们看控制单元和后处理：有了 PE Array 和 UB，NPU 还需要一个状态机把 `LOAD -> MATMUL -> ACT -> POOL -> STORE` 串起来。
