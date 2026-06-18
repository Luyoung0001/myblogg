---
title: "毕业设计记录（12）：Terminal 与 Shell 设计"
date: 2026-01-26 10:03:13
tags:
  - "graduation thesis"
  - "terminal"
  - "shell"
categories:
  - "Thesis Project"
mermaid: true
---

## 前言

上节讨论了 HDMI 控制器的双 buffer 控制机制以及双 linebuffer 交换机制。本节基于 bufferframe 设计显示系统：将图像以及文字显示到 hdmi 显示的组件。

## shell

之前有 uart 显示屏的组件，它的原理就是对底层 putchar 进行重定向，这样只要 printf() 就能在 uart 显示屏上打印了：

```C
int putchar(int c) {
    int tid = get_current_task();

#ifdef SIMULATION
    bsp_uart_putc(0, (char)c);
    return c;
#endif

    /* Output to HDMI if enabled */
    // this process may be slow
    if (get_output_target() & OUTPUT_HDMI) {
        if (tid >= 0) {
            terminal_putchar(tid, (char)c);
        } else {
            // No task context (early boot) - output directly to HDMI
            terminal_putc((char)c);
        }
    }
    /* Output to UART if enabled */
    if (get_output_target() & OUTPUT_UART) {
        bsp_uart_putc(0, (char)c);
    }

    return c;
}
```

同样，我们只要设计 `terminal_putc()` 就能让它屏幕上显示：

```C

void terminal_putc(char c) {
    if (c == '\n') {
        terminal_newline();
        return;
    }
    if (c == '\r') {
        cursor_x = 0;
        return;
    }
    if (c == '\b') {
        if (cursor_x > 0) {
            cursor_x--;
            screen_buffer[cursor_y][cursor_x].ch = ' ';
            screen_buffer[cursor_y][cursor_x].fg_color = current_fg_color;
            screen_buffer[cursor_y][cursor_x].bg_color = current_bg_color;
            terminal_render_char(cursor_x, cursor_y);
        }
        return;
    }
    if (c == '\t') {
        int spaces = 4 - (cursor_x % 4);
        for (int i = 0; i < spaces; i++) {
            terminal_putc(' ');
        }
        return;
    }

    /* Normal character */
    if (c >= 32 && c <= 126) {
        screen_buffer[cursor_y][cursor_x].ch = c;
        screen_buffer[cursor_y][cursor_x].fg_color = current_fg_color;
        screen_buffer[cursor_y][cursor_x].bg_color = current_bg_color;
        terminal_render_char(cursor_x, cursor_y);
        cursor_x++;
        if (cursor_x >= TERMINAL_COLS) {
            terminal_newline();
        }
    }
}
```

我们可以选一个大一点的 framebuffer，这样可以写入更多的信息。这里面有一个细节设计：screenbuffer 的作用是暂存要打印的文字，渲染是 `terminal_render_char(cursor_x, cursor_y)`，它会从字符缓冲 screenbuffer 根据坐标来选择要打印的字符：

```C
void terminal_render_char(int x, int y) {
    if (y >= TERMINAL_TOTAL_ROWS) {
        return;
    }
    terminal_cell_t *cell = &screen_buffer[y][x];
    int pixel_x = x * TERMINAL_FONT_SIZE;
    int pixel_y = y * TERMINAL_FONT_SIZE;
    hdmi_draw_char(pixel_x, pixel_y, cell->ch, cell->fg_color, cell->bg_color);
}
```
`hdmi_draw_char()` 还不是底层，它会根据字符设计大小、具体字符来让 `hdmi_draw_pixel()` 完成真正的显存绘制。这样 hdmi 控制器就会扫描到 `hdmi_draw_pixel()` 的写的信息，写的单位是像素，这里为了节省 fpga 资源以及宽带，每一个像素采用 rgb565：

