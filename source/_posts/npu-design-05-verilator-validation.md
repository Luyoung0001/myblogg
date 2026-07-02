---
title: "NPU 设计（五）：分层验证与波形调试"
date: 2026-05-02 15:03:13
tags:
  - "NPU"
  - "Verilator"
  - "verification"
  - "RTL"
categories:
  - "Computer Architecture"
---

## 前言

RTL 设计最怕“看起来能跑”。一个 NPU 小 demo 如果只做一次端到端输出，很难定位问题。真正稳妥的方式是分层验证：

```text
单 PE
  -> PE Array
  -> Control Unit
  -> NPU Top
  -> 算子链
```

每一层都锁住一部分语义，最终整机才可信。

![NPU Verilator 分层验证](/images/mermaid-svg/npu-design-05-verilator-validation/verilator-validation.svg)

## 第一层：单 PE 验证

单 PE 要验证的不是“能不能输出一个数”，而是乘加合约是否稳定。

最小测试表可以这样写：

| 测试 | 输入 | 期望 |
|---|---|---|
| 正数乘法 | `a=2, b=3, y_in=0` | `6` |
| 累加 | `a=2, b=3, y_in=6` | `12` |
| 负数乘法 | `a=-2, b=3, y_in=0` | `-6` |
| 连续累加 | `1*1 + 2*2 + 3*3` | `14` |

这些测试分别覆盖：

1. 乘法路径；
2. 累加路径；
3. 有符号扩展；
4. valid 和寄存器延迟；
5. clear 时机。

一个 PE 如果这里都不稳定，后面阵列错了就很难定位。

## 第二层：阵列数据流验证

阵列测试重点不是再证明乘法，而是验证数据 layout 和累加维度。

一个很好的 4x4 测试是让 B 为单位矩阵：

```text
A = [ 1  2  3  4 ]       B = [ 1 0 0 0 ]
    [ 5  6  7  8 ]           [ 0 1 0 0 ]
    [ 9 10 11 12 ]           [ 0 0 1 0 ]
    [13 14 15 16 ]           [ 0 0 0 1 ]

C = A * B = A
```

如果输出不是 A，通常说明：

1. A 的行列顺序错了；
2. B 的 lane 顺序错了；
3. 输出 flatten 顺序错了；
4. 某个 valid 时序错位。

另一个测试可以让 A 的每行是常数、B 的每列是常数，方便观察每个 PE 的累加是否一致。

## 第三层：控制单元验证

控制单元不需要知道矩阵答案，它要验证 opcode 到控制信号的映射。

测试可以按命令分类：

| 命令 | 应观察到的控制行为 |
|---|---|
| `LOAD_W` | DMA 写启动，地址和长度正确 |
| `LOAD_A` | DMA 写启动，目标区域正确 |
| `MATMUL` | `pe_start` 拉高，A/B/C 地址生成 |
| `ACT` | `act_enable` 拉高，`act_type` 正确 |
| `POOL` | `pool_enable` 拉高，pool mode 正确 |
| `STORE` | DMA/输出路径启动 |

状态机还应该检查：

```text
IDLE -> FETCH -> DECODE -> EXEC -> DONE -> IDLE
```

以及每个执行状态是否等待对应 done 信号，而不是提前返回。

## 第四层：整机算子链

整机 testbench 应该验证完整链路：

```text
LOAD_W
LOAD_A
MATMUL
STORE
```

进一步可以验证：

```text
LOAD_W
LOAD_A
MATMUL
ACT(ReLU)
STORE
```

再进一步：

```text
LOAD_W
LOAD_A
MATMUL
POOL(Max)
STORE
```

每个链路都应该有可手算期望值。比如：

1. B 是单位矩阵时，MATMUL 输出应该等于 A。
2. 输入让乘法结果全为负时，ReLU 后应该全为 0。
3. 4x4 矩阵经过 2x2 max pooling 后，输出只有 4 个数。

验证整机时，应该捕获输出流，而不是只看最后一行日志。输出数量、输出顺序、输出值都要检查。

## 波形该看什么

打开波形时，不要一上来找所有信号。按数据生命周期看更清楚：

| 阶段 | 关键信号 |
|---|---|
| 命令输入 | `cmd_valid`, `cmd_ready`, `opcode` |
| 状态机 | `state`, `busy`, `done` |
| LOAD | buffer 写地址、写数据、写使能 |
| MATMUL | `pe_start`, `clear_acc`, A/B valid |
| 阵列内部 | PE accumulator、`c_valid` |
| 后处理 | ACT/POOL 读写地址、窗口值 |
| STORE | 输出 valid、输出 data、输出计数 |

波形调试的核心是跟踪同一份数据：它从输入通道进入 buffer，经过阵列，写回结果区，再被后处理和输出。不要在一堆信号里随机搜索。

## Verilator 在这里适合做什么

Verilator 很适合这种教学/研究级 RTL 验证，因为它可以：

1. 把 SystemVerilog 编译成可执行仿真模型；
2. 运行速度比传统事件仿真器快；
3. 导出 VCD/FST 波形；
4. 与 C++/测试脚本结合；
5. 适合 CI 或 smoke test。

但也要知道边界：

1. Verilator 仿真不是物理时序签核。
2. 它不能替代综合、布局布线和 STA。
3. 性能数字不等价于真实芯片性能。

所以在这个阶段，Verilator 的角色是功能验证和调试，不是最终硬件性能证明。

## 好的测试应该留下什么

一个靠谱的 NPU testbench 不应该只打印“PASS”。它至少应该留下：

1. 输入矩阵和期望输出；
2. 实际输出；
3. 错误位置；
4. 波形文件；
5. 关键状态变化；
6. 输出计数。

这样测试失败时，才知道是数据 layout、控制状态、buffer 地址、后处理还是输出通道出了问题。

## 小结

NPU 验证要分层，不要只靠整机跑通。

推荐顺序是：

```text
PE 数值语义
  -> 阵列数据流
  -> 控制命令
  -> 整机算子链
  -> 波形定位
```

到这里，NPU 设计系列形成了一个闭环：从计算公式，到 PE 阵列，到片上 buffer，到控制后处理，再到验证。下一组文章进入另一个方向：CoolDA 这类 SoC demo 如何让 CPU 真正把任务交给一个小型 NPU。
