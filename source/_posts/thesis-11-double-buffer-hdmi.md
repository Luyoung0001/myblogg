---
title: "毕业设计记录（11）：双 buffer HDMI 控制器"
date: 2026-01-25 10:03:13
tags:
  - "graduation thesis"
  - "HDMI"
  - "double buffer"
categories:
  - "Thesis Project"
---

## HDMI 控制器

在目前的 FPGA 板卡上设计的资源占用率如下。其中 mig_axi_32：

```TXT
+----------------------------+------+-------+------------+-----------+-------+
|          Site Type         | Used | Fixed | Prohibited | Available | Util% |
+----------------------------+------+-------+------------+-----------+-------+
| Slice LUTs*                | 6522 |     0 |          0 |     20800 | 31.36 |
|   LUT as Logic             | 5843 |     0 |          0 |     20800 | 28.09 |
|   LUT as Memory            |  679 |     0 |          0 |      9600 |  7.07 |
|     LUT as Distributed RAM |  452 |     0 |            |           |       |
|     LUT as Shift Register  |  227 |     0 |            |           |       |
| Slice Registers            | 5695 |     2 |          0 |     41600 | 13.69 |
|   Register as Flip Flop    | 5683 |     2 |          0 |     41600 | 13.66 |
|   Register as Latch        |    0 |     0 |          0 |     41600 |  0.00 |
|   Register as AND/OR       |   12 |     0 |          0 |     41600 |  0.03 |
| F7 Muxes                   |   63 |     0 |          0 |     16300 |  0.39 |
| F8 Muxes                   |    0 |     0 |          0 |      8150 |  0.00 |
| Unique Control Sets        |  227 |       |          0 |      8150 |  2.79 |
+----------------------------+------+-------+------------+-----------+-------+
```

soc_top：

```txt
+----------------------------+------+-------+------------+-----------+-------+
|          Site Type         | Used | Fixed | Prohibited | Available | Util% |
+----------------------------+------+-------+------------+-----------+-------+
| Slice LUTs*                | 4715 |     0 |          0 |     20800 | 22.67 |
|   LUT as Logic             | 4447 |     0 |          0 |     20800 | 21.38 |
|   LUT as Memory            |  268 |     0 |          0 |      9600 |  2.79 |
|     LUT as Distributed RAM |  264 |     0 |            |           |       |
|     LUT as Shift Register  |    4 |     0 |            |           |       |
| Slice Registers            | 4508 |     0 |          0 |     41600 | 10.84 |
|   Register as Flip Flop    | 4508 |     0 |          0 |     41600 | 10.84 |
|   Register as Latch        |    0 |     0 |          0 |     41600 |  0.00 |
| F7 Muxes                   |   25 |     0 |          0 |     16300 |  0.15 |
| F8 Muxes                   |    3 |     0 |          0 |      8150 |  0.04 |
| Unique Control Sets        |  364 |       |          0 |      8150 |  4.47 |
+----------------------------+------+-------+------------+-----------+-------+
```

可以看到一共消耗了大约 54% 的 LUT，这是相当高的资源占用。soc_top 的 axi_clk 100MHz 的频率已经综合不成功了，因为高的资源占用限制了布局布线路径，带来更多的延迟。因此我将 soc_top 的频率降低到 62.5MHz，将 HDMI 控制器的参数从 60Hz 1080P 降低到 24Hz 1080P。

**说明**：降低刷新率不会影响静态画面的分辨率（仍然是 1920×1080），但会降低动态画面的流畅度。由于本项目主要显示静态文本（Shell 界面），24Hz 的刷新率已经足够，用户几乎感受不到卡顿。如果需要播放动画或视频，则需要更高的刷新率。

### 单 HDMI buffer

现在考虑一种情形。上电后，HDMI 控制器便开始从 DDR 显存中读取数据，这时候 CPU 还没跑到给显存写数据的指令。因此此时显示器会显示未初始化的数据（通常是随机噪点或全黑画面）。

当 CPU 跑到写显存的指令时，HDMI 可能正在扫描高地址区域，因此显示器依然显示的是未初始化的数据。当 HDMI 扫描完一帧后，回到显存低地址继续扫描，这时候就能读取到 CPU 已经写入的数据了。然后就会在显示器上看到部分画面突然出现。由于 HDMI 控制器的读取速度比 CPU 的写入速度快（HDMI 支持 AXI 突发传输，而 CPU 通过 `st.w` 指令逐字写入，且两者还要竞争同一个 AXI 总线），因此显示器的画面会出现渐进式更新：每一帧画面都会比上一帧多显示一些内容，直到 CPU 完成整个帧的渲染。

