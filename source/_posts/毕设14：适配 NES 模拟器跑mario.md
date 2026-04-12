---
title: 毕设(14)：适配 NES 模拟器跑 mario
date: 2026-01-28 10:03:13
tags: 毕设
categories: loongarch32r
mermaid: true
---

本文介绍如何将基于 AbstractMachine (AM) 抽象层的 LiteNES 模拟器移植到 xOS 上运行。重点讲解 AM 的设计理念，以及如何通过实现 AM 适配层让应用程序在不同平台间无缝迁移。

## AbstractMachine 简介

### 什么是 AbstractMachine

AbstractMachine (AM) 是南京大学计算机系统基础课程设计的一套**硬件抽象层**。它的核心思想是：

> **将应用程序与底层硬件解耦，通过一组标准化的 API 接口，让同一份代码可以运行在不同的硬件平台上。**

AM 定义了几个核心抽象：

```
┌─────────────────────────────────────────────────────┐
│                   应用程序 (如 LiteNES)               │
├─────────────────────────────────────────────────────┤
│              AbstractMachine API 层                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │
│  │  TRM    │ │   IOE   │ │   CTE   │ │   VME   │   │
│  │ 图灵机  │ │ I/O扩展 │ │上下文扩展│ │虚存扩展 │   │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │
├─────────────────────────────────────────────────────┤
│              平台相关实现 (native/x86/riscv...)      │
└─────────────────────────────────────────────────────┘
```

对于 NES 模拟器这类应用，主要使用 **IOE (I/O Extension)** 扩展，包括：

| 设备 | 功能 | AM 接口 |
|------|------|---------|
| 定时器 | 获取系统运行时间 | `AM_TIMER_UPTIME` |
| 键盘 | 读取按键事件 | `AM_INPUT_KEYBRD` |
| GPU | 获取屏幕配置、绘制像素 | `AM_GPU_CONFIG`, `AM_GPU_FBDRAW` |

### AM 的优势

1. **可移植性**：应用只需调用 AM API，无需关心底层硬件差异
2. **易于测试**：可以在 native (Linux/macOS) 环境快速开发调试
3. **教学友好**：学生可以专注于系统原理，而非硬件细节

---

## LiteNES 模拟器架构

LiteNES 是一个轻量级的 NES (Nintendo Entertainment System) 模拟器，原本基于 AM 接口开发。其架构如下：

```
┌─────────────────────────────────────────────────────┐
│                    fce.c (主循环)                    │
│         加载 ROM → 初始化 → 运行帧循环               │
├──────────────┬──────────────┬───────────────────────┤
│   cpu.c      │    ppu.c     │       psg.c           │
│  6502 CPU    │  图形处理器   │     手柄输入          │
│  指令解释器   │  背景/精灵渲染│   按键状态管理        │
├──────────────┴──────────────┴───────────────────────┤
│                  memory.c (内存映射)                 │
│         CPU RAM / PPU RAM / ROM / I/O 寄存器        │
├─────────────────────────────────────────────────────┤
│                AbstractMachine API                   │
│    io_read(AM_TIMER_UPTIME)  io_read(AM_INPUT_KEYBRD)│
│    io_read(AM_GPU_CONFIG)    io_write(AM_GPU_FBDRAW) │
└─────────────────────────────────────────────────────┘
```

LiteNES 通过 AM 的 `io_read` 和 `io_write` 宏访问硬件：

```C
// 获取系统运行时间（微秒）
static inline int uptime_ms() {
    return io_read(AM_TIMER_UPTIME).us / 1000;
}

// 绘制一帧画面
io_write(AM_GPU_FBDRAW, x, y, canvas, SCR_W, SCR_H, true);

// 读取键盘输入
AM_INPUT_KEYBRD_T ev = io_read(AM_INPUT_KEYBRD);
if (ev.keycode == AM_KEY_W) { /* 处理上键 */ }
```

---

## 移植策略：实现 AM 适配层

要让 LiteNES 在 xOS 上运行，核心任务是**将 AM 接口映射到 xOS 的驱动 API**。

### 适配层设计

创建 `am_adapter.c` 文件，实现 AM 接口到 xOS 的桥接：

```
┌─────────────────────────────────────────────────────┐
│                    LiteNES                          │
│         io_read(AM_*)    io_write(AM_*)             │
├─────────────────────────────────────────────────────┤
│                  am.h (接口定义)                     │
│    AM_TIMER_UPTIME_T  AM_GPU_CONFIG_T  ...          │
├─────────────────────────────────────────────────────┤
│               am_adapter.c (适配实现)                │
│    am_timer_uptime()  am_gpu_fbdraw()  ...          │
├─────────────────────────────────────────────────────┤
│                   xOS 驱动层                         │
│    timer.c    hdmi.c    ps2.c                       │
└─────────────────────────────────────────────────────┘
```

