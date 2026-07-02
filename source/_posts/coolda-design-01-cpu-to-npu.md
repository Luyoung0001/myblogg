---
title: "CoolDA 设计仿真（一）：CPU 如何把任务交给 NPU"
date: 2026-05-07 15:03:13
tags:
  - "CoolDA"
  - "SoC"
  - "NPU"
  - "runtime"
categories:
  - "Computer Architecture"
---

## 前言

单独写一个 NPU 矩阵乘法模块，只能说明硬件会算。真正有系统意义的问题是：CPU 怎么使用它？

CoolDA 这类 SoC demo 要讲的就是这条路径：

```text
用户命令
  -> 操作系统 shell
  -> runtime
  -> BSP 驱动
  -> APB 总线
  -> NPU 寄存器
  -> 4x4 INT8 矩阵乘法核
```

这篇先建立系统视角。后面再拆 APB 寄存器、BSP/runtime、仿真 shell 和测试契约。

![CPU 到 CoolDA NPU 的系统路径](/images/mermaid-svg/coolda-design-01-cpu-to-npu/soc-path.svg)

## 为什么不是直接调用硬件函数

软件里我们很容易写出这样的幻想：

```c
coolda_matmul(A, B, C);
```

但 CPU 不能真的“调用硬件函数”。硬件不是 C 函数，它通常暴露成一组寄存器。CPU 只能通过 load/store 访问某段地址，间接告诉硬件：

1. 输入数据在哪里；
2. 控制位怎么设置；
3. 什么时候开始；
4. 是否完成；
5. 结果在哪里读。

因此，一个加速器平台至少需要三层软件：

| 层 | 作用 |
|---|---|
| runtime | 把高层任务组织成 job、tile、event |
| BSP driver | 把 job 的一小步翻译成寄存器读写 |
| shell/app | 给用户或测试提供入口 |

硬件则至少需要：

| 层 | 作用 |
|---|---|
| 总线接口 | 接收 CPU 的 MMIO 访问 |
| 寄存器文件 | 保存控制、状态、输入、输出 |
| 计算核 | 真正执行乘加 |

CoolDA 的价值是把这些层都连起来，而不是只展示其中某一层。

## 最小加速路径

一个最小路径可以这样理解：

```text
CPU 运行软件
  |
  | 1. runtime 把 8x8 matmul 拆成 4x4 tile
  v
BSP driver
  |
  | 2. 写 A/B 行寄存器，写 START
  v
APB NPU
  |
  | 3. 4x4 int8 x int8 -> int32
  v
BSP driver
  |
  | 4. 轮询 DONE，读回 C
  v
runtime
  |
  | 5. 累加 partial tile，写回最终矩阵
  v
用户看到结果
```

这条链路看似长，但每层都很薄。它把复杂问题拆开了：

- CPU 负责组织；
- runtime 负责调度；
- BSP 负责敲寄存器；
- NPU 负责固定小块计算。

## 为什么硬件核只做 4x4

教学级设计里，让硬件核固定为 4x4 有几个好处：

1. 寄存器接口简单；
2. 输入输出容易手算；
3. 波形容易观察；
4. 软件 tile 调度容易讲清楚；
5. 不需要一上来引入 DMA、cache coherency、命令队列。

4x4 核的数学任务是：

```text
C[4x4] = A[4x4] * B[4x4]
```

每个输出：

```text
C[row][col] =
  A[row][0] * B[0][col] +
  A[row][1] * B[1][col] +
  A[row][2] * B[2][col] +
  A[row][3] * B[3][col]
```

输入是 INT8，输出是 INT32。这个设计非常适合说明“固定功能 accelerator kernel”。

## SoC 里的 NPU 是一个 MMIO 外设

CPU 访问 CoolDA，不需要新指令。它像访问 UART、timer 一样访问某段地址。

可以把 NPU 想成一张寄存器表：

```text
base + 0x00  CTRL
base + 0x04  STATUS
base + 0x08  INFO
base + 0x10  A0
base + 0x14  A1
base + 0x18  A2
base + 0x1C  A3
base + 0x20  B0
base + 0x24  B1
base + 0x28  B2
base + 0x2C  B3
base + 0x40  C00
...
```

CPU 往 A/B 寄存器写输入，往 CTRL 写 start，从 STATUS 读 done，再从 C 窗口读结果。

这就是最小 MMIO accelerator 的基本形态。

## runtime 为什么不能省

如果只暴露 BSP，程序员每次都要手动：

1. 打包 4x4 A；
2. 打包 4x4 B；
3. 写寄存器；
4. 等硬件；
5. 读 4x4 C；
6. 自己累加到大矩阵。

runtime 的作用是把这堆动作变成更像“任务”的接口：

```c
typedef struct {
  int m, n, k;
  int8_t  *a;
  int8_t  *b;
  int32_t *c;
  int lda, ldb, ldc;
  int flags;   // e.g. ReLU
} coolda_matmul_job_t;
```

有了 job，runtime 才能做 tile 调度、内存检查、同步/异步事件、测试和 benchmark。

## 当前边界

这个最小平台还不是商业 NPU SDK。它故意没有做很多东西：

- DMA；
- 命令队列；
- 中断完成；
- 多 stream；
- 后台硬件 async；
- accelerator-local SRAM；
- 大模型算子库。

但它已经把最关键的骨架打通：

- CPU 可以通过总线访问 NPU；
- BSP 可以驱动寄存器；
- runtime 可以把大任务拆成小 tile；
- shell 可以触发测试；
- simulator 可以观察整条链路。

这就是它作为教学平台的价值。

## 小结

CoolDA 这类 SoC demo 的核心不是“4x4 矩阵乘法多厉害”，而是展示一个加速器平台的最小闭环：

```text
CPU 软件任务
  -> runtime 调度
  -> BSP 寄存器访问
  -> APB NPU
  -> 硬件结果
```

下一篇进入硬件契约：APB 外设和 4x4 INT8 矩阵乘核到底长什么样。
