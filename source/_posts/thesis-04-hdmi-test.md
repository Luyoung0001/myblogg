---
title: "毕业设计记录（4）：UART 与 HDMI 测试"
date: 2026-01-13 10:03:13
tags:
  - "graduation thesis"
  - "HDMI"
  - "UART"
categories:
  - "Thesis Project"
---

## uart
给系统接一个 uart 串口屏，也可以运行一些简单的程序，比如简单的命令行程序。但是肯定没有 HDMI 显示方便，上节已经实现了 uart 输出一个简单的 xOS 终端，uart 的上限也就如此了。

## HDMI

当需要在 HDMI 显示器上显示字符时：

- 软件渲染：CPU/软件（如 xOS 的 hdmi_putc）根据字符 ASCII 码从字库中查找点阵数据，计算目标位置，并将对应的像素颜色值（如 RGB565 格式）写入帧缓冲区（FB）的特定内存位置。

- 主动读取：HDMI 控制器作为 AXI 总线主设备，在内部视频时序发生器的精确驱动下，主动地、以突发方式不断从 FB 的物理基地址开始，按行、按块地读取像素数据流。

- TMDS 编码：控制器内部的 TMDS 编码器将从 FB 读取的像素数据流（可能经过色彩空间转换）编码成符合 HDMI 标准的 TMDS 字符流（包含像素数据和控制信号）。

- 差分输出：编码器输出的并行 TMDS 数据流由差分驱动器转换成 3 路数据差分信号对和1路时钟差分信号对。

- 物理传输：这 4 路高速差分信号通过HDMI物理连接器（PHY） 输出到 HDMI 线缆。

- 显示呈现：显示器接收这些差分信号，解码恢复像素数据和控制信号，驱动显示面板最终将字符图像呈现在屏幕上。


## 仿真

思路是，从 soc_top 拉出 hdmi_master 的数据，在每一个指令周期检查相关 axi 信号，并决定是否将这个数据拉取到 simu 中的 framebuffer 中。等 framebuffer 中的数据攒够了，就可以通过 sdl2 库输出到窗口，这样就完成了 hdmi 显示器的模拟。

## 用户程序

用户程序很简单，通过 `hdmi_init()` 来说明：

```C
int main(void) {
    // 初始化 UART
    bsp_uart_init(0, BAUDRATE);

    myprintf("\n");
    myprintf("========================================\n");
    myprintf("  LoongArch32R HDMI Display Test\n");
    myprintf("========================================\n");
    myprintf("\n");

    // 初始化 HDMI
    myprintf("Initializing HDMI...\n");
    hdmi_init();
    myprintf("HDMI initialized!\n\n");
    // ...
}
```
`hdmi_init()` 本质上就做了一件事：往 framebuffer 中写 0（黑色）。

在 CPU reset 释放后（锁相环锁定时），hdmi master 就不断的通过突发的方式读取 ddr 中的 framebuffer 了。因为我设置了一个 master 仲裁器，多个 master 请求 axi 总线，最终通过一些策略（绝不能把某些 master 通道饿死）决胜出一个 master。大多数时候，cpu master 是不会访问 ddr 的，因此通道被 hdmi master 专用。但是一旦 CPU 有 axi 请求，hdmi master 应该立即释放。换句话说，cpu master 的优先级最高。

之后决胜出的 master 会经过 axi_slave_mux 路由，将请求可能转发到：
- ddr
- apb
- confreg
- brom

对于 hdmi 就会转发到 ddr 中。

仿真的思路就是，通过 soc_top 的 debug 接口，抓取 hdmi master 已经读取到的 ddr 数据：

```Verilog
    // for HDMI trace (HDMI AXI interface)
    ,output wire [31:0] debug_hdmi_araddr  // HDMI 读地址
    ,output wire        debug_hdmi_arvalid // HDMI 读地址有效
    ,output wire        debug_hdmi_arready // HDMI 读地址就绪
    ,output wire [31:0] debug_hdmi_rdata   // HDMI 读数据
    ,output wire        debug_hdmi_rvalid  // HDMI 读数据有效
    ,output wire        debug_hdmi_rready  // HDMI 读数据就绪
```

仿真器：

```Cpp

    // HDMI data capture: 正确处理 AXI burst 传输
    // 1. 在地址握手时，记录 burst 起始地址
    if (hdmi_sim && top->debug_hdmi_arvalid && top->debug_hdmi_arready) {
        hdmi_burst_addr = top->debug_hdmi_araddr;
        hdmi_beat_count = 0;
    }

    // 2. 在每个有效数据周期（rvalid && rready）捕获数据
    // AXI burst 传输中，所有16个 beat 的 rvalid 都为1，需要全部捕获
    if (hdmi_sim && top->debug_hdmi_rvalid && top->debug_hdmi_rready) {
        uint32_t hdmi_data = top->debug_hdmi_rdata;
        // 计算当前 beat 的地址（每个 beat 4 字节）
        uint32_t current_addr = hdmi_burst_addr + hdmi_beat_count * 4;

        // 只捕获显存地址范围内的数据
        if (HDMISimulator::is_hdmi_fb_addr(current_addr)) {
            hdmi_sim->capture_read(current_addr, hdmi_data);
        }

        // 递增 beat 计数
        hdmi_beat_count++;
    }
```

