---
title: "毕业设计记录（13）：Shell 命令扩展"
date: 2026-01-27 10:03:13
tags:
  - "graduation thesis"
  - "shell"
  - "command parser"
categories:
  - "Thesis Project"
---

本文详细介绍 xOS Shell 中各个命令的实现方式。Shell 是操作系统与用户交互的核心组件，通过命令表驱动的方式实现了多种实用命令。

## Shell 架构概述

### 命令表设计

xOS 使用静态命令表来管理所有可用命令：

```C
typedef struct {
    const char *name;           // 命令名称
    const char *help;           // 帮助信息
    shell_cmd_handler_t handler; // 命令处理函数
} shell_cmd_t;

static const shell_cmd_t commands[] = {
    {"help", "Show available commands", cmd_help},
    {"echo", "Echo arguments (e.g. echo hello world)", cmd_echo},
    {"clear", "Clear screen", cmd_clear},
    {"info", "Show system information", cmd_info},
    {"ps", "Show task status", cmd_ps},
    {"fg", "Move task to foreground (e.g. fg 1)", cmd_fg},
    {"bg", "Move task to background (e.g. bg 1)", cmd_bg},
    {"logs", "Show task output history (e.g. logs 1)", cmd_logs},
    {"heap", "Show heap memory statistics", cmd_heap},
    {"countdown", "Countdown from N (e.g. countdown 10)", cmd_countdown},
    {"hdmigc", "Run HDMI buffer garbage collection", cmd_hdmi_buffer_gc},
    {"kill", "Kill task (e.g. kill 1)", cmd_kill},
    {"change", "Change framebuffer (e.g. change A/B/S)", cmd_change},
    {NULL, NULL, NULL} /* End marker */
};
```

### 命令解析

命令行输入通过 `parse_command` 函数解析为 argc/argv 格式：

```C
static int parse_command(char *cmd, char *argv[], int max_args) {
    int argc = 0;
    char *p = cmd;

    while (*p && argc < max_args) {
        /* 跳过前导空格 */
        while (*p == ' ')
            p++;
        if (*p == '\0')
            break;

        /* 记录参数起始位置 */
        argv[argc++] = p;

        /* 找到参数结束位置 */
        while (*p && *p != ' ')
            p++;

        /* 空字符分隔 */
        if (*p) {
            *p++ = '\0';
        }
    }
    return argc;
}
```

### 前台与后台命令

命令分为两类执行方式：

1. **前台命令**：直接在 Shell 上下文中执行，如 `help`、`echo`、`clear` 等
2. **后台命令**：创建独立任务执行，如 `countdown`、`mario`、`tetris` 等

```C
static void execute_command(void) {
    // ... 解析命令 ...

    /* 判断是否需要后台运行 */
    int run_background = 0;
    if (strcmp(cmd->name, "countdown") == 0 ||
        strcmp(cmd->name, "hdmigc") == 0 ||
        strcmp(cmd->name, "mario") == 0 ||
        strcmp(cmd->name, "tetris") == 0) {
        run_background = 1;
    }

    if (run_background) {
        /* 关中断防止竞态条件 */
        irq_global_disable();

        /* 创建后台任务 */
        int tid = task_create(task_wrapper, cmd->name);
        // ... 保存命令上下文 ...

        irq_global_enable();
    } else {
        /* 前台直接执行 */
        cmd->handler(argc, argv);
    }
}
```

---

## 基础命令

### help - 显示帮助信息

遍历命令表，打印所有可用命令及其描述：

```C
int cmd_help(int argc, char *argv[]) {
    (void)argc;
    (void)argv;

    printf("Available commands:\n");
    const shell_cmd_t *cmd = commands;
    while (cmd->name) {
        printf("%s - %s\n", cmd->name, cmd->help);
        cmd++;
    }
    return 0;
}
```

### echo - 回显参数

将命令行参数原样输出，参数之间用空格分隔：

```C
int cmd_echo(int argc, char *argv[]) {
    for (int i = 1; i < argc; i++) {
        printf("%s", argv[i]);
        if (i < argc - 1) {
            printf(" ");
        }
    }
    printf("\n");
    return 0;
}
```

### clear - 清屏

