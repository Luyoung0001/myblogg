---
title: "RISC-V32 OS 实验（五）：多任务机制"
date: 2025-03-20 13:03:13
tags:
  - "RISC-V"
  - "multitasking"
  - "scheduler"
categories:
  - "Operating Systems"
---

## 多任务

首先创造两个任务：

```C
void user_task0(void)
{
	uart_puts("Task 0: Created!\n");
	while (1) {
		uart_puts("Task 0: Running...\n");
		task_delay(DELAY);
		task_yield();
	}
}

void user_task1(void)
{
	uart_puts("Task 1: Created!\n");
	while (1) {
		uart_puts("Task 1: Running...\n");
		task_delay(DELAY);
		task_yield();
	}
}
```

这两个任务相当简单，就来回不断地打印信息。

接着创建这两个任务的栈区以及上下文：

```C

uint8_t __attribute__((aligned(16))) task_stack[MAX_TASKS][STACK_SIZE];
struct context ctx_tasks[MAX_TASKS];


int task_create(void (*start_routin)(void)) {
    if (_top < MAX_TASKS) {
        ctx_tasks[_top].sp = (reg_t)&task_stack[_top][STACK_SIZE];
        ctx_tasks[_top].ra = (reg_t)start_routin;
        _top++;
        return 0;
    } else {
        return -1;
    }
}

void os_main(void){
	task_create(user_task0);
	task_create(user_task1);
}
```

接着，就可以调用 schedule 了：

```C
void schedule() {
    if (_top <= 0) {
        panic("Num of task should be greater than zero!");
        return;
    }

    _current = (_current + 1) % _top;
    struct context* next = &(ctx_tasks[_current]);
    switch_to(next);
}
```

在 start_kernel 调用 schedule 之后，切换到 task0，接着 task0 调用 task_yield 切换到 task1。task1 也会调用 task_yield，结果就是两个任务来回切换：

```bash
Press Ctrl-A and then X to exit QEMU
------------------------------------
Hello, RVOS!
HEAP_START = 0x800070ec(aligned to 0x80008000), HEAP_SIZE = 0x07ff8f14,
num of reserved pages = 8, num of pages to be allocated for heap = 32752
TEXT:   0x80000000 -> 0x80003104
RODATA: 0x80003104 -> 0x80003354
DATA:   0x80004000 -> 0x80004004
BSS:    0x80004010 -> 0x800070ec
HEAP:   0x80010000 -> 0x88000000
Task 0: Created!
Task 0: Running...
Task 1: Created!
Task 1: Running...
Task 0: Running...
Task 1: Running...
Task 0: Running...
```

这就是多任务的切换。




