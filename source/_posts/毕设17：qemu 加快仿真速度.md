---
title: 毕设(17)：利用 qemu 做快速仿真
date: 2026-01-30 10:03:13
tags: 毕设
categories: loongarch32r
mermaid: true
---

## 问题

调试 JIT 尤其是 difftest 的时候，在初期每次都要重新编译，烧录 sd 卡然后启动等待报错，再 debug。这个迭代周期非常多。如果每次都这样做，那么开发周期将会非常漫长，因此在 qemu 中直接跑 JIT 会大大加快开发进度。

## 显示设备

qemu 一般都带显示设备，但是我们的游戏设计要在 hdmi 这种设备播放，因此最好能找一个支持帧缓冲的 qemu 设备就最好了。好消息是真有这个设备：bochs-display。bochs-display 是 QEMU 内建的虚拟显卡，不需要另外安装驱动或插件。只要你的 `qemu-system-loongarch64` 编译时包含默认的显示设备（绝大多数发行版包都带），命令里的 `-device bochs-display,...` 就能直接用。可快速自检：
- `qemu-system-loongarch64 -device help | grep bochs-display` 能看到条目即已内置。
- Guest 侧只需按我们代码里的 MMIO/FBDRAW 驱动访问寄存器，无需额外驱动。

在非 QEMU（真实板卡）路径，`io_write(AM_GPU_FBDRAW, xpad, ypad, canvas, SCR_W, SCR_H,true)` 走到 `am_gpu_fbdraw` 的 “#else” 分支，流程是：

- pixels 指向一块连续的 ARGB32 缓冲（NES 画面 256×240）。
- 驱动获得 HDMI 显存基址 fb = hdmi_get_fb_pointer()（分辨率固定 1920×1080，RGB565）。
- 先做边界检查（x/y+w/h 不能越 1920×1080）。
- 逐行把 ARGB32 转换成 RGB565：
- R = (argb >> 8) & 0xF800，G = (argb >> 5) & 0x07E0，B = (argb >> 3) & 0x001F。
- 合成为 16 位写入目标帧缓冲。
- 内联汇编加速内层循环，fb_row 每行步长 1920。
- 如果 sync == true（NES 提交时传的是 true），调用 `hdmi_swap_buffers()` 执行双缓冲翻转；所以真实硬件是双缓冲，避免撕裂。
- 与 QEMU 路径不同，这里确实需要色彩转换为 RGB565。

QEMU 路径就简单多了：bochs-display 暴露线性 32 位帧缓冲，格式与 pixels 相同（ARGB32），所以直接逐像素拷贝即可，不用做色彩转换或缓冲切换。

- 上层 fce_update_screen() 只调用一次 am_gpu_fbdraw/qemu_fb_blit 生成整帧，拷贝完成后 QEMU/GTK 读取同一块内存展示。
- 没有硬件 VSync，理论上有撕裂风险，但 NES 分辨率低、单次拷贝快，实际肉眼几乎看不到。如果需要更稳，可在 NES 侧做软件双缓冲或用 memcpy 整帧写入以缩短窗口。


## 程序地址布局

程序能否正确运行，很大程度上取决于链接脚本的正确配置。QEMU 和真实硬件的内存布局有显著差异，因此需要两套不同的链接脚本。

### 非 QEMU 的内存映像（真实硬件）

真实硬件使用 `linker.ld`，内存布局如下：

```
512MB DDR 内存布局:
┌─────────────────────────────────────────────────────────┐
│ 0x00000000 - 0x1CFFFFFF : RAM (464MB)                   │
│   ├── .text.entry (0x00000000) : 启动代码               │
│   ├── .exception  (0x00000040) : 异常/中断入口          │
│   ├── .text       : 代码段                              │
│   ├── .rodata     : 只读数据                            │
│   ├── .data       : 已初始化数据                        │
│   ├── .bss        : 未初始化数据                        │
│   └── heap        : 堆 (向上增长至 0x1CFFFFFF)          │
├─────────────────────────────────────────────────────────┤
│ 0x1D000000 - 0x1EFFFFFF : Stack (32MB, 向下增长)        │
├─────────────────────────────────────────────────────────┤
│ 0x1F000000 - 0x1FFFFFFF : Framebuffer (16MB)            │
│   ├── BUFFER_A (0x1F000000) : 双缓冲 A                  │
│   ├── BUFFER_B (0x1F400000) : 双缓冲 B                  │
│   └── BUFFER_S (0x1F800000) : Shell 终端缓冲            │
├─────────────────────────────────────────────────────────┤
│ 0x20000000 - 0x20000FFF : Boot ROM (4KB, BRAM)          │
└─────────────────────────────────────────────────────────┘
```

