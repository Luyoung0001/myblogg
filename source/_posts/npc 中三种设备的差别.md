---
title: npc 中三种设备的区别
date: 2024-10-16 16:20:40
tags:
    - ysyx
    - pa2
    - 设备
categories:
    - ysyx
    - npc
---

<!--more-->

## 前言
在把 PA2.3 做完之后，顺手把 ysyx 的 npc 中的设备模仿 nemu 也实现了一下，分别是串口、时钟、vga，本文讨论一下这三种设备的区别。事实上，这三种设备循序渐进，虽然是三种设备，但其实在 CPU 眼里，它就是同一种东西，都是读写内存。

## 设备
在 nemu 的模型中，程序访问设备的时候，会提供统一的接口，比如 io_read、io_write，比如：

```C
#include <amtest.h>

void rtc_test() {
    AM_TIMER_RTC_T rtc;
    int sec = 1;
    while (1) {
        while (io_read(AM_TIMER_UPTIME).us / 1000000 < sec)
            ;
        rtc = io_read(AM_TIMER_RTC);
        printf("%d-%d-%d %02d:%02d:%02d GMT (", rtc.year, rtc.month, rtc.day,
               rtc.hour, rtc.minute, rtc.second);
        if (sec == 1) {
            printf("%d second).\n", sec);
        } else {
            printf("%d seconds).\n", sec);
        }
        sec++;
    }
}
```
这里读取了 AM_TIMER_UPTIME、AM_TIMER_RTC 中的元素，这是抽象寄存器。io_read 是运行时环境提供的接口，事实上，它封装了 ioe:
```C
#define io_read(reg) \
  ({ reg##_T __io_param; \
    ioe_read(reg, &__io_param); \
    __io_param; })

#define io_write(reg, ...) \
  ({ reg##_T __io_param = (reg##_T) { __VA_ARGS__ }; \
    ioe_write(reg, &__io_param); })
```

ioe 是一种扩展的 io，它提供了一组架构无关性的 API：

```C
bool ioe_init();
void ioe_read(int reg, void *buf);
void ioe_write(int reg, void *buf);
```

第一个 API 用于进行 IOE 相关的初始化操作。后两个 API 分别用于从编号为 reg 的寄存器中读出内容到缓冲区 buf 中， 以及往编号为reg寄存器中写入缓冲区buf中的内容。 在IOE中， 我们希望采用一种架构无关的"抽象寄存器"， 这个reg其实是一个功能编号。 我们约定在不同的架构中，同一个功能编号的含义也是相同的， 这样就实现了设备寄存器的抽象。

ioe_read() 和ioe_write() 都是通过抽象寄存器的编号索引到一个处理函数, 然后调用它. 处理函数的具体功能和寄存器编号相关。

## 串口

串口怎么实现？我觉得还是注册一个地址，然后往这个地址中写入数据就行了。事实上，nemu 的串口就是这样实现的：

比如，我们要实现 putch 功能，我们的愿望是一旦应用程序调用了 putch，比如 putch('A')，那么运行时环境就会打印出来一个 'A'。

串口是一种设备，putch 会将'A'写在某个地址上，而 nemu 中有一种底层的机制：访问内存的 handler 检测到这个写入地址是设备而不是内存之后，就会调用一个函数，让它去处理，比如显示这个字符。简单吧

运行时环境的接口：

```C
static inline void outb(uintptr_t addr, uint8_t  data) { *(volatile uint8_t  *)addr = data; }
#define DEVICE_BASE 0xa0000000
#define MMIO_BASE   0xa0000000

#define SERIAL_PORT     (DEVICE_BASE + 0x00003f8)
void putch(char ch) {
  outb(SERIAL_PORT, ch);
}

```

这里的核心就是 outb 的实现，它是一个访存操作，而且访问的地址是一个特殊的地址。这个访存操作会被编译器编译成sbu 指令：
```asm
80000088 <putch>:
80000088:	a00007b7          	lui	a5,0xa0000
8000008c:	3ea78c23          	sb	a0,1016(a5) # a00003f8 <_end+0x1fff73f8>
80000090:	00008067          	ret
```

之后呢？那当然是把二进制文件丢到 CPU 上执行了，当 cpu 执行到：`sb	a0,1016(a5)`的时候，这时候访存模块就被激活了：

