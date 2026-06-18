---
title: "RISC-V32 OS 实验（十一）：软件定时器"
date: 2025-03-28 13:03:13
tags:
  - "RISC-V"
  - "timer"
  - "scheduler"
categories:
  - "Operating Systems"
---

## 软件定时器

软件定时器和系统实现 _tick 基本没有什么差别。

硬件定时器：芯片本身提供的定时器，一般由外部晶振提供，提供寄存器设置超时时间，并采用外部中断方式通知 CPU，参考 第 12 章介绍。优点是精度高，但定时器个数受硬件芯片的设计限制。

软件定时器：操作系统中基于硬件定时器提供的功能，采用软件方式实现。扩展了硬件定时器的限制，可以提供数目更多（几乎不受限制）的定时器；缺点是精度较低，必须是 Tick 的整数倍。

按照定时器设定方式分：
- 单次触发定时器：创建后只会触发一次定时器通知事件，触发后该定时器自动停止（销毁）
- 周期触发定时器：创建后按照设定的周期无限循环触发定时器通知事件，直到用户手动停止。

按照定时器超时后执行处理函数的上下文环境分：
- 超时函数运行在中断上下文环境中，要求执行函数的执行时间尽可能短，不可以执行等待其他事件等可能导致中断控制路径挂起的操作。优点是响应比较迅速，实时性较高。
- 超时函数运行在任务上下文环境中，即创建一个任务来执行这个函数，函数中可以等待或者挂起，但实时性较差。


## 实现定时任务

在由于时钟中断进入 trap 之后，会调用 trap_handler，只需要处理 time_check 就行：

```C
static inline void timer_check() {
    struct timer* t = &(timer_list[0]);
    for (int i = 0; i < MAX_TIMER; i++) {
        if (NULL != t->func) {
            if (_tick >= t->timeout_tick) {
                t->func(t->arg);
                t->func = NULL;
                t->arg = NULL;

                break;
            }
        }
        t++;
    }
}
```

timer_check 会查询到期的任务，执行之后销毁。

那么怎么添加任务呢？

```C
struct userdata person = {0, "Jack"};

void timer_func(void* arg) {
    if (NULL == arg)
        return;

    struct userdata* param = (struct userdata*)arg;

    param->counter++;
    printf("======> TIMEOUT: %s: %d\n", param->str, param->counter);
}

void user_task0(void) {
    uart_puts("Task 0: Created!\n");

    struct timer* t1 = timer_create(timer_func, &person, 3);
    if (NULL == t1) {
        printf("timer_create() failed!\n");
    }
    struct timer* t2 = timer_create(timer_func, &person, 5);
    if (NULL == t2) {
        printf("timer_create() failed!\n");
    }
    struct timer* t3 = timer_create(timer_func, &person, 7);
    if (NULL == t3) {
        printf("timer_create() failed!\n");
    }
    while (1) {
        uart_puts("Task 0: Running... \n");
        task_delay(DELAY);
    }
}
```

这两个函数注册了三个定时任务，在 trap 中可以通过 `t->func(t->arg)` 来调用注册的任务。

运行效果：

```bash
Task 0: Created!
Task 0: Running...
Task 0: Running...
Task 0: Running...
timer interruption!
tick: 1
Task 1: Created!
Task 1: Running...
Task 1: Running...
Task 1: Running...
timer interruption!
tick: 2
Task 0: Running...
Task 0: Running...
Task 0: Running...
timer interruption!
tick: 3
======> TIMEOUT: Jack: 1
Task 1: Running...
Task 1: Running...
Task 1: Running...
timer interruption!
tick: 4
Task 0: Running...
Task 0: Running...
timer interruption!
tick: 5
======> TIMEOUT: Jack: 2
Task 1: Running...
Task 1: Running...
timer interruption!
tick: 6
Task 0: Running...
Task 0: Running...
Task 0: Running...
timer interruption!
tick: 7
======> TIMEOUT: Jack: 3
Task 1: Running...
Task 1: Running...
Task 1: Running...
timer interruption!
tick: 8
Task 0: Running...
QEMU: Terminated
```

可以看到，任务正常运行。提一句，这里的执行任务是在 trap 中执行的，这会影响实时性，毕竟此时的中断是关闭的。