**这种现象称为"渐进式渲染"或"不完整帧显示"**。虽然不是传统意义上的画面撕裂（screen tearing，即同一帧中显示新旧两部分内容），但本质问题相同：**CPU 正在写入（渲染）的同时，HDMI 控制器正在读取（显示）同一块显存**，导致显示内容与预期不一致。

怎么解决这个问题呢？就要依靠双 frame buffer 来解决。

### 双 HDMI buffer

双 HDMI buffer 的思路是，可以通过软件控制来决定让 hdmi 扫描哪一个 buffer。当处理器正在渲染 buffer0 的时候，通过软件控制让 hdmi 扫描 buffer1。当 buffer0 渲染完成的时候，给 hdmi 的 base_reg 写入 buffer0 的地址让它去扫描 buffer0。

这里还有一个小细节：那就是 hdmi 控制器的垂直同步。虽然 CPU 已经渲染结束了可以通知 hdmi 换 buffer_base 了，但实际上还得等 hdmi 显示垂直同步，这个过程不会等很久：

```C
// 等待垂直同步 (读取硬件V-Sync标志，等待变化)
void hdmi_wait_vsync(void) {
    // vsync_flag 在每帧开始时翻转（0->1 或 1->0）
    // 我们读取当前值，然后轮询直到它变化

    volatile uint32_t* hdmi_status = (volatile uint32_t*)HDMI_STATUS_REG;
    uint32_t last_vsync = *hdmi_status & 0x1;  // 读取当前 vsync_flag (bit0)

    // 轮询直到 vsync_flag 翻转
    while ((*hdmi_status & 0x1) == last_vsync) {
        // 等待 vsync_flag 变化
    }
}
```

hdmi_top 的两个关键信号：

```Verilog
    // 控制信号
    input  wire [31:0]              fb_base_addr,   // 显存基地址
    input  wire                     hdmi_enable,    // HDMI 使能
```
hdmi_enable 这个信号很重要，因为在上电的时候，ddr 可能还没有初始化好，或者可以选择是否打开 hdmi 显示（比如调试的时候，只需要用到 uart）：

```C
void hdmi_init(void) {
    hdmi_enable(1);
    // hdmi_clear(COLOR_BLUE);
}

void hdmi_enable(int enable) {
    volatile uint32_t* hdmi_enable_reg = (volatile uint32_t*)HDMI_ENABLE_REG;
    if (enable) {
        *hdmi_enable_reg = 1;  // 使能 HDMI
    } else {
        *hdmi_enable_reg = 0;  // 禁用 HDMI
    }
}
```

在程序执行的最开始，可以打开 `hdmi_enable()`。对于 fb_base_addr，可以通过软件来切换：

```C
// 获取后台 buffer 指针
void* hdmi_get_back_buffer(void) {
    if (current_draw_buffer == 0) {
        return (void*)HDMI_FB_BASE_A;
    } else {
        return (void*)HDMI_FB_BASE_B;
    }
}

void hdmi_swap_buffers(void) {
    hdmi_wait_vsync();

    volatile uint32_t* hdmi_fb_addr_reg = (volatile uint32_t*)HDMI_FB_ADDR_REG;

    // 切换显示的 buffer
    if (current_display_buffer == 0) {
        // 当前显示 Buffer A，切换到 Buffer B
        *hdmi_fb_addr_reg = HDMI_FB_BASE_B;
        current_display_buffer = 1;
        current_draw_buffer = 0;
    } else {
        *hdmi_fb_addr_reg = HDMI_FB_BASE_A;
        current_display_buffer = 0;
        current_draw_buffer = 1;
    }
    fb = (volatile uint16_t*)hdmi_get_back_buffer();
}
```

### 软件控制流程

上电的时候，默认让 fb 指向 buffer1。当 buffer1 渲染结束后，hdmi 就可以读取 buffer1 了（默认刚开始是 buffer0，显示初始值）。

### HDMI 核心

HDMI 控制器的核心是一组控制器，这个 IP 是板卡提供的，并提供了一个实例。我只需要把 axi 通道拿到的数据替换掉 `hdmi_data` 就能显示：