```C
extern "C" void memo_access_write(int waddr, int wdata, char wmask) {
    // 总是往地址为`waddr & ~0x3u`的4字节按写掩码`wmask`写入`wdata`
    // `wmask`中每比特表示`wdata`中1个字节的掩码,
    // 如`wmask = 0x3`代表只写入最低2个字节, 内存中的其它字节保持不变

    switch (wmask) {
        case 0x1:
            paddr_write(waddr, 1, wdata);
            break;
        case 0x3:
            paddr_write(waddr, 2, wdata);
            break;
        case 0xf:
            paddr_write(waddr, 4, wdata);
            break;
        default:
            paddr_write(waddr, 4, wdata);
    }
}
```
这是 verilog 中的 DPI-C 函数，它会调用 `paddr_write(waddr, 1, wdata)`，之后进入：

```C
void paddr_write(paddr_t addr, int len, word_t data) {
    IFDEF(CONFIG_MTRACE, display_memory_write(addr, len, data));
    if (likely(in_pmem(addr))) {
        pmem_write(addr, len, data);
        return;
    }
    IFDEF(CONFIG_DEVICE, mmio_write(addr, len, data); return );
    out_of_bound(addr);
}
```

这时候，就会检查地址，如果不是访存，那么就会进行到`IFDEF(CONFIG_DEVICE, mmio_write(addr, len, data); return )`:

```C
void mmio_write(paddr_t addr, int len, word_t data) {
    map_write(addr, len, data, fetch_mmio_map(addr));
}
```
这个函数会根据地址，获取相应的注册了的 mmio_map，mmio_map 是系统启动的时候注册的内存映射：

```C
void init_serial() {
  serial_base = new_space(8);
  add_mmio_map("serial", CONFIG_SERIAL_MMIO, serial_base, 8, serial_io_handler);
}
```

拿到 mmip_map 之后，就会进入：

```C
void map_write(paddr_t addr, int len, word_t data, IOMap* map) {
    assert(len >= 1 && len <= 8);
    check_bound(map, addr);
    paddr_t offset = addr - map->low;
    host_write(map->space + offset, len, data);
    invoke_callback(map->callback, offset, len, true);
    IFDEF(CONFIG_DTRACE, trace_dwrite(addr, len, data, map););
}
```

分别做一些检查，然后计算 offset，然后准确地将这个数据写入到目标地址（准备数据）。然后激活回调函数：
```C
static void serial_putc(char ch) {
  MUXDEF(CONFIG_TARGET_AM, putch(ch), putc(ch, stderr));
}

static void serial_io_handler(uint32_t offset, int len, bool is_write) {
  assert(len == 1);
  switch (offset) {
    case CH_OFFSET:
      if (is_write) serial_putc(serial_base[0]);
      else printf("do not support read");
      break;
    default: printf("do not support offset = %d", offset);
  }
}
```
可以看到，回调函数其实是在实现功能。

以上就是串口的整个过程。

## 时钟

事实上，时钟和串口很接近，但是也有一些不同。

首先是注册 mmio_map：

```C
static void rtc_io_handler(uint32_t offset, int len, bool is_write) {
    assert(offset == 0 || offset == 4);
    if (!is_write && offset == 4) { // 确保每一次读取时间(64位)，只更新一次寄存器
        uint64_t us = get_time();
        rtc_port_base[0] = (uint32_t)us;
        rtc_port_base[1] = us >> 32;
    }
}

void init_timer() {
    rtc_port_base = (uint32_t*)new_space(8);
    add_mmio_map("rtc", CONFIG_RTC_MMIO, rtc_port_base, 8, rtc_io_handler);
}
```

这里的回调函数是在将时间放到两个字，也就是准备数据。

当应用程序调用时间接口：

```C
#include <amtest.h>

void rtc_test() {
    AM_TIMER_RTC_T rtc;
    int sec = 1;
    while (1) {
        while (io_read(AM_TIMER_UPTIME).us / 1000000 < sec)
            ;
        rtc = io_read(AM_TIMER_RTC);
        printf("%d-%d-%d %02d:%02d:%02d GMT (", rtc.year, rtc.month, rtc.day,
               rtc.hour, rtc.minute, rtc.second);
        if (sec == 1) {
            printf("%d second).\n", sec);
        } else {
            printf("%d seconds).\n", sec);
        }
        sec++;
    }
}

```

实际上，在调用这个接口：