---

## AM 接口定义

首先在 `am.h` 中定义 AM 的数据类型和接口：

```C
// am.h - AbstractMachine 适配层接口定义
#ifndef AM_H
#define AM_H

#include <stdint.h>
#include <stdbool.h>

// 键码定义（对应 PS2 扫描码）
#define AM_KEY_NONE  0
#define AM_KEY_W     0x1D   // 上
#define AM_KEY_A     0x1C   // 左
#define AM_KEY_S     0x1B   // 下
#define AM_KEY_D     0x23   // 右
#define AM_KEY_J     0x3B   // A 按钮
#define AM_KEY_K     0x42   // B 按钮
#define AM_KEY_U     0x3C   // Select
#define AM_KEY_I     0x43   // Start

// AM 数据类型定义
typedef struct {
    int width;
    int height;
} AM_GPU_CONFIG_T;

typedef struct {
    uint32_t us;  // 微秒
} AM_TIMER_UPTIME_T;

typedef struct {
    bool keydown;
    int keycode;
} AM_INPUT_KEYBRD_T;

// AM 接口函数声明
void ioe_init(void);
AM_TIMER_UPTIME_T am_timer_uptime(void);
AM_GPU_CONFIG_T am_gpu_config(void);
AM_INPUT_KEYBRD_T am_input_keybrd(void);
void am_gpu_fbdraw(int x, int y, uint32_t *pixels, int w, int h, bool sync);

#endif
```

### io_read/io_write 宏的实现

AM 使用宏来统一读写接口，通过宏展开映射到具体函数：

```C
// 宏定义（兼容 AM 代码）
#define io_read(dev) _io_read_##dev()
#define io_write(dev, ...) _io_write_##dev(__VA_ARGS__)

// 宏展开映射
#define _io_read_AM_TIMER_UPTIME() am_timer_uptime()
#define _io_read_AM_GPU_CONFIG() am_gpu_config()
#define _io_read_AM_INPUT_KEYBRD() am_input_keybrd()
#define _io_write_AM_GPU_FBDRAW(x, y, p, w, h, s) am_gpu_fbdraw(x, y, p, w, h, s)
```

当 LiteNES 调用 `io_read(AM_TIMER_UPTIME)` 时，预处理器展开为 `am_timer_uptime()`。

---

## 适配层实现详解

### 1. IO 初始化 (ioe_init)

初始化函数负责设置 HDMI 双缓冲：

```C
void ioe_init(void) {
    // 初始化 HDMI 双缓冲
    hdmi_fb_write_base_set(BUFFER_B);
    hdmi_fb_show_base_set(BUFFER_B);
    hdmi_clear(0x0000);  // 黑色

    hdmi_fb_write_base_set(BUFFER_A);
    hdmi_fb_show_base_set(BUFFER_A);
    hdmi_clear(0x0000);

    printf("[AM] IO devices initialized\n");
}
```

### 2. 定时器适配 (am_timer_uptime)

将 xOS 的硬件定时器映射到 AM 接口：

```C
AM_TIMER_UPTIME_T am_timer_uptime(void) {
    AM_TIMER_UPTIME_T ret;
    ret.us = timer_get_uptime_us();  // 调用 xOS 定时器驱动
    return ret;
}
```

### 3. GPU 配置 (am_gpu_config)

返回 NES 的原始分辨率：

```C
AM_GPU_CONFIG_T am_gpu_config(void) {
    AM_GPU_CONFIG_T cfg;
    cfg.width = 256;   // NES 原始宽度
    cfg.height = 240;  // NES 原始高度
    return cfg;
}
```

### 4. GPU 绘制 (am_gpu_fbdraw)

这是最关键的适配函数，需要将 NES 的 32 位 ARGB 像素转换为 HDMI 的 16 位 RGB565 格式：