使用 ANSI 转义序列清除屏幕并将光标移到左上角：

```C
int cmd_clear(int argc, char *argv[]) {
    (void)argc;
    (void)argv;

    /* ANSI escape sequence: 清屏 + 光标归位 */
    printf("\033[2J\033[H");
    return 0;
}
```

- `\033[2J`：清除整个屏幕
- `\033[H`：将光标移动到 (0,0) 位置

需要注意的是，改命令目前仅仅对于 `minicom` 有效，因为后面需要实现它的 hdmi 显示逻辑，比如清屏。

### info - 系统信息

显示 xOS 的硬件和软件配置信息：

```C
int cmd_info(int argc, char *argv[]) {
    printf("xOS System Information\n");
    printf("----------------------\n");
    printf("CPU:           LoongArch32R\n");
    printf("Board:         A7-Lite FPGA\n");
    printf("RAM:           512MB\n");
    printf("FRAMBUF:       0x1F000000\n");
    printf("FRAMEEND:      0x1FCFF000\n");
    printf("ROM:           8KB\n");
    printf("LIBC:          mylibc\n");
    printf("Keyboard:      PS2 (Interrupt-driven)\n");
    printf("Serial:        UART 9600 8N1\n");
    return 0;
}
```

---

## 进程管理命令

### ps - 进程状态

遍历任务表 `tasks[]`，显示所有活动进程的信息：

```C
int cmd_ps(int argc, char *argv[]) {
    const char *state_names[] = {"UNUSED", "READY ", "RUN   ", "DEAD  "};

    printf("TID STATE  FG NAME        OUTPUT\n");
    printf("--- ------ -- ----------- ------\n");

    int current = get_current_task();

    for (int i = 0; i < MAX_TASKS; i++) {
        const task_t *task = get_task_info(i);
        if (task && task->state != TASK_UNUSED) {
            /* 当前任务标记 * */
            if (i == current) {
                printf("*");
            } else {
                printf(" ");
            }

            printf("%d  ", i);                              // TID
            printf("%s ", state_names[task->state]);        // 状态
            printf("%s ", task->output.is_foreground ? "FG " : "BG "); // 前/后台
            printf("%s ", task->name);                      // 名称

            uint32_t output_kb = task->output.total_bytes / 1024;
            printf("%uKB\n", output_kb);                    // 输出缓冲区大小
        }
    }
}
```

任务状态定义在 `sched.h` 中：

```C
typedef enum {
    TASK_UNUSED = 0,    // 未使用的槽位
    TASK_READY,         // 就绪态，等待调度
    TASK_RUNNING,       // 运行态
    TASK_DEAD           // 已结束，等待清理
} task_state_t;
```

### fg - 移至前台

将指定任务设置为前台任务，其输出将直接显示到 UART：

```C
int cmd_fg(int argc, char *argv[]) {
    if (argc < 2) {
        printf("Usage: fg <task_id>\n");
        return -1;
    }

    /* 解析任务 ID */
    int tid = 0;
    char *p = argv[1];
    while (*p >= '0' && *p <= '9') {
        tid = tid * 10 + (*p - '0');
        p++;
    }

    if (tid < 0 || tid >= MAX_TASKS) {
        printf("Invalid task ID: %d\n", tid);
        return -1;
    }

    /* 调用调度器 API 设置前台 */
    if (task_set_foreground(tid, 1) < 0) {
        printf("Failed to set task %d to foreground\n", tid);
        return -1;
    }

    printf("Task %d moved to foreground\n", tid);
    return 0;
}
```

`task_set_foreground` 的实现会清除之前前台任务的标志：

```C
int task_set_foreground(int tid, int foreground) {
    // ... 参数检查 ...

    if (foreground) {
        /* 清除之前的前台任务 */
        if (tid != 0 && foreground_task != 0) {
            tasks[foreground_task].output.is_foreground = 0;
        }
        foreground_task = tid;
    }
    tasks[tid].output.is_foreground = foreground;
    return 0;
}
```

### bg - 移至后台

将任务移至后台运行，其输出将被缓存而非直接显示：