```Verilog
module hdmi_colorbar(
         input          clk             ,
         input          rst_n           ,
          //hdmi信号
        output          hdmi_hpd       ,
        output          hdmi_tx_clk_n  ,
	    output          hdmi_tx_clk_p  ,
	    output          hdmi_tx_chn_r_n,
	    output          hdmi_tx_chn_r_p,
        output          hdmi_tx_chn_g_n,
        output          hdmi_tx_chn_g_p,
	    output          hdmi_tx_chn_b_n,
	    output          hdmi_tx_chn_b_p
    );
wire          clk1x          ;
wire          clk5x          ;
wire          hdmi_data_req  ;
wire [23:0]   hdmi_data      ; //{R,G,B}
wire [11:0]   h_addr         ; //数据横坐标地址
wire [11:0]   v_addr         ;//数据纵坐标
assign  hdmi_hpd=1'b1;
  clock_hdmi clock_hdmi
   (
    // Clock out ports
    .clk_out1(clk1x),     // output clk_out1
    .clk_out2(clk5x),     // output clk_out2
    // Status and control signals
    .resetn(rst_n), // input resetn
    .locked(),       // output locked
   // Clock in ports
    .clk_in1(clk));      // input clk_in1
gen_wave u_gen_wave(
    .clk_in       ( clk1x       ),
    .rst_n        ( rst_n        ),
    .rd_data_req  ( hdmi_data_req ),
    .rd_data      ( hdmi_data     ),
    .h_addr       ( h_addr       ),
    .v_addr       ( v_addr       )
);

hdmi_driver u_hdmi_driver(
    .clk1x           ( clk1x           ),
    .clk5x           ( clk5x           ),
    .rst_n           ( rst_n           ),
    .hdmi_data_req   ( hdmi_data_req   ),
    .hdmi_data       ( hdmi_data       ),
    .h_addr          ( h_addr          ),
    .v_addr          ( v_addr          ),
    .hdmi_tx_clk_n   ( hdmi_tx_clk_n   ),
    .hdmi_tx_clk_p   ( hdmi_tx_clk_p   ),
    .hdmi_tx_chn_r_n ( hdmi_tx_chn_r_n ),
    .hdmi_tx_chn_r_p ( hdmi_tx_chn_r_p ),
    .hdmi_tx_chn_g_n ( hdmi_tx_chn_g_n ),
    .hdmi_tx_chn_g_p ( hdmi_tx_chn_g_p ),
    .hdmi_tx_chn_b_n ( hdmi_tx_chn_b_n ),
    .hdmi_tx_chn_b_p  ( hdmi_tx_chn_b_p  )
);

endmodule
```

### HDMI 顶层模块

HDMI 模块实现了从 DDR 内存通过 AXI 总线读取帧缓冲数据，并输出到 HDMI 显示器的完整数据通路。核心挑战在于：

- **时钟域不同**：AXI 总线工作在 62.5 MHz，HDMI 像素时钟为 59.4 MHz (1080P@24Hz)；
- **速率匹配**：AXI 突发读取是间歇性的，HDMI 输出是连续的；
- **数据完整性**：必须保证显示数据不出现撕裂或闪烁。

### 系统架构

```txt
┌────────────────────────────────────────────────────────────────────┐
│                           hdmi_top                                 │
│                                                                    │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │              │    │                  │    │                  │  │
│  │  hdmi_axi    │───▶│  hdmi_line       │───▶│   hdmi_driver    │  │
│  │  _reader     │    │  _buffer         │    │                  │  │
│  │              │    │                  │    │                  │  │
│  └──────────────┘    └──────────────────┘    └──────────────────┘  │
│        │                     │                       │             │
│        │ AXI 时钟域          │ 跨时钟域              │ 像素时钟域      │
│        │ (62.5 MHz)         │                       │ (59.4 MHz)   │
│        ▼                     │                       ▼             │
│  ┌──────────────┐           │              ┌──────────────────┐    │
│  │   AXI        │           │              │   HDMI PHY       │    │
│  │   总线        │           │              │   (TMDS 输出)    │    │
│  └──────────────┘           │              └──────────────────┘    │
│                              │                                     │
└────────────────────────────────────────────────────────────────────┘
```

从上图可以看到，这里面的核心设计是双行缓冲 (Double Line Buffer)，使得 hdmi_driver 的取读连续起来。工作原理大致如下：

```txt
时刻 T1:
  Buffer A: [HDMI 正在读取第 N 行]
  Buffer B: [AXI 正在写入第 N+1 行]

时刻 T2 (切换后):
  Buffer A: [AXI 正在写入第 N+2 行]
  Buffer B: [HDMI 正在读取第 N+1 行]
```
这里面隐含了一个条件，那就是 axi 一定要比 hdmi 速度快，事实上 62.5MHz 和 59.4MHz 相比已经很极限了。来做一个简单的数学题：

