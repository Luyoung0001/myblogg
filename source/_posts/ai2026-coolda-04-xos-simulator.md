---
title: "CoolDA 设计仿真笔记（四）：xOS Shell 与 Verilator 交互式仿真"
date: 2026-05-10 15:03:13
tags:
  - "AI2026"
  - "CoolDA"
  - "Verilator"
  - "xOS"
categories:
  - "Computer Architecture"
---

## 前言

加速器 demo 如果只能在 testbench 里改输入，看起来就还是硬件实验。CoolDA 更有意思的地方，是它把一个精简 xOS shell 放进 SoC 仿真里：host 终端输入命令，xOS 解析命令，runtime 调 BSP，BSP 写 APB NPU，UART 再把结果打印回 host。

这让 Verilator 不只是“跑 RTL 的工具”，而成为一个能观察系统行为的交互式环境。

![xOS Shell 与 Verilator 仿真链路](/images/mermaid-svg/ai2026-coolda-04-xos-simulator/xos-simulator.svg)

## xOS shell 是应用层入口

`shell_coolda.c` 文件开头就说明它是 CoolDA simulation demo 的 lean xOS shell，见 `AI2026/coolda-soc/software/xos_pro_max/src/shell_coolda.c:1` 到 `:3`。

命令表在同文件 `:21` 到 `:36`，除了 `help/echo/clear/info/heap/ps` 这些基础命令，还包括：

- `coolda`
- `cooldatest`
- `cooldabench`
- `cooldavectest`
- `cooldavecbench`
- `cooldaapi`
- `cooldaevent`
- `cooldacheck`

这张表的意义是：CoolDA 的测试、benchmark、API 展示和 event 展示都不是 host 脚本直接调用 C 函数，而是通过 SoC 内部运行的软件命令进入 runtime。

这比“在 host 上跑一个 C++ 模型”更接近真实嵌入式系统。

## 命令解析保持很小

shell 的命令解析很朴素。`parse_command()` 用空格切分 argv，见 `AI2026/coolda-soc/software/xos_pro_max/src/shell_coolda.c:39` 到 `:61`。`execute_command()` 在命令表里查找名字并调用 handler，见同文件 `:63` 到 `:87`。