关键配置：

```ld
/* linker.ld 核心配置 */
MEMORY {
    RAM (rwx) : ORIGIN = 0x00000000, LENGTH = 464M
}

/* 启动代码必须在地址 0x00 */
. = 0x00000000;
.text.entry : {
    KEEP(*(.text.start))
    . = 0x40;  /* 填充到 0x40，为异常入口预留空间 */
} > RAM

/* 异常入口紧随其后，地址 0x40 */
.exception : {
    KEEP(*(.exception))
} > RAM

/* 堆和栈区域 */
_heap_start = .;
_heap_end = 0x1D000000;
_stack_bottom = 0x1D000000;
_stack_top = 0x1F000000;

/* Framebuffer 区域 */
_framebuf_base = 0x1F000000;
_framebuf_end = 0x20000000;
```

### QEMU 版本的内存映像

QEMU virt 机器使用 `linker_qemu.ld`，内存布局有所不同：

```
QEMU virt 机器内存布局:
┌─────────────────────────────────────────────────────────┐
│ 0x00100000 - 0x00200000 : FDT (设备树，QEMU 自动放置)   │
├─────────────────────────────────────────────────────────┤
│ 0x00200000 - 0x0DFFFFFF : RAM (约 220MB)                │
│   ├── .text.entry (0x00200000) : 启动代码               │
│   ├── .exception  (ALIGN 64)   : 异常/中断入口          │
│   ├── .text       : 代码段                              │
│   ├── .rodata     : 只读数据                            │
│   ├── .data       : 已初始化数据                        │
│   ├── .bss        : 未初始化数据                        │
│   └── heap        : 堆 (向上增长至 0x0E000000)          │
├─────────────────────────────────────────────────────────┤
│ 0x0E000000 - 0x0EFFFFFF : Stack (16MB, 向下增长)        │
├─────────────────────────────────────────────────────────┤
│ 0x0F000000 - 0x0FFFFFFF : Framebuffer (16MB, 内存模拟)  │
├─────────────────────────────────────────────────────────┤
│ 0x1FE001E0              : UART (ns16550a)               │
├─────────────────────────────────────────────────────────┤
│ 0x20000000              : PCI ECAM 配置空间             │
├─────────────────────────────────────────────────────────┤
│ 0x40000000              : PCI MMIO 空间 (bochs-display) │
└─────────────────────────────────────────────────────────┘
```

关键配置：

```ld
/* linker_qemu.ld 核心配置 */

/* 程序起始地址必须避开 FDT */
_PROGRAM_START = 0x00200000;

MEMORY {
    RAM (rwx) : ORIGIN = 0x00200000, LENGTH = 256M
}

/* 启动代码从 0x00200000 开始 */
. = _PROGRAM_START;
.text.entry : {
    KEEP(*(.text.start))
} > RAM

/* 异常入口需要 64 字节对齐 */
.exception ALIGN(64) : {
    KEEP(*(.exception))
} > RAM

/* 堆和栈区域 */
_heap_start = .;
_heap_end = 0x0E000000;
_stack_bottom = 0x0E000000;
_stack_top = 0x0F000000;

/* Framebuffer (QEMU 无 HDMI，用内存模拟) */
_framebuf_base = 0x0F000000;
_framebuf_end = 0x10000000;
```

### 两者对比

