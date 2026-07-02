---
title: "CoolDA 设计仿真笔记（三）：BSP、runtime 与 tile 调度"
date: 2026-05-09 15:03:13
tags:
  - "AI2026"
  - "CoolDA"
  - "runtime"
  - "BSP"
categories:
  - "Computer Architecture"
---

## 前言

CoolDA 的硬件核只会做 4x4 INT8 matmul。要让它服务一个 8x8，甚至更大一点的矩阵乘法，不能指望硬件突然变聪明；必须让软件把大任务拆成小 tile，再一块块喂给硬件。

这就是 BSP 和 runtime 的分工：BSP 负责薄薄一层寄存器读写，runtime 负责内存、job、tile、event 和结果检查。

![CoolDA BSP 与 runtime 分层](/images/mermaid-svg/ai2026-coolda-03-bsp-runtime/bsp-runtime.svg)

## BSP 应该薄

BSP 的目标不是做复杂调度，而是把硬件寄存器包装成可调用的小函数。

`bsp/src/coolda.c` 里最底层是两个 volatile MMIO helper：`coolda_write()` 和 `coolda_read()`，见 `AI2026/coolda-soc/bsp/src/coolda.c:3` 到 `:13`。

往上才是具体动作：

1. `bsp_coolda_reset()` 写 `CLEAR`，见 `AI2026/coolda-soc/bsp/src/coolda.c:22` 到 `:24`。
2. `bsp_coolda_load_matrix_a()` 写 A0 到 A3，见同文件 `:34` 到 `:39`。
3. `bsp_coolda_load_matrix_b()` 写 B0 到 B3，见同文件 `:41` 到 `:46`。
4. `bsp_coolda_start()` 写 start/relu 控制位，见同文件 `:48` 到 `:54`。
5. `bsp_coolda_wait_done()` 轮询 status，见同文件 `:56` 到 `:65`。
6. `bsp_coolda_read_matrix_c()` 读回 C，见同文件 `:67` 到 `:77`。

这层越薄越好。它不应该知道 8x8 怎么拆，也不应该知道 event 怎么推进。BSP 是硬件说明书的 C 语言版本。

## runtime 把硬件能力提升成 job

runtime 的接口在 `coolda_runtime.h`。矩阵乘 job 包含 flags、m/n/k、A/B/C 指针和 leading dimension，见 `AI2026/coolda-soc/software/xos_pro_max/include/coolda_runtime.h:18` 到 `:29`。

这比 BSP 高了一个抽象层次。BSP 只会“写四行 A、写四行 B、启动、读 C”；runtime 说的是“我要算一个 m x n 的 C = A x B”。

runtime 还定义了模拟 device heap，容量是 64KB，见 `AI2026/coolda-soc/software/xos_pro_max/include/coolda_runtime.h:11`。具体 heap 数组和分配记录在 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:21` 到 `:23`，`coolda_malloc()` 的实现从同文件 `:708` 到 `:742`。

这说明 CoolDA 在软件层模拟了一个很小的 device memory 模型。它不是真 DMA，也不是真实显存，但它能让 API 呈现出类似 accelerator runtime 的形状：申请设备内存、拷贝到设备、launch、拷贝回主机。

## tile 调度：把大矩阵拆给 4x4 小核

runtime 中的 tile 大小写在头文件里：

```text
COOLDA_TILE_M = 4
COOLDA_TILE_N = 4
COOLDA_TILE_K = 4
```

定义位置是 `AI2026/coolda-soc/software/xos_pro_max/include/coolda_runtime.h:7` 到 `:9`。

![CoolDA tile 调度示意](/images/mermaid-svg/ai2026-coolda-03-bsp-runtime/tile-scheduler.svg)

调度时，runtime 先把 A 和 B 的对应 tile pack 出来。`pack_a_tile()` 负责取 A 的 `row_base, k_base` 小块，越界则补 0，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:250` 到 `:268`。`pack_b_tile()` 负责取 B 的 `k_base, col_base` 小块，同样越界补 0，见同文件 `:270` 到 `:288`。

