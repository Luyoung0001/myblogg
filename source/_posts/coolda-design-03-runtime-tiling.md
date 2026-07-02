---
title: "CoolDA 设计仿真（三）：BSP、runtime 与 tile 调度"
date: 2026-05-09 15:03:13
tags:
  - "CoolDA"
  - "BSP"
  - "runtime"
  - "tiling"
categories:
  - "Computer Architecture"
---

## 前言

硬件核只会做 4x4，但用户想算的矩阵可能是 8x8、16x16，甚至更大。解决方法不是让 CPU 直接操作每一个寄存器细节，而是分两层：

1. BSP：把寄存器表包装成薄函数；
2. runtime：把大任务拆成 4x4 tile，并调度硬件核反复执行。

![CoolDA BSP 与 runtime 分层](/images/mermaid-svg/coolda-design-03-runtime-tiling/bsp-runtime.svg)

## BSP：越薄越好

BSP 的职责是硬件寄存器访问。它应该非常薄。

最底层是 MMIO helper：

```c
static inline volatile uint32_t *reg_ptr(uint32_t offset) {
  return (volatile uint32_t *)(COOLDA_BASE + offset);
}

static inline void write_reg(uint32_t offset, uint32_t value) {
  *reg_ptr(offset) = value;
}

static inline uint32_t read_reg(uint32_t offset) {
  return *reg_ptr(offset);
}
```

往上是硬件动作：

```c
void load_matrix_a(const int8_t a[4][4]) {
  write_reg(REG_A0, pack_row(a[0]));
  write_reg(REG_A1, pack_row(a[1]));
  write_reg(REG_A2, pack_row(a[2]));
  write_reg(REG_A3, pack_row(a[3]));
}

void load_matrix_b(const int8_t b[4][4]) {
  write_reg(REG_B0, pack_row(b[0]));
  write_reg(REG_B1, pack_row(b[1]));
  write_reg(REG_B2, pack_row(b[2]));
  write_reg(REG_B3, pack_row(b[3]));
}

void start(int relu_enable) {
  uint32_t ctrl = CTRL_START;
  if (relu_enable) ctrl |= CTRL_RELU;
  write_reg(REG_CTRL, ctrl);
}
```

BSP 不应该知道 8x8 怎么拆，也不应该知道 event 怎么推进。它只负责把“写 A、写 B、启动、等待、读 C”这些硬件动作封装好。

## runtime：把硬件动作变成任务

runtime 关心的是 job：

```c
typedef struct {
  uint32_t flags;
  uint32_t m, n, k;
  int8_t  *a;
  int8_t  *b;
  int32_t *c;
  uint32_t lda, ldb, ldc;
} matmul_job_t;
```

这个结构比 BSP 高一层。它描述的是：

```text
C[m x n] = A[m x k] * B[k x n]
```

runtime 可以再提供类似接口：

```c
int launch_matmul(const matmul_job_t *job);
int launch_matmul_async(event_t *event, const matmul_job_t *job);
int event_poll(event_t *event);
int event_wait(event_t *event);
```

这样用户面对的是“矩阵任务”，不是寄存器细节。

## tile 调度

硬件 tile 是 4x4x4：

```text
M tile = 4
N tile = 4
K tile = 4
```

![CoolDA tile 调度示意](/images/mermaid-svg/coolda-design-03-runtime-tiling/tile-scheduler.svg)

如果要算 8x8 矩阵：

```text
C[8x8] = A[8x8] * B[8x8]
```

那么 C 会被拆成 2x2 个输出 tile：

```text
C00  C01
C10  C11
```

每个 C tile 还要沿 K 维累加两次：

```text
C00 = A00*B00 + A01*B10
C01 = A00*B01 + A01*B11
C10 = A10*B00 + A11*B10
C11 = A10*B01 + A11*B11
```

通用伪代码：

```c
for (row_base = 0; row_base < M; row_base += 4) {
  for (col_base = 0; col_base < N; col_base += 4) {
    int32_t acc[4][4] = {0};

    for (k_base = 0; k_base < K; k_base += 4) {
      int8_t  a_tile[4][4];
      int8_t  b_tile[4][4];
      int32_t partial[4][4];

      pack_a_tile(job, row_base, k_base, a_tile);
      pack_b_tile(job, k_base, col_base, b_tile);
      run_hardware_tile(a_tile, b_tile, partial);
      accumulate(acc, partial);
    }

    store_c_tile(job, row_base, col_base, acc);
  }
}
```

这就是 runtime 的核心价值：硬件只会做一个小块，runtime 负责把小块拼成完整任务。

## 边界 tile 要补零

如果矩阵大小不是 4 的倍数，tile 会越界。例如 `M=6` 时，第二个 row tile 只有 2 行有效。

pack 函数要做边界判断：

```c
if (src_row < job->m && src_k < job->k)
  a_tile[row][kk] = A[src_row][src_k];
else
  a_tile[row][kk] = 0;
```

B tile 同理。补零以后，硬件仍然只看到一个完整 4x4 tile；越界部分对结果没有贡献。

这是 tile runtime 里非常基础但非常重要的处理。

## 为什么 ReLU 放在最终 store

4x4 硬件核可以支持 ReLU bit，但在 tiled matmul 中，ReLU 不能随便提前。

假设一个输出需要两个 K tile 累加：

```text
sum = partial0 + partial1
```

如果先对 `partial0` 做 ReLU：

```text
wrong = ReLU(partial0) + partial1
```

它通常不等于：

```text
right = ReLU(partial0 + partial1)
```

所以 runtime 更合理的做法是：

```c
accumulate all K tiles first;
if (relu && value < 0) value = 0;
store value to C;
```

硬件负责 partial tile 的乘加，runtime 负责跨 K tile 的累加和最终后处理。

## async event 是软件调度语义

一个轻量 runtime 可以提供 event：

```c
typedef enum {
  EVENT_IDLE,
  EVENT_RUNNING,
  EVENT_DONE,
  EVENT_ERROR
} event_state_t;

typedef struct {
  matmul_job_t job;
  event_state_t state;
  uint32_t total_steps;
  uint32_t completed_steps;
  uint32_t row_base, col_base, k_base;
  int32_t accum_tile[4][4];
} event_t;
```

`event_poll()` 每次推进一个 tile step：

```c
int event_poll(event_t *e) {
  if (e->completed_steps >= e->total_steps) {
    e->state = EVENT_DONE;
    return 1;
  }

  pack_a_tile(...);
  pack_b_tile(...);
  run_hardware_tile(...);
  accumulate(...);
  advance_indices(e);

  return 0;
}
```

注意：这是一种软件 async/event 语义，不等于硬件后台队列。每次 poll 都还是由 CPU 推进。真正的硬件 async 需要命令队列、中断、DMA 和后台执行。

## 小结

CoolDA 的 BSP/runtime 分层可以记成：

```text
BSP = 寄存器动作
runtime = 任务、tile、event、内存语义
```

硬件小核只做 4x4，runtime 通过 tile 调度把它扩展成更大的矩阵乘法。这是很多加速器平台最核心的思想：硬件提供 primitive，软件负责 orchestration。

下一篇看仿真环境：xOS shell 和 Verilator 如何让这条路径可交互、可脚本化、可观察。