| 项目 | 真实硬件 | QEMU |
|------|----------|------|
| 程序起始地址 | 0x00000000 | 0x00200000 |
| 异常入口地址 | 0x00000040 | ALIGN(64) |
| 堆结束地址 | 0x1D000000 | 0x0E000000 |
| 栈顶地址 | 0x1F000000 | 0x0F000000 |
| Framebuffer | 硬件 HDMI | 内存模拟 |
| UART 地址 | 0x1FE001E0 | 0x1FE001E0 (相同) |

## PS2、中断等

外设驱动是 QEMU 和真实硬件差异最大的部分。需要通过条件编译 `#ifdef QEMU_RUN` 来区分两种环境。

### 非 QEMU 情况（真实硬件）

真实硬件上，PS2 键盘通过自定义的 MMIO 接口访问：

```c
/* PS2 寄存器地址（真实硬件） */
#define PS2_BASE        0x1FD0F050
#define PS2_DATA_REG    (*(volatile uint32_t *)0x1FD0F050)
#define PS2_STATUS_REG  (*(volatile uint32_t *)0x1FD0F054)
#define PS2_CTRL_REG    (*(volatile uint32_t *)0x1FD0F058)

/* 状态寄存器位定义 */
#define PS2_STATUS_VALID       (1 << 0)  /* 有新数据 */
#define PS2_STATUS_PARITY_ERR  (1 << 1)  /* 奇偶校验错误 */

/* 控制寄存器位定义 */
#define PS2_CTRL_ENABLE        (1 << 1)  /* PS2 接收使能 */
#define PS2_CTRL_INT_ENABLE    (1 << 2)  /* 中断使能 */
```

中断处理流程：

```c
/* main.c 中的 PS2 中断处理 */
void ps2_irq_handler(trap_frame_t *tf) {
    while (bsp_ps2_data_available()) {
        uint8_t scancode = (uint8_t)bsp_ps2_read();
        int next = (kb_head + 1) % KB_BUF_SIZE;
        if (next != kb_head) {
            kb_buffer[kb_head] = scancode;
            kb_head = next;
        }
    }
}

/* 系统初始化时注册 PS2 中断 */
irq_register(IRQ_PS2, ps2_irq_handler);
bsp_ps2_init(1);  /* 1 = 使能中断 */
```

### QEMU 情况

QEMU virt 机器没有 PS2 控制器，因此需要禁用 PS2 相关代码：

```c
/* ps2.h 中的 QEMU 适配 */
#ifdef QEMU_RUN
// QEMU 模式：PS2 不可用，使用虚拟地址（不会被访问）
#define PS2_BASE        0x0FD0F050
#define PS2_DATA_REG    (*(volatile uint32_t *)0x0FD0F050)
#define PS2_STATUS_REG  (*(volatile uint32_t *)0x0FD0F054)
#else
// 真实硬件模式
#define PS2_BASE        0x1FD0F050
...
#endif
```

在 `main.c` 中跳过 PS2 初始化：

```c
static void system_init(void) {
#ifndef QEMU_RUN
    /* 只在真实硬件上注册 PS2 中断 */
    irq_register(IRQ_PS2, ps2_irq_handler);
    bsp_ps2_init(1);
#endif
    ...
}
```

### 中断系统对比

| 中断源 | 真实硬件 | QEMU |
|--------|----------|------|
| 定时器 (IRQ_TIMER) | 支持 | 支持 |
| PS2 键盘 (IRQ_PS2) | 支持 | 不支持 |
| UART (IRQ_UART) | 支持 | 支持 |
| 软中断 (SWI0) | 支持 | 支持 |

中断初始化代码：

```c
/* trap.c */
void trap_init(void) {
    /* 初始化所有中断处理函数为默认 */
    for (int i = 0; i < IRQ_MAX; i++) {
        irq_handlers[i] = default_irq_handler;
    }

    /* 注册 SWI0 处理函数（用于 task_yield） */
    irq_handlers[IRQ_SWI0] = swi0_irq_handler;

    /* 设置异常入口点 */
    uint32_t entry_addr;
    asm volatile ("la.local %0, trap_entry" : "=r"(entry_addr));
    write_csr_eentry(entry_addr);

    /* 使能 SWI0、UART 和 PS2 中断 */
    uint32_t ecfg = ECFG_LIE_SWI0 | ECFG_LIE_HWI1 | ECFG_LIE_HWI2;
    write_csr_ecfg(ecfg);
}
```

