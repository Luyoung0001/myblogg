---
title: "CoolDA 设计仿真笔记（五）：烟测不是跑命令，而是验证契约"
date: 2026-05-11 15:03:13
tags:
  - "AI2026"
  - "CoolDA"
  - "testing"
  - "Verilator"
categories:
  - "Computer Architecture"
---

## 前言

工程里的 smoke test 很容易被写成“自动跑几条命令”。这种说法太浅了。一个好的 smoke test 应该验证跨层契约：固件能构建，SoC 能仿真，shell 能收命令，runtime 能调度，BSP 能写寄存器，NPU 能给出正确结果，UART 能把判定输出给 host。

CoolDA 的 smoke 目标正好体现了这个思路。它不是为了向读者展示终端输出，而是把 CPU、总线、runtime、NPU 和 simulator 的主链路变成一个自动判定。

![CoolDA smoke 测试契约](/images/mermaid-svg/ai2026-coolda-05-test-contract/test-contract.svg)

## 测试契约从构建开始

`coolda-soc/Makefile` 里 `coolda-smoke` 目标先构建 xOS firmware，再构建 SoC Verilator simulator，然后用脚本化 shell 命令进入系统，见 `AI2026/coolda-soc/Makefile:182` 到 `:197`。

这一步验证的不只是软件，也包括构建边界：

1. xOS 精简固件能被当前工具链编出来。
2. SoC RTL 和 C++ simulator 能被 Verilator/C++ 编译链接受。
3. ELF 能被仿真器加载。
4. shell 能在 SoC 内部启动。

如果这些都不成立，后面的 CoolDA runtime 根本没有展示舞台。

## Makefile 里的 grep 不是“脆弱输出检查”

很多人看到 smoke test 里 grep 输出字符串，会本能觉得“不够高级”。但这里要看它 grep 的是什么。

`coolda-smoke` 检查了：

- `Result check: PASS`
- `Vecadd result check: PASS`
- `Coolda runtime check: PASS`
- `Kernel: vecadd`
- `Coolda runtime API`
- `[SIM] Script commands completed.`

这些检查写在 `AI2026/coolda-soc/Makefile:204` 到 `:211`。

它们不是随机日志，而是跨层语义哨兵：

1. `Result check: PASS` 表示 matmul 结果与 CPU reference 对齐。
2. `Vecadd result check: PASS` 表示 runtime 的 device memory/vecadd API 路径还活着。
3. `Coolda runtime check: PASS` 表示 runtime 自检通过。
4. `Kernel: vecadd` 表示 benchmark 入口被执行。
5. `Coolda runtime API` 表示 API 展示命令能进入 runtime。
6. `[SIM] Script commands completed.` 表示仿真器脚本注入和 shell prompt 同步机制正常结束。

所以 grep 在这里是最终契约的观察点。更复杂的项目当然可以用结构化日志或 test protocol，但这套 demo 当前用文本哨兵很合理。

## matmul 功能检查验证什么

`coolda_run_test_internal()` 是 matmul 功能验证核心。它准备 demo context，执行同步或异步 job，可选打印矩阵，然后读取 hardware/runtime 结果并与 CPU reference 比较，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:925` 到 `:963`。

CPU reference 在 `cpu_matmul_ref()`，按 `row/col/k` 三层循环计算，并根据 job flags 做 ReLU，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:181` 到 `:199`。

这层测试覆盖的是完整 matmul 软件路径：

1. device heap 准备。
2. host 数据拷贝到 device model。
3. runtime 生成 job。
4. tiled 调度调用 BSP/NPU。
5. 结果拷回 host。
6. 与 CPU reference 对比。

也就是说，它不只是验证 4x4 RTL 小核，而是验证 runtime 对 4x4 小核的使用方式。

## event demo 验证异步语义