```C
int cmd_bg(int argc, char *argv[]) {
    if (argc < 2) {
        printf("Usage: bg <task_id>\n");
        return -1;
    }

    int tid = 0;
    char *p = argv[1];
    while (*p >= '0' && *p <= '9') {
        tid = tid * 10 + (*p - '0');
        p++;
    }

    if (tid == 0) {
        printf("Cannot move shell to background\n");
        return -1;  // Shell 不能移至后台
    }

    if (task_set_foreground(tid, 0) < 0) {
        printf("Failed to set task %d to background\n", tid);
        return -1;
    }

    printf("Task %d moved to background\n", tid);
    return 0;
}
```

### kill - 终止任务

强制终止指定任务，释放其资源：

```C
int cmd_kill(int argc, char *argv[]) {
    if (argc < 2) {
        printf("Usage: kill <task_id>\n");
        return -1;
    }

    int tid = 0;
    char *p = argv[1];
    while (*p >= '0' && *p <= '9') {
        tid = tid * 10 + (*p - '0');
        p++;
    }

    if (tid == 0) {
        printf("Cannot kill shell task\n");
        return -1;  // 不能杀死 Shell
    }

    if (task_kill(tid) < 0) {
        printf("Failed to kill task %d\n", tid);
        return -1;
    }

    printf("Task %d killed\n", tid);
    return 0;
}
```

`task_kill` 在调度器中的实现：

```C
int task_kill(int tid) {
    task_t *task = &tasks[tid];

    /* 检查任务状态 */
    if (task->state == TASK_UNUSED || task->state == TASK_DEAD) {
        return -1;
    }

    /* 释放 output buffer */
    if (task->output.buffer != NULL) {
        free(task->output.buffer);
        task->output.buffer = NULL;
    }

    /* 如果是前台任务，重置前台为 shell */
    if (foreground_task == tid) {
        foreground_task = 0;
        tasks[0].output.is_foreground = 1;
    }

    /* 标记为 DEAD */
    task->state = TASK_DEAD;
    num_tasks--;
    return 0;
}
```

### logs - 查看任务输出

显示指定任务的历史输出，使用环形缓冲区存储：

```C
int cmd_logs(int argc, char *argv[]) {
    if (argc < 2) {
        printf("Usage: logs <task_id>\n");
        return -1;
    }

    int tid = /* 解析 task_id */;
    const task_t *task = get_task_info(tid);

    if (!task || task->state == TASK_UNUSED) {
        printf("Task %d does not exist\n", tid);
        return -1;
    }

    printf("=== Task %d (%s) Output ===\n", tid, task->name);

    uint32_t total = task->output.total_bytes;
    uint32_t buf_size = TASK_OUTPUT_BUF_SIZE;  // 1MB

    if (total == 0) {
        printf("(no output)\n");
        return 0;
    }

    /* 确定起始位置（环形缓冲区） */
    uint32_t start_pos, bytes_to_print;
    if (total <= buf_size) {
        /* 缓冲区未满 */
        start_pos = 0;
        bytes_to_print = total;
    } else {
        /* 缓冲区已回绕，从 write_pos 开始是最旧的数据 */
        start_pos = task->output.write_pos;
        bytes_to_print = buf_size;
    }

    /* 打印缓冲区内容 */
    for (uint32_t i = 0; i < bytes_to_print; i++) {
        uint32_t pos = (start_pos + i) % buf_size;
        printf("%c", task->output.buffer[pos]);
    }

    printf("\n=== End of output (%u bytes) ===\n", total);
    return 0;
}
```

每个任务都有一个 1MB 的环形输出缓冲区：

```C
typedef struct {
    char *buffer;           // 指向 1MB 缓冲区
    uint32_t write_pos;     // 写入位置（循环）
    uint32_t total_bytes;   // 总写入字节数
    int is_foreground;      // 是否前台任务
} task_output_t;
```

---

## 内存管理命令

### heap - 堆内存统计

显示堆内存的使用情况：

```C
int cmd_heap(int argc, char *argv[]) {
    uint32_t total, used, free_size;
    heap_stats(&total, &used, &free_size);

    printf("Heap Memory Statistics\n");
    printf("----------------------\n");
    printf("Total:     %u MB (%u bytes)\n", total / (1024 * 1024), total);
    printf("Used:      %u KB (%u bytes)\n", used / 1024, used);
    printf("Free:      %u MB (%u bytes)\n", free_size / (1024 * 1024), free_size);
    printf("Usage:     %u%%\n", (used * 100) / total);
    return 0;
}
```