## 启动流程

### 非 QEMU 版本（真实硬件）

真实硬件的启动流程：

```
┌─────────────────────────────────────────────────────────┐
│ 1. FPGA 上电，从 Boot ROM (0x20000000) 加载启动代码     │
│    └── Boot ROM 将程序从 SD 卡加载到 RAM (0x00000000)   │
├─────────────────────────────────────────────────────────┤
│ 2. 跳转到 _start (0x00000000)                           │
│    └── start.S: 设置栈指针、清零 BSS、调用 main        │
├─────────────────────────────────────────────────────────┤
│ 3. main() 执行系统初始化                                │
│    ├── terminal_init()  : HDMI 终端初始化               │
│    ├── heap_init()      : 堆内存初始化                  │
│    ├── trap_init()      : 中断系统初始化                │
│    ├── sched_init()     : 调度器初始化                  │
│    ├── bsp_ps2_init(1)  : PS2 键盘初始化（使能中断）    │
│    └── timer_init()     : 定时器初始化                  │
├─────────────────────────────────────────────────────────┤
│ 4. 创建 shell 任务，启动定时器，进入调度循环            │
└─────────────────────────────────────────────────────────┘
```

### QEMU 版本

QEMU 的启动流程有所不同：

```
┌─────────────────────────────────────────────────────────┐
│ 1. QEMU 加载 ELF 文件到内存 (0x00200000)                │
│    └── 使用 -kernel 参数直接加载，无需 Boot ROM        │
├─────────────────────────────────────────────────────────┤
│ 2. 跳转到 _start (0x00200000)                           │
│    └── start.S: 设置栈指针、清零 BSS、调用 main        │
├─────────────────────────────────────────────────────────┤
│ 3. main() 执行系统初始化（QEMU 模式）                   │
│    ├── heap_init()      : 堆内存初始化                  │
│    ├── trap_init()      : 中断系统初始化                │
│    ├── sched_init()     : 调度器初始化                  │
│    ├── qemu_fb_init()   : bochs-display 初始化          │
│    └── timer_init()     : 定时器初始化                  │
│    注意：跳过 terminal_init() 和 bsp_ps2_init()         │
├─────────────────────────────────────────────────────────┤
│ 4. 创建 shell 任务，启动定时器，进入调度循环            │
└─────────────────────────────────────────────────────────┘
```

启动代码 `start.S` 是通用的：

```asm
/* start.S - 通用启动代码 */
    .section .text.start
    .globl _start

_start:
    /* 1. 设置栈指针（使用链接脚本定义的 __stack_top） */
    la.local    $sp, __stack_top

    /* 2. 清零 BSS 段 */
    la.local    $t0, __bss_start
    la.local    $t1, __bss_end
.L_clear_bss:
    beq         $t0, $t1, .L_bss_done
    st.w        $zero, $t0, 0
    addi.w      $t0, $t0, 4
    b           .L_clear_bss

.L_bss_done:
    /* 3. 调用 main */
    bl          main

    /* 4. main 返回后死循环 */
.L_halt:
    b           .L_halt
```

---

## 条件编译机制

通过 Makefile 传递 `QEMU_RUN` 宏来区分编译目标：

```makefile
# common.mk 中的条件编译支持
ifdef QEMU_RUN
CFLAGS  += -DQEMU_RUN
endif

# 根据 QEMU_RUN 选择不同的链接脚本
ifdef QEMU_RUN
LDSCRIPT := $(SCRIPTS_DIR)/linker_qemu.ld
else
LDSCRIPT := $(SCRIPTS_DIR)/linker.ld
endif
```

编译命令：

```bash
# 编译真实硬件版本
make clean && make

# 编译 QEMU 版本
make clean && make QEMU_RUN=1
```

---

## QEMU Framebuffer 驱动详解

bochs-display 是 QEMU 内置的 PCI 显卡设备，需要通过 PCI 配置空间来初始化。