`coolda_run_event_demo()` 会初始化一个 8x8 job，调用 `coolda_launch_matmul_async()`，然后循环 `coolda_event_poll()` 并打印每一步 state 和下一组 row/col/k 索引，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:1001` 到 `:1092`。

最后它把 device 结果拷回 host，与 CPU reference 比较，并打印 final state 和 result check，见同文件 `:1094` 到 `:1116`。

这个测试的重点不是性能，而是事件状态机：

1. launch 后进入 RUNNING。
2. 每次 poll 推进一个 tile step。
3. tile 全部完成后进入 DONE。
4. 结果仍然与 reference 一致。

这正好对应当前 runtime 的 async 边界：它是软件 event/poll 语义，不是真后台硬件队列。

## vecadd 检查的真实意义

当前 CoolDA 的 vecadd 并不是 RTL kernel。`coolda_launch_vecadd()` 在 runtime 里调用 CPU reference，并检查 device memory 范围，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:904` 到 `:918`。

那为什么 smoke 还检查 `Vecadd result check: PASS`？

因为它验证的是 runtime API 的另一个维度：device heap、memcpy、job 参数、结果拷回和 benchmark 框架。`coolda_run_vec_test()` 会 launch vecadd、拷回 host、逐项比较 reference，并打印 pass/fail，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:1119` 到 `:1152`。

这提醒我们：测试名不一定只对应硬件核，也可能对应 runtime 层契约。文章里必须把这个边界说清楚，避免读者误以为 RTL 已经实现了 vecadd accelerator。

## runtime 自检应该看什么

`cooldacheck` 命令最终调用 `coolda_run_runtime_check()`，命令 handler 在 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_cmd.c:221` 到 `:229`。

虽然这里不展开所有自检细节，但从 runtime 设计可以看出它应该覆盖几类风险：

1. device heap 分配是否越界。
2. memcpy to/from device 是否检查地址范围。
3. job 参数是否合法。
4. event 状态是否能从 RUNNING 到 DONE。
5. 不同 API 是否共享同一套初始化和内存模型。

`coolda_memcpy_to_device()` 和 `coolda_memcpy_to_host()` 都会检查 device range，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:760` 到 `:793`。这类防线对 runtime 自检很关键。

## simulator 结束条件也是测试契约

自动 smoke 不只要执行命令，还要知道什么时候结束。

`sim_script_tick()` 会观察 UART 输出里的 shell prompt，注入下一条排队命令；当所有命令完成且启用退出条件时，打印 `[SIM] Script commands completed.` 并把 `running` 置 false，见 `AI2026/coolda-soc/simulator/src/uart.cpp:65` 到 `:92`。

这就是仿真器和 shell 之间的同步契约。没有它，host 侧脚本可能在 shell 还没准备好时塞命令，也可能不知道最后一条命令是否真的跑完。

把 `[SIM] Script commands completed.` 放进 smoke 检查，验证的是“命令注入机制完整结束”，不是某个 runtime 函数本身。

## benchmark 结果要克制解读

CoolDA 有 benchmark 命令，`coolda_run_bench()` 会打印 launch mode、problem size、work per launch、total work、CPU/Coolda 统计和 speedup 信息，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_runtime.c:965` 到 `:999`。

但作为当前 demo，benchmark 更适合做回归观察，而不是严肃性能宣称。

原因很简单：

1. 当前硬件是寄存器喂数据的 4x4 小核。
2. 没有 DMA 和后台队列。
3. Verilator 仿真时间不等价于真实硅上周期成本。
4. README 也说明裸机 timer 分辨率会影响 CPU throughput 口径，见 `AI2026/coolda-soc/README.md:131` 到 `:139`。

所以博客里可以讲 benchmark 结构，但不应该把某个数字写成硬件性能结论。更准确的说法是：benchmark 是接口和回归工具，不是最终性能报告。

## 这篇的核心结论

CoolDA 的 smoke 测试验证的是一组跨层契约：

1. firmware、RTL、simulator 可以一起构建。
2. shell 脚本注入能按 prompt 同步。
3. matmul runtime 结果能对齐 CPU reference。
4. vecadd/runtime API 路径能通过自检。
5. 仿真器能在所有脚本命令完成后干净退出。

这组契约让 CoolDA 从“能演示”变成“可回归”。到这里，CoolDA 系列也形成了闭环：系统路径、APB NPU、BSP/runtime、xOS 仿真、测试契约。后续如果继续扩展 DMA、命令队列或中断完成，就可以沿着这五层逐步加复杂度，而不是推倒重来。