```C
// all the character drawing functions draw on buffer S directly
// we don't want to our game drawing functions to be affected by character drawing
void hdmi_draw_pixel(int x, int y, uint16_t color) {
    if (x >= 0 && x < HDMI_WIDTH && y >= 0 && y < FRAMEBUF_S_NUM_ROWS) {
        // 不用切换，直接在 buffer S 上绘制
        volatile uint16_t *hdmi_fb_ptr_S = (volatile uint16_t *)HDMI_FB_BASE_S;
        hdmi_fb_ptr_S[y * HDMI_WIDTH + x] = color;
    }
}
```
这样，我们就能在 HDMI 看到刚刚写的字符了。每一个 `printf()` 最终打印的目标就是 `HDMI_FB_BASE_S`，但是前面说了这个 framebuffer 比较大，因为我不想让它打印到屏幕底部后等它清屏，这样太慢。等到坐标更新到了屏幕之外，最简单的办法就是重新设置 HDMI_FB_BASE_S，在它的基础上增加一行字符的便宜，这样就实现了滚动效果：

```C
void terminal_scroll_up(void) {
    display_start_row++;

    uint32_t offset = display_start_row * TERMINAL_FONT_SIZE * HDMI_WIDTH * 2;
    hdmi_set_show_addr(offset + HDMI_FB_BASE_S);

    /* 清除新出现的底部行 */
    int new_row = cursor_y;
    int pixel_row_start = new_row * TERMINAL_FONT_SIZE;
    int pixel_row_end = pixel_row_start + TERMINAL_FONT_SIZE;
    hdmi_clear_line(pixel_row_start, pixel_row_end, current_bg_color);
}
```
其中，`hdmi_set_show_addr()` 就是在重新设置 base，显示新的行出来后用 `hdmi_clear_line()` 清理下，这都是细节。

有一个问题是，滚动到 framebufferS 的最后一行了怎么办？目前的思路是直接回到 `HDMI_FB_BASE_S`，但是这里面的东西是脏的，需要擦拭，还是避免不了需要等。这个问题很好办，因为我的 xos 支持并发，我只需要在 shell 中输入 hdmigc，就会有专门的程序去擦除已经滚动上去的 buffer。问题是擦完了怎么办，还要占着时间片？小问题，实现软件中断就好：

```C
int cmd_hdmi_buffer_gc(int argc, char *argv[]) {
         &display_start_row, display_start_row);
    while (1) {
        if (shell_gc_pointer < display_start_row || shell_gc_pointer > display_start_row) {
            // clear one line
            int pixcel_row_start = shell_gc_pointer * TERMINAL_FONT_SIZE;
            int pixcel_row_end = pixcel_row_start + TERMINAL_FONT_SIZE;
            hdmi_clear_line(pixcel_row_start, pixcel_row_end, current_bg_color);
            shell_gc_pointer = (shell_gc_pointer + 1) % TERMINAL_TOTAL_ROWS;
        } else {
            // by calling task_yield(), we can yield cpu to other tasks
            task_yield();
            // by saving easy context, we can yield cpu to other tasks
            // there is a problem here, unsolved YET.
            // task_yield_simple();
        }
    }
    return 0;
```

这样，就不会让一个空任务白白浪费处理器。

## 图像显示

图像显示一定是要多 framebuffer 的（目前是双 buffer）。当显示 A 的时候，我们让 CPU 去绘制 B，绘制好了或者 A 显示结束了我们想切换了直接 `swap()` 就好，`swap()` 会将当前显示的 buffer 和绘制的 buffer 切换：

```C
// 交换前后台 buffer（硬件零拷贝切换）
void hdmi_swap_buffers(void) {
    // 等待 V-Sync，避免画面撕裂
    hdmi_wait_vsync();
    volatile uint32_t *hdmi_fb_addr_reg = (volatile uint32_t *)HDMI_FB_ADDR_REG;


    // 切换显示的 buffer
    if (current_display_buffer == 0) {
        // 当前显示 Buffer A，切换到 Buffer B
        *hdmi_fb_addr_reg = HDMI_FB_BASE_B;
        current_display_buffer = 1;
        current_draw_buffer = 0; // 下次绘制到 A
    } else {
        // 当前显示 Buffer B，切换到 Buffer A
        *hdmi_fb_addr_reg = HDMI_FB_BASE_A;
        current_display_buffer = 0;
        current_draw_buffer = 1; // 下次绘制到 B
    }
    hdmi_fb_ptr = (volatile uint16_t *)hdmi_get_back_buffer();
}
```

不管是游戏的设计，还是 ppt 的播放，都需要实现这样的 AB 切换，这个切换不需要任何时间复杂度，都是常数级别的，而且响应时间极短。

这样我们的termimal 就实现了，实现了终端在 HDMI 的显示，滚动等。