### PCI 设备扫描

```c
/* qemu_fb.c - PCI 配置空间访问 */
#define PCI_ECAM_BASE       0x20000000
#define BOCHS_VGA_VENDOR    0x1234
#define BOCHS_VGA_DEVICE    0x1111

/* 计算 PCI ECAM 地址 */
static inline uint32_t pci_ecam_addr(int bus, int dev, int func, int reg) {
    return PCI_ECAM_BASE | (bus << 20) | (dev << 15) | (func << 12) | reg;
}

/* 扫描 PCI 总线查找 bochs-display */
static int pci_find_bochs_display(int *out_bus, int *out_dev) {
    for (int bus = 0; bus < 1; bus++) {
        for (int dev = 0; dev < 32; dev++) {
            uint16_t vendor = pci_read16(bus, dev, 0, PCI_VENDOR_ID);
            uint16_t device = pci_read16(bus, dev, 0, PCI_DEVICE_ID);
            if (vendor == BOCHS_VGA_VENDOR && device == BOCHS_VGA_DEVICE) {
                *out_bus = bus;
                *out_dev = dev;
                return 0;
            }
        }
    }
    return -1;
}
```

### VBE 寄存器配置

bochs-display 使用 VBE (VESA BIOS Extensions) 寄存器来配置显示模式：

```c
/* VBE 寄存器索引 */
#define VBE_DISPI_INDEX_XRES        0x01  /* 水平分辨率 */
#define VBE_DISPI_INDEX_YRES        0x02  /* 垂直分辨率 */
#define VBE_DISPI_INDEX_BPP         0x03  /* 位深度 */
#define VBE_DISPI_INDEX_ENABLE      0x04  /* 使能控制 */

#define VBE_DISPI_ENABLED           0x01
#define VBE_DISPI_LFB_ENABLED       0x40  /* 线性帧缓冲使能 */

/* 设置显示分辨率 */
int qemu_fb_set_resolution(uint16_t width, uint16_t height) {
    vbe_write(VBE_DISPI_INDEX_ENABLE, VBE_DISPI_DISABLED);
    vbe_write(VBE_DISPI_INDEX_XRES, width);
    vbe_write(VBE_DISPI_INDEX_YRES, height);
    vbe_write(VBE_DISPI_INDEX_BPP, 32);  /* 32 位色深 */
    vbe_write(VBE_DISPI_INDEX_ENABLE,
              VBE_DISPI_ENABLED | VBE_DISPI_LFB_ENABLED);

    fb_info.width = width;
    fb_info.height = height;
    fb_info.bpp = 32;
    fb_info.pitch = width * 4;
    return 0;
}
```

### Framebuffer 初始化流程

```c
int qemu_fb_init(void) {
    int bus, dev;

    /* 1. 扫描 PCI 总线查找 bochs-display */
    if (pci_find_bochs_display(&bus, &dev) < 0) {
        printf("[QEMU_FB] bochs-display not found\n");
        return -1;
    }

    /* 2. 读取 BAR0 (帧缓冲地址) 和 BAR2 (MMIO 寄存器) */
    uint32_t bar0 = pci_read32(bus, dev, 0, PCI_BAR0) & ~0xF;
    uint32_t bar2 = pci_read32(bus, dev, 0, PCI_BAR2) & ~0xF;

    /* 3. 如果 BAR 未分配，手动分配 */
    if (bar0 == 0) pci_write32(bus, dev, 0, PCI_BAR0, 0x40000000);
    if (bar2 == 0) pci_write32(bus, dev, 0, PCI_BAR2, 0x41000000);

    /* 4. 使能 PCI 内存访问 */
    uint16_t cmd = pci_read16(bus, dev, 0, PCI_COMMAND);
    pci_write16(bus, dev, 0, PCI_COMMAND, cmd | 0x03);

    /* 5. 设置分辨率 */
    qemu_fb_set_resolution(640, 480);

    fb_info.initialized = true;
    return 0;
}
```

### AM 适配层的 QEMU 分支

`am_adapter.c` 中通过条件编译区分 QEMU 和真实硬件：