```C
void am_gpu_fbdraw(int x, int y, uint32_t *pixels, int w, int h, bool sync) {
    volatile uint16_t *fb = hdmi_get_fb_pointer();

    // 边界检查
    if (x < 0 || y < 0 || x + w > 1920 || y + h > 1080) {
        return;
    }

    // 预计算 y 方向偏移（y * 1920 使用移位优化）
    int y_offset = (y << 10) + (y << 9) + (y << 8) + (y << 7);
    volatile uint16_t *fb_row = fb + y_offset + x;
    uint32_t *src = pixels;

    for (int j = 0; j < h; j++) {
        for (int i = 0; i < w; i++) {
            uint32_t argb = src[i];
            // ARGB8888 -> RGB565 转换
            uint16_t r = (argb >> 8) & 0xF800;
            uint16_t g = (argb >> 5) & 0x07E0;
            uint16_t b = (argb >> 3) & 0x001F;
            fb_row[i] = r | g | b;
        }
        src += w;
        fb_row += 1920;
    }

    if (sync) {
        hdmi_swap_buffers();  // 双缓冲切换
    }
}
```

#### 颜色格式转换原理

NES 输出 32 位 ARGB 格式，HDMI 使用 16 位 RGB565：

```
ARGB8888:  AAAA AAAA | RRRR RRRR | GGGG GGGG | BBBB BBBB
                       ↓ >> 8      ↓ >> 5      ↓ >> 3
RGB565:               RRRR R | GGG GGG | BBBB B
```

### 5. 键盘输入适配 (am_input_keybrd)

将 PS2 扫描码转换为 AM 键码：

```C
static int ps2_to_am_keycode(uint8_t scancode) {
    switch (scancode) {
    case 0x1D: return AM_KEY_W;  // W (上)
    case 0x1C: return AM_KEY_A;  // A (左)
    case 0x1B: return AM_KEY_S;  // S (下)
    case 0x23: return AM_KEY_D;  // D (右)
    case 0x3B: return AM_KEY_J;  // J (A 按钮)
    case 0x42: return AM_KEY_K;  // K (B 按钮)
    case 0x3C: return AM_KEY_U;  // U (Select)
    case 0x43: return AM_KEY_I;  // I (Start)
    default:   return -1;
    }
}
```

完整的键盘读取函数：

```C
AM_INPUT_KEYBRD_T am_input_keybrd(void) {
    AM_INPUT_KEYBRD_T ev;
    ev.keydown = false;
    ev.keycode = 0;

    extern int kb_get_scancode(void);
    int scancode = kb_get_scancode();

    if (scancode < 0) {
        return ev;  // 没有新按键
    }

    // 检查 break code 前缀
    if (scancode == 0xF0) {
        is_break_code = true;
        return ev;
    }

    int keycode = ps2_to_am_keycode(scancode);
    if (keycode >= 0) {
        ev.keycode = keycode;
        ev.keydown = !is_break_code;
        key_states[keycode] = ev.keydown ? 1 : 0;
    }

    is_break_code = false;
    return ev;
}
```

---

## NES 手柄输入处理

NES 手柄通过 I/O 端口 `$4016` 读取，采用串行方式逐位读取 8 个按键状态：

```C
// psg.c - 手柄输入处理
static int key_state[256];

// 按键映射：AM 键码 -> NES 按钮
static int MAP[256] = {
    0,          // On/Off
    AM_KEY_J,   // A 按钮
    AM_KEY_K,   // B 按钮
    AM_KEY_U,   // Select
    AM_KEY_I,   // Start
    AM_KEY_W,   // 上
    AM_KEY_S,   // 下
    AM_KEY_A,   // 左
    AM_KEY_D,   // 右
};

// 每帧检测按键状态
void psg_detect_key() {
    while (1) {
        AM_INPUT_KEYBRD_T ev = io_read(AM_INPUT_KEYBRD);
        if (ev.keycode == AM_KEY_NONE) break;
        key_state[ev.keycode] = ev.keydown;
    }
}
```

---

## NES 模拟器主循环

### ROM 加载

LiteNES 支持 iNES 格式的 ROM 文件：

```C
typedef struct {
    char signature[4];      // "NES\x1A"
    byte prg_block_count;   // PRG ROM 块数 (16KB 每块)
    byte chr_block_count;   // CHR ROM 块数 (8KB 每块)
    word rom_type;          // Mapper 类型
    byte reserved[8];
} ines_header;

int fce_load_rom(char *rom) {
    buf = (byte *)rom;
    fce_rom_header = (ines_header *)romread(sizeof(ines_header));

    // 验证 NES 文件签名
    if (memcmp(fce_rom_header->signature, "NES\x1A", 4)) {
        return -1;
    }

    // 加载 PRG ROM 到 CPU 地址空间
    int prg_size = fce_rom_header->prg_block_count * 0x4000;
    byte *blk = romread(prg_size);
    mmc_copy(0x8000, blk, prg_size);

    // 加载 CHR ROM 到 PPU 地址空间
    for (int i = 0; i < fce_rom_header->chr_block_count; i++) {
        byte *blk = romread(0x2000);
        if (i == 0) {
            ppu_copy(0x0000, blk, 0x2000);
        }
    }
    return 0;
}
```

