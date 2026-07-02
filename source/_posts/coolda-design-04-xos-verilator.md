---
title: "CoolDA 设计仿真（四）：xOS Shell 与 Verilator 交互"
date: 2026-05-10 15:03:13
tags:
  - "CoolDA"
  - "Verilator"
  - "xOS"
  - "simulation"
categories:
  - "Computer Architecture"
---

## 前言

硬件 demo 如果只能在 testbench 里改输入，系统感会很弱。CoolDA 这类 SoC 仿真更进一步：让 CPU 启动一个精简 shell，用户或脚本通过 shell 触发 runtime，runtime 再驱动 NPU。

这样 Verilator 就不只是“跑 RTL”，而是变成一个可交互的系统实验环境。

![xOS Shell 与 Verilator 仿真链路](/images/mermaid-svg/coolda-design-04-xos-verilator/xos-simulator.svg)

## 交互链路

整个交互链路可以写成：

```text
host terminal
  -> simulator input bridge
  -> xOS shell
  -> command parser
  -> CoolDA runtime
  -> BSP
  -> APB NPU
  -> UART output
  -> host stdout
```

这条链路的意义是：用户看到的是 shell 命令，实际验证的是 SoC 内部的软件栈和硬件外设。

## shell 命令表

一个精简 shell 可以维护一张命令表：

```c
typedef int (*cmd_handler_t)(int argc, char **argv);

typedef struct {
  const char    *name;
  const char    *help;
  cmd_handler_t  handler;
} shell_cmd_t;

static const shell_cmd_t commands[] = {
  {"help",          "show commands",             cmd_help},
  {"info",          "show system info",          cmd_info},
  {"coolda",        "run combined demo",         cmd_coolda},
  {"cooldatest",    "run matmul test",           cmd_coolda_test},
  {"cooldabench",   "run matmul benchmark",      cmd_coolda_bench},
  {"cooldavectest", "run vecadd API test",       cmd_coolda_vec_test},
  {"cooldacheck",   "run runtime self-check",    cmd_coolda_check},
  {"cooldaapi",     "show runtime API shape",    cmd_coolda_api},
  {"cooldaevent",   "show event progression",    cmd_coolda_event},
  {NULL, NULL, NULL}
};
```

shell 不懂矩阵乘法。它只负责把字符串变成 handler 调用。

## 命令解析

命令解析可以非常小：

```c
int parse_command(char *line, char *argv[], int max_args) {
  int argc = 0;
  char *p = line;

  while (*p && argc < max_args) {
    while (*p == ' ') p++;
    if (*p == '\0') break;

    argv[argc++] = p;
    while (*p && *p != ' ') p++;
    if (*p) *p++ = '\0';
  }

  return argc;
}
```

执行阶段就是查表：

```c
for (cmd = commands; cmd->name; cmd++) {
  if (strcmp(argv[0], cmd->name) == 0) {
    return cmd->handler(argc, argv);
  }
}
```

这类 shell 的重点不是功能华丽，而是提供一个稳定入口，让 runtime 能在真实 CPU 软件环境中被触发。

## simulator 主循环

Verilator simulator 的主循环通常做这些事：

```c
init_sram();
load_program_image();
create_verilated_context();
create_soc_top();
init_uart();
init_input_bridge();
reset_soc();

while (running) {
  drive_input_bridge();
  toggle_clock_rising();
  handle_soc_memory_access();
  handle_uart_output();
  handle_script_injection();
  dump_wave_if_enabled();
  toggle_clock_falling();
}
```

这个循环把硬件世界和 host 世界接起来：

- SoC 里的 UART 写，会变成 host stdout；
- host 输入，会被转换成 SoC 能理解的键盘/串口输入；
- 仿真器可以在每个周期导出波形；
- 脚本可以等待 shell prompt 再注入下一条命令。

## 为什么输入桥可以先简单一点

真实 UART RX、PS2 键盘协议都可以很复杂。教学 demo 里，输入桥可以先简化：

```text
host 字符
  -> ASCII
  -> scancode 或 bypass 信号
  -> shell input
```

伪代码：

```c
void queue_text_as_input(const char *text) {
  for (const char *p = text; *p; p++) {
    uint8_t code = ascii_to_scancode(*p);
    if (code) queue_push(code);
  }
}

void drive_input_bridge(void) {
  if (device_ready() && !queue_empty()) {
    top->sim_input_data  = queue_pop();
    top->sim_input_valid = 1;
  } else {
    top->sim_input_valid = 0;
  }
}
```

这不是最终产品级 I/O，但它能让 shell 交互先工作起来。工程上，这种取舍很常见：先把主链路跑通，再逐步替换更真实的外设模型。

## UART 输出和脚本同步

自动测试最怕脚本把命令一股脑塞进去，而 shell 还没准备好。

更稳的方式是观察 prompt：

```c
void note_uart_char(char ch) {
  prompt_window.push(ch);
  keep_last_few_chars(prompt_window);
}

void script_tick(void) {
  if (!prompt_window.ends_with("xos> ")) return;

  if (command_inflight) {
    command_inflight = false;
    if (exit_after_cmds && queue_empty())
      running = false;
    return;
  }

  if (!queue_empty()) {
    inject_text(queue_pop() + "\n");
    command_inflight = true;
    prompt_window.clear();
  }
}
```

这个机制很重要。它不是等固定时间，而是等系统真的回到可接收命令的状态。

## 为什么要精简 firmware

如果目标是讲 CoolDA，就不要把无关 demo 全塞进 firmware。一个好的精简固件只需要：

1. 启动代码；
2. 基础输出；
3. heap；
4. timer；
5. shell；
6. CoolDA runtime；
7. CoolDA 命令；
8. BSP。

这样读者能看清主线：CPU 如何在 SoC 仿真里调 NPU。否则项目会被 UI、游戏、显示、历史实验等内容稀释。

## 小结

xOS shell + Verilator simulator 的价值在于把硬件 demo 变成系统 demo：

```text
不是 testbench 直接戳信号，
而是 CPU 跑软件，
软件调 runtime，
runtime 调 BSP，
BSP 写 APB NPU。
```

下一篇讲测试契约：如何判断这条跨层链路是真的健康，而不是某次演示刚好有输出。