再把抓来的数据收集到仿真环境中的缓存中，就可以利用 SDL2 等来做呈现，这样 hdmi 仿真器就成功了：

```Cpp

    // 分配临时 RGB8888 缓冲区
    uint8_t *rgb8888_buffer = new uint8_t[HDMI_WIDTH * HDMI_HEIGHT * 4];

    // 转换 RGB565 -> RGB888
    // SDL_PIXELFORMAT_RGBA8888: R-G-B-A 字节顺序
    for (uint32_t i = 0; i < HDMI_WIDTH * HDMI_HEIGHT; i++) {
        uint8_t r, g, b;
        rgb565_to_rgb888(framebuffer[i], r, g, b);
        rgb8888_buffer[i * 4 + 0] = 255; // Alpha
        rgb8888_buffer[i * 4 + 1] = b;   // B
        rgb8888_buffer[i * 4 + 2] = g;   // G
        rgb8888_buffer[i * 4 + 3] = r;   // R
    }

    // 更新纹理
    SDL_UpdateTexture(texture, nullptr, rgb8888_buffer, HDMI_WIDTH * 4);

    // 渲染到窗口
    SDL_RenderClear(renderer);
    SDL_RenderCopy(renderer, texture, nullptr, nullptr);
    SDL_RenderPresent(renderer);

    delete[] rgb8888_buffer;

    frame_count++;
    if (frame_count % 60 == 0) {
        printf("[HDMI] Frame %u rendered\n", frame_count);
    }
```

### hdmi 驱动

Xilinx 7系列（Artix-7, Kintex-7, Virtex-7）基本单元：

- 每个 BRAM Primitive = 36Kb
- 36Kb = 36,864 bits = 4,608 bytes = 4.5 KB

可配置方式：
1. RAMB36（36Kb 整块）
- 数据宽度可配置：1, 2, 4, 9, 18, 36, 72 位
- 深度相应调整
   - 例如：4K × 9bit, 2K × 18bit, 1K × 36bit

2. RAMB18（18Kb 半块）
- 一个 RAMB36 可以分成 2 个 RAMB18
- 每个 RAMB18 = 18Kb = 2.25 KB

为了节省资源，在硬件中使用 bram 来做行缓冲。由于 1080P 每一行需要 1920 个像素，如果使用 RGB565，这样 1920 个像素总共会使用 960 * 32b < 1024 * 36b，刚好用不到一个 bram 资源。于是 RGB565 就是非常好的选择：

```C
// RGB565 颜色定义
#define RGB565(r, g, b) ((((r) >> 3) << 11) | (((g) >> 2) << 5) | ((b) >> 3))

// 常用颜色
#define COLOR_BLACK   RGB565(0, 0, 0)
#define COLOR_WHITE   RGB565(255, 255, 255)
#define COLOR_RED     RGB565(255, 0, 0)
#define COLOR_GREEN   RGB565(0, 255, 0)
#define COLOR_BLUE    RGB565(0, 0, 255)
#define COLOR_YELLOW  RGB565(255, 255, 0)
#define COLOR_CYAN    RGB565(0, 255, 255)
#define COLOR_MAGENTA RGB565(255, 0, 255)
#define COLOR_GRAY    RGB565(128, 128, 128)
```

如果想要绘制一个字符，只需要：

```C
static volatile uint16_t* fb = (volatile uint16_t*)HDMI_FB_BASE_A;
void hdmi_draw_pixel(int x, int y, uint16_t color) {
    if (x >= 0 && x < HDMI_WIDTH && y >= 0 && y < HDMI_HEIGHT) {
        fb[y * HDMI_WIDTH + x] = color;
    }
}
void hdmi_draw_char(int x, int y, char c, uint16_t fg_color, uint16_t bg_color) {
    if (c < 32 || c > 126) c = ' ';

    const uint8_t* glyph = font_8x8[c - 32];

    for (int row = 0; row < 8; row++) {
        uint8_t line = glyph[row];
        for (int col = 0; col < 8; col++) {
            uint16_t color = (line & (1 << (7 - col))) ? fg_color : bg_color;
            hdmi_draw_pixel(x + col, y + row, color);
        }
    }
}
```

这里面有一个 font_8x8 的概念，这是 8x8 ASCII 的字体，对于每一个字符，都对应于这个字符的绘制数据，比如对于 '$'：
```C
    // 0x24 '$'
    {0x0C, 0x3E, 0x03, 0x1E, 0x30, 0x1F, 0x0C, 0x00},
```

其它图形的绘制也是类似。

这样就完成了从应用到仿真，再到物理设计。





