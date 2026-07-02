---
title: "CoolDA 设计仿真笔记（一）：CPU 如何把任务交给 NPU"
date: 2026-05-07 15:03:13
tags:
  - "AI2026"
  - "CoolDA"
  - "SoC"
  - "NPU"
categories:
  - "Computer Architecture"
---

## 前言

如果 NPU 只是一个单独 testbench 里的矩阵乘法模块，它能讲清楚“硬件怎么算”，但讲不清楚“系统怎么用”。CoolDA 这个 demo 的价值恰好在后一件事：它把一个小型 NPU 挂进 LoongArch32R SoC，让 CPU、runtime、BSP、APB 外设和 Verilator simulator 形成一条完整路径。

项目 README 对 CoolDA 的定位很直接：这是一个 “CPU 调 NPU 加速” 的轻量 SoC 仿真 demo，保留 LoongArch32R CPU、AXI/APB、UART、Coolda NPU、xOS 精简 shell 和 Verilator 仿真器，见 `AI2026/coolda-soc/README.md:1` 到 `:3`。

这一篇先不急着看寄存器，而是先建立系统视角：CPU 怎样把一个高层矩阵任务交给硬件。

![CPU 到 CoolDA NPU 的系统路径](/images/mermaid-svg/ai2026-coolda-01-soc-path/soc-path.svg)

## 一个最小加速平台需要几层

从软件视角看，“调用 NPU”好像只是一句函数调用。从硬件视角看，“启动 NPU”好像只是写一个 start bit。真正的系统路径夹在两者之间。

CoolDA README 把路径写成：

```text
host terminal
  -> xOS lean shell
  -> Coolda runtime
  -> BSP MMIO driver
  -> APB peripheral
  -> coolda_npu
```

这条链路在 `AI2026/coolda-soc/README.md:9` 到 `:30`。其中最关键的一句话是：CPU 组织任务，runtime 把 `8x8` 矩阵乘法拆成 `4x4` tile，BSP 写寄存器，RTL 里的 `coolda_npu` 做 `int8 x int8 -> int32` 乘加，见 `AI2026/coolda-soc/README.md:30`。

这个分层非常像真实加速平台的缩小版：

1. App/shell 负责发起任务。
2. runtime 负责描述 job、管理内存、做 tile 调度。
3. BSP 负责把抽象操作翻译成 MMIO 寄存器访问。
4. 总线和外设负责把 CPU load/store 送到硬件。
5. NPU RTL 负责固定功能计算。

不要小看这种“缩小版”。现代 GPU/NPU SDK 也离不开这些层，只是每层都更复杂。

## 为什么 CoolDA 不是单独 NPU testbench

项目架构文档把核心命题压缩成一句话：在一个可仿真的 LoongArch32R SoC 里，把 `CPU -> runtime -> MMIO NPU` 这条最小加速路径完整跑通，并用可解释方式呈现现代加速平台关键层次，见 `AI2026/coolda-soc/docs/COOLDA_ARCHITECTURE.md:41` 到 `:45`。

这句话里有两个很重要的边界。

第一，CoolDA 是 SoC demo。它不是一个独立 RTL kernel，也不是只有 C 函数的 runtime mock。架构文档在 `AI2026/coolda-soc/docs/COOLDA_ARCHITECTURE.md:78` 到 `:89` 强调了 CPU、AXI/APB、xOS、shell、MMIO 外设和仿真器这些系统背景。

第二，CoolDA 是最小路径。它暂时不追 DMA、命令队列、中断、多 stream、大模型、多核 PE 阵列，见 `AI2026/coolda-soc/docs/COOLDA_ARCHITECTURE.md:91` 到 `:104`。这些功能都重要，但对第一阶段的学习目标来说不是主菜。

这让 CoolDA 成为一个很好的工程教学对象：小到能读，大到有系统层次。

## 硬件挂在 APB 上

CoolDA NPU 在 SoC 里不是特殊魔法，而是一个 APB 外设。

`apb_dev_top_no_nand.v` 里有一个 APB mux。UART 接在 `apb0`，CoolDA 接在 `apb1`，相关连接在 `AI2026/coolda-soc/IP/APB_DEV/apb_dev_top_no_nand.v:293` 到 `:337`。真正实例化 `coolda_npu` 的位置在同文件 `:367` 到 `:381`。