```C
static inline uint32_t inl(uintptr_t addr) {
    return *(volatile uint32_t*)addr;
}

#define DEVICE_BASE 0xa0000000
#define RTC_ADDR (DEVICE_BASE + 0x0000048)
void __am_timer_uptime(AM_TIMER_UPTIME_T* uptime) {
    uint32_t high = inl(RTC_ADDR + 4);
    uint32_t low = inl(RTC_ADDR);
    uptime->us = (uint64_t)low + (((uint64_t)high) << 32);
}

void __am_timer_rtc(AM_TIMER_RTC_T* rtc) {
    rtc->second = 0;
    rtc->minute = 0;
    rtc->hour = 0;
    rtc->day = 0;
    rtc->month = 0;
    rtc->year = 1900;
}
```
可以看到，它和串口几乎一样，都是在读取数据。这个读取会被翻译成汇编指令：

```asm
800013b4 <__am_timer_uptime>:
800013b4:	a00007b7          	lui	a5,0xa0000
800013b8:	04c7a703          	lw	a4,76(a5) # a000004c <_end+0x1ff6304c>
800013bc:	0487a783          	lw	a5,72(a5)
800013c0:	00e52223          	sw	a4,4(a0)
800013c4:	00f52023          	sw	a5,0(a0)
800013c8:	00008067          	ret
```

之后，会被CPU执行，然后读取：

```C
word_t mmio_read(paddr_t addr, int len) {
    return map_read(addr, len, fetch_mmio_map(addr));
}
```

同样，这里也会根据地址获取 mmio_map，之后就进入了 map_read:

```C
word_t map_read(paddr_t addr, int len, IOMap* map) {
    assert(len >= 1 && len <= 8);
    check_bound(map, addr);
    paddr_t offset = addr - map->low;
    invoke_callback(map->callback, offset, len, false);  // prepare data to read
    word_t ret = host_read(map->space + offset, len);
    IFDEF(CONFIG_DTRACE, trace_dread(addr, len, map));
    return ret;
}

```

同样就是安全检查，然后这里首先是激活回调函数：因为读取之前，首先得准备数据：

```C
static void rtc_io_handler(uint32_t offset, int len, bool is_write) {
    assert(offset == 0 || offset == 4);
    if (!is_write && offset == 4) { // 确保每一次读取时间(64位)，只更新一次寄存器
        uint64_t us = get_time();
        rtc_port_base[0] = (uint32_t)us;
        rtc_port_base[1] = us >> 32;
    }
}
```

这路需要注意的是，只有当 offset == 4的时候，才会更新寄存器。也就是当：

```C
void __am_timer_uptime(AM_TIMER_UPTIME_T* uptime) {
    uint32_t high = inl(RTC_ADDR + 4);
    uint32_t low = inl(RTC_ADDR);
    uptime->us = (uint64_t)low + (((uint64_t)high) << 32);
}
```
第一次获取 high 的时候 更新，获取 low 的时候，不会更新时间寄存器。

当时间准备好后，就可以读取以及返回了：

```C
word_t ret = host_read(map->space + offset, len);
```

这样就完成了时钟。

## VGA

VGA 前两个设备很不一样，因为它没有回调函数。这是为什么呢？


```C
...

static inline void update_screen() {
    SDL_UpdateTexture(texture, NULL, vmem, SCREEN_W * sizeof(uint32_t));
    SDL_RenderClear(renderer);
    SDL_RenderCopy(renderer, texture, NULL, NULL);
    SDL_RenderPresent(renderer);
}

void vga_update_screen() {
    uint32_t sync = vgactl_port_base[1];
    if (sync) {
        update_screen();
        vgactl_port_base[1] = 0;
    }
}

void init_vga() {
    vgactl_port_base = (uint32_t*)new_space(8);
    vgactl_port_base[0] = (screen_width() << 16) | screen_height();

    add_mmio_map("vgactl", CONFIG_VGA_CTL_MMIO, vgactl_port_base, 8, NULL);

    vmem = new_space(screen_size());
    add_mmio_map("vmem", CONFIG_FB_ADDR, vmem, screen_size(), NULL);
    IFDEF(CONFIG_VGA_SHOW_SCREEN, init_screen());
    IFDEF(CONFIG_VGA_SHOW_SCREEN, memset(vmem, 0, screen_size()));
}
```

因为，视频显示并不是说需要一个特定的事件（比如读写某些特定地址）去触发，而是需要每时每刻都将画面更新一下。那我们应该选用什么样子的事件呢？没错就是 CPU 每执行一个指令，我们就刷新：

```C
// 执行一步
void stepi() {
    step();
    IFDEF(CONFIG_DIFFTEST,
          difftest_step(top->rootp->Top__DOT__core__DOT__pc_reg, 0));
    IFDEF(CONFIG_DEVICE, device_update());
}
```

