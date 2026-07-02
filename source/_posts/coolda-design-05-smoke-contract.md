---
title: "CoolDA 设计仿真（五）：烟测不是跑命令，而是验证契约"
date: 2026-05-11 15:03:13
tags:
  - "CoolDA"
  - "testing"
  - "Verilator"
  - "verification"
categories:
  - "Computer Architecture"
---

## 前言

很多工程报告会把 smoke test 写成“跑了几条命令，看到 PASS”。这种写法对没有环境的读者不友好，也讲不出测试设计的价值。

更好的讲法是：smoke test 在验证跨层契约。CoolDA 这种 SoC demo 的 smoke test 应该证明：

1. 固件能构建；
2. SoC 能仿真；
3. shell 能接收脚本输入；
4. runtime 能调度任务；
5. BSP 能驱动 APB NPU；
6. NPU 结果能对齐 reference；
7. 仿真器能干净结束。

![CoolDA smoke 测试契约](/images/mermaid-svg/coolda-design-05-smoke-contract/test-contract.svg)

## smoke test 的层次

一个好的 smoke test 不是单点测试，而是链路测试：

```text
firmware build
  -> simulator build
  -> program load
  -> shell boot
  -> scripted commands
  -> runtime calls
  -> BSP MMIO
  -> NPU compute
  -> UART observations
  -> host-side assertions
```

这里每一层都可能坏：

| 层 | 可能问题 |
|---|---|
| firmware | 链接脚本、启动代码、符号缺失 |
| simulator | RTL/C++ 链接失败，模型接口变动 |
| shell | prompt 不出现，命令解析错 |
| runtime | tile 调度错，device heap 越界 |
| BSP | 寄存器 offset 错，busy/done 轮询错 |
| NPU RTL | 有符号乘法错，C 输出顺序错 |
| UART/script | 输出捕获错，命令注入乱序 |

smoke test 的任务就是用少量但高价值的断言覆盖这些层。

## matmul 功能契约

矩阵乘法功能测试应该包含 CPU reference：

```c
void cpu_matmul_ref(const job_t *job) {
  for (int row = 0; row < job->m; row++) {
    for (int col = 0; col < job->n; col++) {
      int32_t sum = 0;
      for (int k = 0; k < job->k; k++) {
        sum += (int32_t)job->a[row * job->lda + k] *
               (int32_t)job->b[k * job->ldb + col];
      }
      if ((job->flags & JOB_RELU) && sum < 0)
        sum = 0;
      job->c[row * job->ldc + col] = sum;
    }
  }
}
```

NPU 路径则走：

```text
host A/B
  -> device memory model
  -> runtime tile scheduler
  -> BSP
  -> APB NPU
  -> device C
  -> host C
```

最后逐项比较：

```c
for (int i = 0; i < total; i++) {
  assert(npu_c[i] == ref_c[i]);
}
```

这个测试验证的不只是 4x4 硬件核，还验证 runtime 是否正确使用了硬件核。

## runtime 内存契约

即使硬件没有问题，runtime 也可能出错。至少要检查：

1. `malloc(0)` 应失败；
2. 超出 device heap 的分配应失败；
3. host 指针和 device 指针不能混用；
4. memcpy to device 要检查目标范围；
5. memcpy to host 要检查来源范围；
6. free 后的空间要标记为不用。

可以把 device heap 想成一个小数组：

```c
static uint8_t device_heap[64 * 1024];

typedef struct {
  uint32_t offset;
  uint32_t bytes;
  uint8_t  in_use;
} alloc_t;
```

内存检查伪代码：

```c
int check_device_range(void *ptr, size_t bytes) {
  if (!is_device_ptr(ptr)) return -1;
  if (!range_inside_known_alloc(ptr, bytes)) return -1;
  return 0;
}
```

这类检查让 runtime 更像一个真正的加速器运行时，而不只是几层函数封装。

## event 契约

如果 runtime 提供 async/event，smoke test 应该验证事件状态：

```text
IDLE -> RUNNING -> DONE
```

以及错误情况：

```text
invalid job -> ERROR 或返回失败
```

一个 event poll 的核心契约是：

```c
int event_poll(event_t *e) {
  if (e->state != RUNNING) return error_or_done(e);

  run_one_tile_step(e);
  e->completed_steps++;

  if (e->completed_steps == e->total_steps)
    e->state = DONE;
}
```

测试不需要读者看终端输出。文章里应该讲清楚：每次 poll 推进一个 tile step，所有 step 完成后结果必须与 CPU reference 一致。

## vecadd 为什么也能出现在 smoke 里

如果当前硬件只实现 matmul，而 vecadd 仍由 CPU reference 实现，那 vecadd 测试还有意义吗？

有。它验证的是 runtime API 和 device memory 路径，而不是 RTL kernel。

vecadd job 很简单：

```c
typedef struct {
  uint32_t n;
  const int32_t *a;
  const int32_t *b;
  int32_t *c;
} vecadd_job_t;
```

reference：

```c
for (int i = 0; i < n; i++) {
  c[i] = a[i] + b[i];
}
```

它能覆盖：

1. device allocation；
2. memcpy to device；
3. launch API；
4. memcpy to host；
5. result compare；
6. benchmark 框架。

所以文章里要说清楚：vecadd 测的是 runtime 形状，不代表已经有 vecadd RTL accelerator。

## simulator 完成契约

自动测试还需要知道什么时候结束。

如果仿真器通过 shell prompt 注入命令，就应该有这样的契约：

```text
看到 prompt
  -> 注入下一条命令
  -> 等待命令执行结束并再次出现 prompt
  -> 如果队列为空，则停止仿真
```

这比“睡眠一段固定时间再输入下一条”可靠得多。

伪代码：

```c
if (prompt_seen()) {
  if (command_inflight) {
    command_inflight = false;
    if (exit_after_commands && command_queue_empty())
      running = false;
  } else if (!command_queue_empty()) {
    inject_next_command();
    command_inflight = true;
  }
}
```

这里验证的是 simulator、UART 捕获、shell prompt 和脚本注入之间的同步。

## benchmark 要克制解读

smoke test 里可以包含 benchmark 入口，但不要把它写成真实性能结论。

原因很简单：

1. Verilator 时间不是芯片真实时间；
2. 4x4 寄存器喂数小核不代表最终 NPU 性能；
3. 没有 DMA 时，CPU 参与数据搬运成本很高；
4. 裸机 timer 精度可能影响短 benchmark；
5. simulator 编译选项也会影响 host 侧耗时。

benchmark 在这个阶段更适合做回归信号：同一实现下，性能数字是否异常漂移。它不是最终性能报告。

## 小结

CoolDA smoke test 的重点不是“跑命令”，而是验证契约：

```text
构建契约
  -> shell 契约
  -> runtime 契约
  -> BSP/MMIO 契约
  -> NPU 数值契约
  -> simulator 完成契约
```

到这里，CoolDA 系列也形成了闭环：CPU 到 NPU 的系统路径、APB 寄存器契约、BSP/runtime 分层、xOS/Verilator 交互、跨层测试契约。后续要加 DMA、命令队列或中断，也可以沿着这些层逐步扩展，而不是推倒重来。