这件事翻译成系统语言就是：CPU 通过普通 MMIO load/store 访问 CoolDA。它没有专门指令，也没有特殊协处理器端口。对 CPU 来说，CoolDA 首先是一片地址范围。

BSP 里定义的基地址是 `0x1FE04000`，见 `AI2026/coolda-soc/bsp/include/coolda.h:6`。寄存器偏移从 `CTRL`、`STATUS`、`INFO` 到 A/B 输入和 C 输出窗口，见同文件 `:8` 到 `:19`。

这种设计的优点很朴素：

1. CPU 端容易驱动。
2. RTL 接口容易验证。
3. 软件和硬件的边界可以用寄存器文档讲清楚。

缺点也同样明显：没有 DMA 时，数据搬运成本都压在 CPU 和 BSP 上。作为教学 demo，这个代价是值得的，因为它让路径透明。

## CPU 负责组织，NPU 只负责固定小核

CoolDA NPU 的 RTL 是一个固定 `4x4` INT8 matmul 小核。`coolda_npu.v` 中寄存器定义在 `AI2026/coolda-soc/IP/APB_DEV/coolda_npu.v:15` 到 `:28`，A/B 输入行寄存器和 C 输出窗口都非常明确。

硬件计算主循环在 `AI2026/coolda-soc/IP/APB_DEV/coolda_npu.v:118` 到 `:216`。启动后，硬件把 `busy` 拉高、清掉 `done`，让 `k_idx` 从 0 到 3，每个 `k_idx` 对 4x4 的所有 row/col 做一次乘加，关键乘加在同文件 `:161` 到 `:190`。

注意它的能力边界：

1. 它只会算 4x4。
2. 输入通过寄存器行喂进去。
3. 输出通过 C 寄存器窗口读出来。
4. 它不会自己去主存搬大矩阵。

所以大任务必须由软件拆小。这个拆分不是临时补丁，而是 runtime 的职责。

## runtime 为什么存在

如果只有 BSP，软件每次都要手动准备 A/B 小矩阵、写寄存器、等 done、读 C、累加 partial sum。这样能工作，但无法成为一个像样的编程接口。

CoolDA runtime 把这些动作包装成更高层的 job。`coolda_runtime.h` 里定义了 `coolda_matmul_job_t`，包含 flags、m/n/k、A/B/C 指针和 leading dimension，见 `AI2026/coolda-soc/software/xos_pro_max/include/coolda_runtime.h:18` 到 `:29`。

它还定义了 event 状态：`IDLE`、`RUNNING`、`DONE`、`ERROR`，见同文件 `:44` 到 `:49`。public API 包括 init、malloc、memcpy、同步/异步 matmul、event poll/wait、vecadd、synchronize、test/bench 等，见 `AI2026/coolda-soc/software/xos_pro_max/include/coolda_runtime.h:63` 到 `:85`。

这就是 runtime 的价值：让“很多次寄存器读写”变成“一个有语义的加速任务”。

## 当前边界要写清楚

CoolDA 很适合讲平台，但不能把它夸成商业 CUDA。

README 明确列了当前还没有的能力：DMA、命令队列、中断完成、真实后台硬件 async、accelerator-local SRAM/UB、大模型算子库、多 stream/多 kernel 调度，见 `AI2026/coolda-soc/README.md:153` 到 `:164`。

它已经有的，是 SoC 级集成、MMIO NPU、BSP 驱动、runtime job/event 模型、tiled matmul、功能测试/benchmark 分离和 host 交互式仿真，见 `AI2026/coolda-soc/README.md:165` 到 `:173`。

这个边界很重要。技术文章最好不要把 demo 写成已经拥有完整加速平台所有能力；更准确的表达是：CoolDA 把最小闭环做完整了，并给后续 DMA、队列、中断和多 kernel 调度留下了清晰扩展点。

## 这篇的核心结论

CoolDA 的第一层价值不是 4x4 矩阵乘法本身，而是系统路径：

```text
CPU / xOS
  -> runtime
  -> BSP MMIO
  -> APB peripheral
  -> 4x4 NPU RTL
```

这条路径让读者能看见一个加速器平台的骨架。下一篇我们深入最底层的硬件契约：APB 外设寄存器如何喂给一个 4x4 INT8 矩阵乘核。