device_update():
```C
void device_update() {
    static uint64_t last = 0;
    uint64_t now = get_time();
    if (now - last < 1000000 / TIMER_HZ) {
        return;
    }
    last = now;

    IFDEF(CONFIG_HAS_VGA, vga_update_screen());

#ifndef CONFIG_TARGET_AM
    SDL_Event event;
    while (SDL_PollEvent(&event)) {
        switch (event.type) {
            case SDL_QUIT:
                npc_state.state = NPC_QUIT;
                break;
#ifdef CONFIG_HAS_KEYBOARD
            // If a key was pressed
            case SDL_KEYDOWN:
            case SDL_KEYUP: {
                uint8_t k = event.key.keysym.scancode;
                bool is_keydown = (event.key.type == SDL_KEYDOWN);
                send_key(k, is_keydown);
                break;
            }
#endif
            default:
                break;
        }
    }
#endif
}
```

至于访问设备，也就是写（没错，只有写）：

```C
void __am_gpu_fbdraw(AM_GPU_FBDRAW_T* ctl) {
    int x = ctl->x, y = ctl->y, w = ctl->w, h = ctl->h;
    if (!ctl->sync && (w == 0 || h == 0))
        return;
    uint32_t* pixels = ctl->pixels;
    uint32_t* fb = (uint32_t*)(uintptr_t)FB_ADDR;
    uint32_t screen_w = inl(VGACTL_ADDR) >> 16;

    for (int i = y; i < y + h; i++) {
        for (int j = x; j < x + w; j++) {
            fb[screen_w * i + j] = pixels[w * (i - y) + (j - x)];
        }
    }
    if (ctl->sync) {
        outl(SYNC_ADDR, 1);
    }
}
```

其它的就没什么区别了：
```C
void map_write(paddr_t addr, int len, word_t data, IOMap* map) {
    assert(len >= 1 && len <= 8);
    check_bound(map, addr);
    paddr_t offset = addr - map->low;
    host_write(map->space + offset, len, data);
    invoke_callback(map->callback, offset, len, true);
    IFDEF(CONFIG_DTRACE, trace_dwrite(addr, len, data, map););
}
```

这里写完之后，就结束了。（激活回调函数不起作用）。

## 总结
这篇文章主要讨论了在 **NEMU** 和 **ysyx** 项目中的三种设备（**串口**、**时钟**、**VGA**）的实现和它们的区别。文章以 **PA2.3** 为背景，模仿 **NEMU** 实现了这三种设备，文章按设备类型进行了详细的实现分析，并对每个设备的访问、触发机制进行了说明。

1. **前言**：
   - 在 NEMU 中，设备访问是通过统一的接口 `io_read` 和 `io_write` 实现的。文章讨论了通过这些接口实现的三种设备（串口、时钟、VGA），它们本质上在 CPU 眼里是读写内存的操作。

2. **串口设备**：
   - 串口的实现通过将字符写入特定的内存地址实现，NEMU 中通过 `outb` 操作将字符输出到设备地址。这个地址被访问时，会触发一系列的访存操作，最终通过回调函数将字符打印到终端。
   - 通过 `paddr_write`，内存访问会检测是否是设备访问，然后调用 `serial_io_handler` 处理写入并完成输出。

3. **时钟设备**：
   - 时钟和串口的工作原理类似，但时钟设备通过 `io_read` 读取当前时间。每次读取时会触发相应的回调函数 `rtc_io_handler`，在读取前更新系统时间寄存器。
   - 时钟的读取操作首先通过汇编指令执行访存操作，然后调用 `mmio_read` 来获取时间数据。

4. **VGA 设备**：
   - VGA 与前两个设备有所不同，它没有依赖回调函数，而是通过周期性的屏幕刷新机制实现。
   - 文章解释了 `vga_update_screen` 函数是如何通过定时器定期刷新屏幕，屏幕内容的更新通过写入帧缓冲区 (`vmem`) 完成。
   - 每当有新的图形数据写入时，VGA 控制器会将数据写入指定地址，并通过 SDL 进行屏幕刷新。

5. **设备间的区别**：
   - **串口**：依赖回调函数，写入地址时触发输出操作。
   - **时钟**：在读取时依赖回调函数，读取前准备数据。
   - **VGA**：不依赖回调函数，而是周期性刷新屏幕。

### 结论：
文章通过分析串口、时钟和 VGA 设备的实现，展示了不同设备在硬件模拟器中的处理机制。虽然这三种设备在 CPU 眼里都表现为内存读写，但它们在触发机制和使用场景上存在显著区别。串口和时钟依赖回调函数，而 VGA 则通过定期更新机制进行屏幕刷新。



