xOS 使用 First-Fit 算法的堆管理器，内存块结构如下：

```C
typedef struct block_header {
    size_t size;                    // 块大小（不含头部）
    struct block_header* next;      // 空闲链表下一个块
    int is_free;                    // 1=空闲, 0=已分配
} block_header_t;
```

`malloc` 实现采用首次适配策略，找到足够大的空闲块后进行分割：

```C
void* malloc(size_t size) {
    size = ALIGN(size);  // 8 字节对齐

    block_header_t* current = free_list;
    while (current) {
        if (current->is_free && current->size >= size) {
            /* 如果块足够大，进行分割 */
            if (current->size >= size + BLOCK_HEADER_SIZE + ALIGN_SIZE) {
                block_header_t* new_block = (block_header_t*)
                    ((char*)current + BLOCK_HEADER_SIZE + size);
                new_block->size = current->size - size - BLOCK_HEADER_SIZE;
                new_block->next = current->next;
                new_block->is_free = 1;

                current->size = size;
                current->next = new_block;
            }

            current->is_free = 0;
            heap_used += current->size + BLOCK_HEADER_SIZE;
            return (void*)((char*)current + BLOCK_HEADER_SIZE);
        }
        current = current->next;
    }
    return NULL;
}
```

`free` 释放内存后会合并相邻的空闲块：

```C
void free(void* ptr) {
    if (!ptr) return;

    block_header_t* block = (block_header_t*)((char*)ptr - BLOCK_HEADER_SIZE);
    block->is_free = 1;
    heap_used -= block->size + BLOCK_HEADER_SIZE;

    /* 合并相邻空闲块 */
    merge_free_blocks();
}
```

---

## 显示相关命令

### hdmigc - HDMI 缓冲区垃圾回收

这是一个后台任务，负责清理 HDMI 终端滚动后留下的旧行：

```C
int cmd_hdmi_buffer_gc(int argc, char *argv[]) {
    printf("hdmigc entered\n");

    while (1) {
        /* shell_gc_pointer 追踪已清理的行
         * display_start_row 是当前显示的起始行
         * 当两者不相等时，说明有旧行需要清理 */
        if (shell_gc_pointer != display_start_row) {
            /* 清理一行 */
            int pixel_row_start = shell_gc_pointer * TERMINAL_FONT_SIZE;
            int pixel_row_end = pixel_row_start + TERMINAL_FONT_SIZE;
            hdmi_clear_line(pixel_row_start, pixel_row_end, current_bg_color);

            /* 移动到下一行（环形） */
            shell_gc_pointer = (shell_gc_pointer + 1) % TERMINAL_TOTAL_ROWS;
        } else {
            /* 没有需要清理的行，让出 CPU */
            task_yield();
        }
    }
    return 0;
}
```

`task_yield` 通过触发软中断让出 CPU：

```C
void task_yield(void) {
    /* 触发 SWI0 软中断 */
    uint32_t estat = csr_read(CSR_ESTAT);
    estat |= ECFG_LIE_SWI0;
    csr_write(CSR_ESTAT, estat);
}
```

### change - 切换帧缓冲区

xOS 支持三个帧缓冲区，用于双缓冲和 Shell 显示：

```C
int cmd_change(int argc, char *argv[]) {
    if (argc < 2) {
        printf("Usage: change <buffer>\n");
        printf("  A - Buffer A (0x1F000000)\n");
        printf("  B - Buffer B (0x1F400000)\n");
        printf("  S - Buffer S (0x1F800000, Shell)\n");
        return -1;
    }

    char buffer_name = argv[1][0];
    enum BUFFER target_buffer;

    if (buffer_name == 'A' || buffer_name == 'a') {
        target_buffer = BUFFER_A;
    } else if (buffer_name == 'B' || buffer_name == 'b') {
        target_buffer = BUFFER_B;
    } else if (buffer_name == 'S' || buffer_name == 's') {
        target_buffer = BUFFER_S;
    } else {
        printf("Error: Invalid buffer '%c'\n", buffer_name);
        return -1;
    }

    /* 同时切换显示和写入缓冲区 */
    hdmi_fb_show_base_set(target_buffer);
    hdmi_fb_write_base_set(target_buffer);

    printf("Framebuffer switched successfully!\n");
    return 0;
}
```