```c
void am_gpu_fbdraw(int x, int y, uint32_t *pixels, int w, int h, bool sync) {
#ifdef QEMU_RUN
    /* QEMU 模式：直接写入 32 位 framebuffer，无需颜色转换 */
    qemu_fb_blit(x, y, pixels, w, h);
    (void)sync;  /* QEMU 不需要同步 */
#else
    /* 硬件模式：ARGB32 -> RGB565 转换 + HDMI 双缓冲 */
    volatile uint16_t *fb = hdmi_get_fb_pointer();
    // ... 颜色转换和内联汇编优化 ...
    if (sync) hdmi_swap_buffers();
#endif
}
```

---

## QEMU 运行命令

编译完成后，使用以下命令启动 QEMU：

```bash
qemu-system-loongarch64 \
    -M virt \
    -m 256M \
    -cpu la464 \
    -kernel build/xos.elf \
    -nographic \
    -device bochs-display \
    -serial mon:stdio
```

参数说明：

| 参数 | 说明 |
|------|------|
| `-M virt` | 使用 virt 虚拟机器 |
| `-m 256M` | 分配 256MB 内存 |
| `-cpu la464` | 使用 LA464 CPU 模型 |
| `-kernel` | 直接加载 ELF 文件 |
| `-nographic` | 无图形界面（串口输出） |
| `-device bochs-display` | 添加 bochs 显卡 |
| `-serial mon:stdio` | 串口重定向到终端 |

---

## UART 配置差异

UART 地址在 QEMU 和真实硬件上相同（0x1FE001E0），但时钟频率不同：

```c
/* uart.h */
#ifdef QEMU_RUN
#define UART_CLK 100000000      /* QEMU virt 机器时钟 100MHz */
#else
#define UART_CLK 75000000       /* 真实硬件时钟 75MHz */
#endif

#define UART_BASE 0x1FE001E0    /* 地址相同 */
```

波特率计算：

```c
/* divisor = CLK / (16 * baudrate) */
/* 9600 波特率:
 *   QEMU:  100000000 / (16 * 9600) = 651
 *   硬件:  75000000 / (16 * 9600) = 488
 */
```

---

## 总结

### QEMU 仿真的优势

| 方面 | 真实硬件 | QEMU 仿真 |
|------|----------|-----------|
| 编译-测试周期 | 编译→烧录SD卡→上电→等待 | 编译→直接运行 |
| 调试能力 | 有限（UART 打印） | GDB 远程调试 |
| 迭代速度 | 慢（分钟级） | 快（秒级） |
| 硬件依赖 | 需要 FPGA 板卡 | 仅需 PC |

### 移植要点清单

1. **链接脚本**：调整程序起始地址（避开 FDT）
2. **外设驱动**：条件编译区分 QEMU/硬件
3. **显示设备**：使用 bochs-display 替代 HDMI
4. **中断系统**：禁用不支持的外设中断
5. **时钟配置**：调整 UART 时钟频率

### 文件结构

```
scripts/
├── linker.ld           # 真实硬件链接脚本
├── linker_qemu.ld      # QEMU 链接脚本
└── common.mk           # 通用编译规则

bsp/
├── start.S             # 通用启动代码
├── src/
│   ├── uart.c          # UART 驱动（通用）
│   ├── ps2.c           # PS2 驱动（仅硬件）
│   ├── hdmi.c          # HDMI 驱动（仅硬件）
│   └── qemu_fb.c       # QEMU 显示驱动
└── include/
    ├── uart.h          # 条件编译时钟频率
    ├── ps2.h           # 条件编译寄存器地址
    └── qemu_fb.h       # QEMU FB 接口

software/xos_pro_max/
├── src/
│   ├── main.c          # 条件编译初始化流程
│   └── litenes/
│       └── am_adapter.c # AM 适配层（条件编译）
└── Makefile            # 支持 QEMU_RUN=1
```

通过 QEMU 仿真，JIT 和 difftest 的开发调试效率大幅提升，从原来的"编译-烧录-等待-调试"循环缩短为"编译-运行"，极大加快了开发迭代速度。