### 帧循环

NES 每帧包含 262 条扫描线：

```C
void fce_run() {
    int nr_draw = 0;
    uint32_t last = uptime_ms();

    while (1) {
        int scanlines = 262;
        while (scanlines-- > 0) {
            ppu_cycle();       // PPU 处理一条扫描线
            psg_detect_key();  // 检测按键
        }

        nr_draw++;
        int upt = uptime_ms();
        if (upt - last > 1000) {
            printf("FPS = %d\n", nr_draw);
            nr_draw = 0;
            last = upt;
        }
    }
}
```

### PPU 扫描线处理

每条扫描线执行 CPU 指令并渲染图形：

```C
void ppu_cycle() {
    cpu_run(256);  // 执行 CPU 指令
    ppu.scanline++;

    // 渲染可见扫描线 (0-239)
    if (ppu.scanline < SCR_H && ppu_shows_background()) {
        ppu_draw_background_scanline(false);
        ppu_draw_background_scanline(true);
    }

    cpu_run(85 - 16);

    if (ppu.scanline < SCR_H && ppu_shows_sprites()) {
        ppu_draw_sprite_scanline();
    }

    cpu_run(16);

    // VBlank 开始 (扫描线 241)
    if (ppu.scanline == 241) {
        ppu_set_in_vblank(true);
        cpu_interrupt();  // 触发 NMI
    }
    // 帧结束 (扫描线 262)
    else if (ppu.scanline == 262) {
        ppu.scanline = -1;
        ppu_set_in_vblank(false);
        fce_update_screen();  // 刷新屏幕
    }
}
```

---

## 屏幕刷新

每帧结束时将画布内容绘制到屏幕：

```C
void fce_update_screen() {
    AM_GPU_CONFIG_T cfg = io_read(AM_GPU_CONFIG);
    int xpad = (cfg.width - SCR_W) / 2;
    int ypad = (cfg.height - SCR_H) / 2;

    // 绘制到屏幕中央
    io_write(AM_GPU_FBDRAW, xpad, ypad, canvas, SCR_W, SCR_H, true);

    // 清空画布为背景色
    int idx = ppu_ram_read(0x3F00);
    uint32_t bgc = palette[idx];
    for (int i = 0; i < SCR_W * SCR_H; i++)
        canvas[i] = bgc;
}
```

---

## Shell 集成

在 xOS Shell 中添加 `mario` 命令：

```C
int cmd_mario(int argc, char *argv[]) {
    extern char rom_mario_nes[];
    extern void ioe_init(void);
    extern int fce_load_rom(char *rom);
    extern void fce_init(void);
    extern void fce_run(void);

    printf("Super Mario Bros - NES Emulator\n");
    printf("Controls: W/A/S/D=Move, J=Jump, K=Run\n");

    ioe_init();

    if (fce_load_rom((void *)rom_mario_nes) < 0) {
        printf("ERROR: Failed to load ROM!\n");
        return -1;
    }

    fce_init();
    fce_run();
    return 0;
}
```

---

## 总结

### 移植要点

将基于 AM 的应用移植到新平台，核心步骤：

1. **定义 AM 接口头文件** (`am.h`)
2. **实现适配层** (`am_adapter.c`)
3. **映射底层驱动** (定时器、显示、输入)

### AM 适配层映射表

| AM 接口 | xOS 实现 |
|---------|----------|
| `ioe_init()` | HDMI 双缓冲初始化 |
| `AM_TIMER_UPTIME` | `timer_get_uptime_us()` |
| `AM_GPU_CONFIG` | 返回 256x240 |
| `AM_GPU_FBDRAW` | ARGB→RGB565 + HDMI 写入 |
| `AM_INPUT_KEYBRD` | PS2 扫描码转换 |

### 文件结构

```
src/litenes/
├── am_adapter.c   # AM 适配层
├── fce.c          # 主循环
├── cpu.c          # 6502 CPU 模拟
├── ppu.c          # 图形处理
├── psg.c          # 手柄输入
├── memory.c       # 内存映射
└── mario-rom.c    # ROM 数据

include/litenes/
├── am.h           # AM 接口定义
├── klib.h         # 标准库映射
└── ...
```

通过 AM 抽象层，LiteNES 无需修改核心代码即可在 xOS 上运行，体现了硬件抽象的价值。