硬件只算一个 K tile 的 partial result。runtime 用 `accumulate_c_tile()` 把 partial tile 累加到当前 C tile 上，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:301` 到 `:311`。等 K 维 tile 都完成后，`store_c_tile()` 再写回最终 C，并在需要时做 ReLU，见同文件 `:313` 到 `:333`。

这就是 tiled matmul 的基本结构：

```text
for row_tile
  for col_tile
    accum = 0
    for k_tile
      partial = NPU(A_tile, B_tile)
      accum += partial
    store accum
```

硬件小，软件就要承担调度复杂度。

## runtime 如何调用硬件 tile

`coolda_run_tile()` 是 runtime 和 BSP 的交界点。它依次执行 reset、load A、load B、start、wait done、read C，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:335` 到 `:347`。

这里有一个细节值得写清楚：`coolda_run_tile()` 调用 `bsp_coolda_start(0)`，也就是当前 tiled matmul 的硬件 tile 阶段不启用 RTL ReLU，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:338` 到 `:342`。

原因是跨 K tile 累加时不能提前 ReLU。假设某个 partial sum 暂时为负，后续 K tile 可能把它加成正数；如果在硬件每个 partial tile 后就 ReLU，会破坏完整矩阵乘法语义。

所以当前 runtime 的 job 级 ReLU 放在 `store_c_tile()` 最终写回阶段处理，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:325` 到 `:329`。

这不是硬件无能，而是分层选择：硬件 tile 负责乘加，runtime 负责跨 tile 累加和最终后处理。

## event 模型：async 不是硬件后台队列

CoolDA runtime 有 `coolda_launch_matmul_async()`、`coolda_event_poll()` 和 `coolda_event_wait()`。API 定义在 `AI2026/coolda-soc/software/xos_pro_max/include/coolda_runtime.h:71` 到 `:74`。

但这里的 async 要准确理解。它是软件事件模型，不是硬件命令队列，也不是真正后台 DMA/NPU 并行。

`coolda_launch_matmul_async()` 会初始化 event，计算 tile_rows、tile_cols、tile_ks，设置 `total_steps`，清空累加 tile，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:804` 到 `:831`。

真正推进一次工作发生在 `coolda_event_poll()`：pack A/B tile，调用 `coolda_run_tile()`，累加 partial result，推进 row/col/k 索引，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:833` 到 `:876`。

这意味着每次 poll 推进一步。它非常适合教学 event 语义，但不能说成硬件后台异步队列。项目 README 也明确把“真后台硬件 async”列为当前没有的能力，见 `AI2026/coolda-soc/README.md:153` 到 `:160`。

## vecadd 为什么要特别说明

runtime 里还有 vecadd API。`coolda_vecadd_job_t` 定义在 `AI2026/coolda-soc/software/xos_pro_max/include/coolda_runtime.h:31` 到 `:36`，public API 里也有 `coolda_launch_vecadd()`，见同文件 `:75`。

但当前 `coolda_launch_vecadd()` 调的是 CPU reference，并检查 device memory 范围，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:904` 到 `:918`。

这件事也要诚实写出来。vecadd 在当前项目里主要服务 runtime API 和测试契约，不代表 RTL 里已经有 vecadd kernel。

## 这篇的核心结论

CoolDA 的 BSP/runtime 分层可以这样理解：

1. BSP 是寄存器协议的薄封装。
2. runtime 是 job、device heap、tile 调度和 event 语义。
3. 4x4 硬件核通过 tiled matmul 被提升成更大任务。
4. 当前 async 是软件 poll/event，不是硬件后台队列。

下一篇我们看 xOS shell 和 Verilator simulator：这些软件/仿真层如何把 CoolDA 变成一个可交互、可脚本化观察的平台。