CoolDA 相关命令的参数解析放在 `coolda_cmd.c`。`parse_mode_args()` 处理 `relu`、`verbose/brief`、`sync/async` 和迭代次数，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_cmd.c:54` 到 `:110`。

具体命令 handler 只是把参数翻译成 runtime 调用：

1. `cmd_coolda_test()` 调 `coolda_run_test()`，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_cmd.c:133` 到 `:151`。
2. `cmd_coolda_bench()` 调 `coolda_run_bench()`，见同文件 `:153` 到 `:171`。
3. `cmd_coolda_check()` 调 `coolda_run_runtime_check()`，见同文件 `:221` 到 `:229`。
4. `cmd_coolda_event()` 调 `coolda_run_event_demo()`，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_event_cmd.c:6` 到 `:20`。

这种结构很干净：shell 不懂矩阵乘法，shell 只负责把用户意图变成 runtime API。

## runtime API 展示也是系统文档

`coolda_api_cmd.c` 里的 `cmd_coolda_api()` 专门打印 runtime API 形状。它展示硬件 tile 尺寸、device heap、sync/async 语义、event 状态和主要 API，见 `AI2026/coolda-soc/software/xos_pro_max/src/coolda_api_cmd.c:5` 到 `:45`。

这类命令不只是为了“好看”。它让读者在 shell 里看到当前抽象层：

1. 硬件 tile 是 `4x4x4`。
2. 设备模型是 runtime-managed simulated device heap。
3. async 是 event-driven software scheduler。
4. public API 包括 init、malloc/free、memcpy、launch、event poll/wait。

换句话说，shell 也承担了活文档的角色。

## Verilator simulator 的主循环

仿真器主入口在 `simulator/main.cpp`。文件头部列出功能：模拟 SRAM、模拟 UART、生成 FST 波形、支持 ELF 程序加载，见 `AI2026/coolda-soc/simulator/main.cpp:1` 到 `:9`。

main 流程是：

1. 解析参数。
2. 初始化 SRAM。
3. 加载 ELF 或 binary。
4. 创建 Verilated context 和顶层对象。
5. 初始化 UART、PS2、trace。
6. reset。
7. 进入 `exec()` 主循环。

这些动作在 `AI2026/coolda-soc/simulator/main.cpp:23` 到 `:73`。

`cpu.cpp` 里的 `parse_arg()` 支持 `--elf`、`--bin`、`--cmd` 和 `--exit-after-cmds`，见 `AI2026/coolda-soc/simulator/src/cpu.cpp:103` 到 `:135`。这让仿真器既能交互运行，也能把 shell 命令排队注入，形成自动验证。

## host 输入为什么走 PS2 bypass

项目保留了一个 host 输入到 PS2 scancode 的 bypass。`ps2.cpp` 的注释说明 host keyboard input 会被转换成 PS2 Set-2 scancode，然后通过 RTL simulation bypass 接口注入，见 `AI2026/coolda-soc/simulator/src/ps2.cpp:1` 到 `:6`。

`ps2_queue_ascii_text()` 可以把一串 ASCII 文本转换成 scancode 序列，见 `AI2026/coolda-soc/simulator/src/ps2.cpp:92` 到 `:103`。`ps2_drive_host_bypass()` 每个仿真 tick 处理输入、更新队列，并驱动 `sim_ps2_scancode` 和 valid 信号，见同文件 `:130` 到 `:140`。

README 也解释了这个取舍：当前 shell 交互仍通过 `sim_ps2_scancode` 把 host 输入送进 SoC，这不是项目主角，而是仿真输入桥，见 `AI2026/coolda-soc/README.md:141` 到 `:151`。

这就是工程上的务实选择：UART 输出已经能映射到 host stdout；输入侧用 PS2 bypass 保持 shell 可用，先服务 CoolDA 主线。

## UART 输出如何变成 host 可见文本

`uart.cpp` 里有两件事。

第一，UART 输出会被转到 host stdout。`uart_read_handling()` 监控 APB UART 写，当地址是 UART THR 时，把可打印字符输出到 host，并通知脚本系统，见 `AI2026/coolda-soc/simulator/src/uart.cpp:143` 到 `:162`。

第二，脚本命令注入依赖 shell prompt。`sim_script_tick()` 维护一个 prompt window，看到 `xos> ` 后，如果有排队命令，就通过 `ps2_queue_ascii_text()` 注入下一条命令，见 `AI2026/coolda-soc/simulator/src/uart.cpp:65` 到 `:92`。

这很巧妙：仿真脚本没有硬塞一段输入流，而是等 shell 真正回到 prompt，再注入下一条命令。这样 smoke 测试就不容易因为命令执行时间不同而乱序。

## xOS firmware 为什么要精简

`software/xos_pro_max/Makefile` 写明这个 firmware 是 lean Coolda simulation firmware，只 boot CPU、初始化最小 xOS 服务，并通过 UART shell 暴露 CoolDA runtime 命令，见 `AI2026/coolda-soc/software/xos_pro_max/Makefile:1` 到 `:6`。

它的源码白名单也很清楚：核心 xOS 源文件、CoolDA shell/runtime/命令文件，再 include BSP 和 mylibc，见同文件 `:24` 到 `:53`。

这就是一个很好的工程取舍。为了讲 CoolDA，不把 JIT、HDMI、游戏、PPT 等历史素材混进默认路径，避免读者在不相关模块里迷路。

## 这篇的核心结论

CoolDA 的仿真环境不是简单“跑一下 RTL”，而是一条完整交互链：

```text
host terminal
  -> Verilator input bridge
  -> xOS shell
  -> CoolDA runtime
  -> BSP
  -> APB NPU
  -> UART stdout
```

这条链让 CoolDA 同时具备教学性和工程可观察性。下一篇我们看测试：smoke 目标到底在验证什么，为什么它不是“跑命令”，而是跨层契约检查。