```txt
像素时钟: 59.4 MHz
每行总像素: 2200 (包含消隐区)

行周期 = 2200 / 59.4 MHz = 37.0 μs

AXI 提供一行数据时间

每行数据: 1920 像素 × 2 字节 = 3840 字节
AXI 数据宽度: 32 bit = 4 字节
需要传输次数: 3840 / 4 = 960 次

AXI 时钟: 62.5 MHz
理想时间 = 960 / 62.5 MHz = 15.4 μs

```
看似 AXI 速度约是 HDMI 消耗速度的 2 倍，裕量充足。但是别忘了 CPU 和 HDMI 都是 master，它们共享一个 axi 通道。尤其是访存的时候，还要不断的和 ddr 进行握手等待，一个突发传输大约消耗 20 个时钟周期，一次突发长度为 16 的话，需要的突发次数为 60 次，这就消耗掉了 1200 个 axi 周期，HDMI 访存的效率可能就大打折扣了。

如果不去掉 cache、tlb 等模块，axi 频率也就最多 30MHz，根本达不到 62.5MHz。频率上不去，AXI 的带宽持续性不足，不管设置多少 line buffer 都无法满足最低要求的 1080P * 24Hz。带宽是硬性约束：AXI 带宽 ≥ HDMI 消耗带宽，否则无解。

### 切换条件

缓冲切换必须同时满足两个条件：
1. **HDMI 读完当前行** (`rd_line_done`)
2. **AXI 写完下一行** (`wr_ready`)

```verilog
// 切换逻辑
always @(posedge rd_clk or negedge rd_rst_n) begin
    if (!rd_rst_n) begin
        rd_buf_sel <= 1'b0;
    end else if (rd_line_done && wr_ready) begin
        rd_buf_sel <= ~rd_buf_sel;  // 翻转选择信号
    end
end
```

这个切换逻辑按理说不应该存在，因为 HDMI 读取是自动的进行了，没有其它约束。但是 axi 写完是一个很难满足的条件，只要 axi 速度不够，rd_buf_sel 就算没反转，hdmi_driver 都会把采集来的信号打到显示屏下一行。

因此，这个可以优化成：

```Verilog
always @(posedge rd_clk or negedge rd_rst_n) begin
    if (!rd_rst_n) begin
        rd_buf_sel <= 1'b0;
    end else if (rd_line_done) begin
        rd_buf_sel <= ~rd_buf_sel;
    end
end
```

这样管是否写完，都进行换行读写。如果此时没有写完，但是 hdmi_driver 已经读完，hdmi 就会显示没有写完的行，表现为某一行前半部分有画面后半部分没有。如果每一行都如此，那只能说明 axi 带宽不够了，只能增加 axi 频率。

事实上，我就遇见了这样的性能问题，将处理器频率提高到 75MHz 后，依然只能渲染 8/7 的屏幕，于是我直接将 hdmi_reader 的 axi 突发长度从原来的 16 提升到了 256，带宽提升了 2 倍多，于是自然就支持 1080P 30Hz 的刷新率了。

### 数据流

```txt
DDR 内存
    │
    │ AXI 突发读取 (16 beats × 60 bursts)
    ▼
hdmi_axi_reader
    │
    │ line_wr_en, line_wr_addr, line_wr_data
    ▼
hdmi_line_buffer (Buffer A 或 B)
    │
    │ rd_data (32-bit)
    ▼
像素选择 (h_addr[0])
    │
    │ line_rd_data (16-bit RGB565)
    ▼
RGB565 → RGB888 转换
    │
    │ hdmi_data (24-bit RGB888)
    ▼
hdmi_driver → TMDS 编码 → HDMI 输出

```

### 时序分析

#### 时钟参数 (1080P@24Hz)

| 时钟 | 频率 | 用途 |
|------|------|------|
| aclk | 62.5 MHz | AXI 总线 |
| pix_clk | 59.4 MHz | 像素时钟 |
| pix_clk_5x | 297 MHz | TMDS 串行化 |

#### 1080P 时序参数

| 参数 | 值 |
|------|-----|
| 有效像素 | 1920 × 1080 |
| 总像素 | 2200 × 1125 |
| 行周期 | 2200 / 59.4 MHz = 37.0 μs |
| 帧周期 | 1125 × 37.0 μs = 41.6 ms |

#### 带宽需求

```
每行数据量 = 1920 × 2 = 3840 字节
每秒行数 = 1125 × 24 = 27000 行
带宽需求 = 3840 × 27000 = 103.68 MB/s

AXI 理论带宽 = 62.5 MHz × 4 字节 = 250 MB/s
利用率 = 103.68 / 250 = 41.5%
```

#### 行读取时间裕量

```
行周期 = 37.0 μs
AXI 读取一行时间 = 60 bursts × 16 beats / 62.5 MHz ≈ 15.4 μs
裕量 = 37.0 - 15.4 = 21.6 μs
```