---

## 非内置命令

### countdown - 倒计时

一个简单的后台任务示例，演示任务调度：

```C
int cmd_countdown(int argc, char *argv[]) {
    int count = 5;  // 默认值

    if (argc > 1) {
        /* 简单的 atoi 实现 */
        count = 0;
        char *p = argv[1];
        while (*p >= '0' && *p <= '9') {
            count = count * 10 + (*p - '0');
            p++;
        }
        if (count <= 0 || count > 100) {
            printf("Invalid count (must be 1-100)\n");
            return -1;
        }
    }

    printf("Countdown starting from %d...\n", count);

    for (int i = count; i > 0; i--) {
        printf("%d...\n", i);

        /* 忙等待延时 */
        volatile int delay;
        for (delay = 0; delay < 5000; delay++) {
            __asm__ volatile("nop");
        }
    }

    printf("Countdown complete!\n");
    return 0;
}
```

## 输入处理

### PS2 键盘扫描码处理

Shell 通过中断驱动方式接收 PS2 键盘输入：

```C
static void process_scancode(uint8_t scancode) {
    /* 处理扩展码前缀 */
    if (scancode == PS2_EXTENDED_CODE) {
        is_extended = 1;
        return;
    }

    /* 处理断码前缀 */
    if (scancode == PS2_BREAK_CODE) {
        is_break_code = 1;
        return;
    }

    /* 忽略按键释放事件 */
    if (is_break_code) {
        is_break_code = 0;
        is_extended = 0;
        return;
    }

    /* 转换扫描码为 ASCII */
    char c = bsp_ps2_to_ascii(scancode);
    if (c) {
        shell_input_char(c);
    }
}
```

### 字符输入处理

```C
void shell_input_char(char c) {
    switch (c) {
    case '\n':        /* Enter - 执行命令 */
        printf("\n");
        execute_command();
        cmd_pos = 0;
        shell_print_prompt();
        break;

    case '\b':        /* Backspace - 删除字符 */
        if (cmd_pos > 0) {
            cmd_pos--;
            /* 屏幕上擦除：退格、空格、退格 */
            bsp_uart_putc(0, '\b');
            bsp_uart_putc(0, ' ');
            bsp_uart_putc(0, '\b');
        }
        break;

    case 0x1B:        /* ESC - 清空当前行 */
        while (cmd_pos > 0) {
            cmd_pos--;
            bsp_uart_putc(0, '\b');
            bsp_uart_putc(0, ' ');
            bsp_uart_putc(0, '\b');
        }
        break;

    default:          /* 可打印字符 */
        if (c >= ' ' && c <= '~' && cmd_pos < SHELL_CMD_MAX_LEN - 1) {
            cmd_buffer[cmd_pos++] = c;
            putchar(c);  /* 回显 */
        }
        break;
    }
}
```

---

## 总结

xOS Shell 实现了一个功能完整的命令行界面：

| 命令 | 类型 | 功能 |
|------|------|------|
| help | 前台 | 显示帮助信息 |
| echo | 前台 | 回显参数 |
| clear | 前台 | 清屏 |
| info | 前台 | 系统信息 |
| ps | 前台 | 进程状态 |
| fg/bg | 前台 | 前后台切换 |
| kill | 前台 | 终止任务 |
| logs | 前台 | 查看任务输出 |
| heap | 前台 | 内存统计 |
| change | 前台 | 切换帧缓冲区 |
| countdown | 后台 | 倒计时演示 |
| hdmigc | 后台 | HDMI 垃圾回收 |

关键设计要点：

1. **命令表驱动**：易于扩展新命令
2. **前后台分离**：长时间运行的任务不阻塞 Shell
3. **环形输出缓冲**：后台任务输出可追溯
4. **中断驱动输入**：响应及时，不占用 CPU