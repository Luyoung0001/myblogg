---
title: "毕业设计记录（9）：PS/2 键盘输入"
date: 2026-01-20 11:03:13
tags:
  - "graduation thesis"
  - "PS/2"
  - "keyboard"
categories:
  - "Thesis Project"
---

## ps2 键盘的软件仿真

在用户程序层面：

```C
int kb_get_scancode(void) {
    if (kb_head == kb_tail) {
        return -1; /* Buffer empty */
    }
    uint8_t code = kb_buffer[kb_tail];
    kb_tail = (kb_tail + 1) % KB_BUF_SIZE;
    return code;
}

void shell_run(void) {
    printf("\n");
    printf("========================================\n");
    printf("  xOS - Simple Operating System\n");
    printf("  for LoongArch32R SoC\n");
    printf("========================================\n");
    printf("\n");
    printf("Type 'help' for available commands.\n");
    printf("\n");

    shell_print_prompt();

    /* Main loop - keyboard input comes from interrupt buffer */
    while (1) {
        int scancode = kb_get_scancode();
        if (scancode >= 0) {
            process_scancode((uint8_t)scancode);
        }
    }
}
```

当程序进入 `while(1)`中的时候，程序会不断地尝试从 `kb_buffer` 中获取数据，如果没有拿到就就继续循环。如果这时候用户从键盘输入了字符，那么进入中断后，中断处理程序就会给 `kb_buffer` 中填充数据，填充完后就会返回，那么再次拿数据就会拿到。ps2 键盘的中断处理程序如下：

```C
void __attribute__((noinline)) ps2_irq_handler(void) {
    while (bsp_ps2_data_available()) {
        uint8_t scancode = (uint8_t)bsp_ps2_read();
        int next = (kb_head + 1) % KB_BUF_SIZE;
        if (next != kb_tail) {
            kb_buffer[kb_head] = scancode;
            kb_head = next;
        }
    }
}
```
`bsp_ps2_read()` 一旦读取，ps2 中断信号就会消失，不然会一直中断：

```Verilog
        if (rd_ack) begin
            rx_valid_reg <= 1'b0;
        end
//=============================================================================
// 中断输出
// 当有新数据且中断使能时，产生中断信号
// CPU 读取 PS2_DATA 后 (rd_ack)，rx_valid 清除，中断也随之清除
//=============================================================================
assign interrupt = rx_valid_reg & int_enable;
```

## 仿真系统如何输入

### 简单粗暴的模拟输入

#### 模拟器注入字符

在每一个周期，模拟器都会去检查键盘的输入情况：

```C
static inline void exec_once() {
    // Process PS2 keyboard input from host
    ps2_process_input();
    ps2_update();
    // Set PS2 simulation signals to top module
    top->sim_ps2_scancode = ps2_get_scancode();
    top->sim_ps2_scancode_valid = ps2_scancode_valid() ? 1 : 0;
    ...
}
```
如果确实读取到了字符，就把字符转化为扫描码，然后将通码、断码入队：

```C
void ps2_process_input(void) {
    // Read from stdin (non-blocking, already set up by uart)
    char ch;
    if (read(STDIN_FILENO, &ch, 1) == 1) {
        // Convert ASCII to scancode
        uint8_t scancode = ascii_to_ps2_scancode(ch);
        if (scancode != 0) {
            // Queue make code (key press)
            ps2_scancode_queue.push(scancode);

            // Queue break code (key release): 0xF0 followed by scancode
            ps2_scancode_queue.push(PS2_BREAK_PREFIX);
            ps2_scancode_queue.push(scancode);
        }
    }
}
```

接着仿真环境继续处理，根据上一步，判断是否接收到了扫描码，如果接收到了，就拉高 valid 信号，并且在队列中取出一个扫描码。

一般而言，如果 CPU 的频率是 20MHz，ps2 时钟一般在 10～16.7KHz，这里面就要过采样。这样简单算一下，采样时钟和 ps2 的时钟比例差不多是 2000 左右。这意味着，我们在仿真环境如果发射一个比特，那么至少让 CPU 应该保持 2000 个周期。

一个扫描码 11 位，那么就需要 CPU 时钟经过大约 22000 个周期，因此每次处理一个扫描码都得至少冷静 22000 个周期。


冷静期没结束，直接返回。如果冷静期结束，看信号是否有效，如果依然有效，再看是否依然需要保持，如果需要就继续保持，否则清除仿真系统生成的信号，并且设置冷静期。

如果冷静期结束且队列不空，那么就去 queue 中取，取到之后，重新设置信号保持时间：

```C
void ps2_update(void) {
    // Cooldown period between scancodes
    if (ps2_cooldown > 0) {
        ps2_cooldown--;
        return;
    }
    // If currently injecting a scancode, keep valid signal high for a short time
    if (ps2_valid_signal) {
        if (ps2_inject_delay > 0) {
            ps2_inject_delay--;
        } else {
            // Clear valid signal after injection pulse complete
            ps2_clear_scancode();
            // Start cooldown to give CPU time to process
            ps2_cooldown = PS2_COOLDOWN_CYCLES;
        }
        return;
    }
    // If queue has scancodes and we're not currently injecting
    if (!ps2_scancode_queue.empty()) {
        // Get next scancode from queue
        ps2_current_scancode = ps2_scancode_queue.front();
        ps2_scancode_queue.pop();

        // Set valid signal for a short pulse
        ps2_valid_signal = 1;
        ps2_inject_delay = PS2_VALID_CYCLES;
    }
}
```

假设用户第一次按下键盘，队列中保存了 3 个扫描码。首先会在队列中取一个扫描码，然后设置有效保持的时间。

然后我们从仿真往 ps2 控制器中注入信号：

```C
    top->sim_ps2_scancode = ps2_get_scancode();
    top->sim_ps2_scancode_valid = ps2_scancode_valid() ? 1 : 0;
```

此时 soc_top 会把信号传递到 confreg，confreg 把信号传送到 ps2_ctrl：

```Verilog
    // PS2 simulation bypass interface (for Verilator simulation)
    input  wire [7:0]                sim_ps2_scancode,      // Simulation scancode input
    input  wire                      sim_ps2_scancode_valid // Simulation scancode valid

    ...

module ps2_ctrl (
    input  wire        clk,           // 系统时钟 (50 MHz)
    input  wire        resetn,        // 复位 (低有效)

    // PS2 接口 (from GPIO)
    input  wire        ps2_clk_in,    // PS2 时钟输入
    input  wire        ps2_data_in,   // PS2 数据输入

    // 寄存器接口
    output reg  [7:0]  rx_data,       // 接收数据
    output wire        rx_valid,      // 数据有效
    output reg         parity_err,    // 奇校验错误
    output reg         frame_err,     // 帧错误
    output reg         overflow,      // 缓冲区溢出

    input  wire        rd_ack,        // CPU 读取确认 (清除 rx_valid)
    input  wire        clear_err,     // 清除错误标志
    input  wire        enable,        // 接收使能
    input  wire        int_enable,    // 中断使能

    // 中断输出
    output wire        interrupt,     // 中断信号 (active high)

    // Simulation bypass interface (直接注入扫描码，绕过PS2协议)
    input  wire [7:0]  sim_scancode,      // 仿真扫描码输入
    input  wire        sim_scancode_valid // 仿真扫描码有效信号
);
...

// Simulation bypass: detect rising edge of sim_scancode_valid
reg sim_scancode_valid_prev;
wire sim_inject = sim_scancode_valid && !sim_scancode_valid_prev;

always @(posedge clk or negedge resetn) begin
    if (!resetn) begin
        sim_scancode_valid_prev <= 1'b0;
    end else begin
        sim_scancode_valid_prev <= sim_scancode_valid;
    end
end

```

一旦检测到 `sim_scancode_valid` 的上升沿后，就会拉高 `sim_inject` 信号。`sim_inject` 信号很关键，它有效的时候，直接将数据旁路给 `rx_data`、`rx_valid_reg`，下一个周期中断信号也会拉起来：

```Verilog
        // Simulation bypass: 直接注入扫描码 (优先级最高)
        if (sim_inject) begin
            if (rx_valid_reg) begin
                overflow <= 1'b1;
            end
            rx_data      <= sim_scancode;
            rx_valid_reg <= 1'b1;
            // 仿真注入不产生校验/帧错误
        end
...
//=============================================================================
// 当有新数据且中断使能时，产生中断信号
//==================================================
assign interrupt = rx_valid_reg & int_enable;
...
    // 寄存器接口
    output reg  [7:0]  rx_data,       // 接收数据
    output wire        rx_valid,      // 数据有效
...
```
`ps2_ctrl` 会把`rx_data` 一直保存起来，直到下一个仿真到来，这样可以随时让 `confreg` 读取：

```Verilog
// PS2 status register (read-only, directly from controller)
wire [31:0] ps2_status = {28'd0, ps2_overflow, ps2_frame_err, ps2_parity_err, ps2_rx_valid};
always @(posedge clk) begin
    if (~resetn) begin
        conf_rdata_reg <= 32'd0;
    end
    else if (conf_ren) begin
        case (conf_raddr[15:0])
...
            `PS2_DATA_ADDR:    conf_rdata_reg <= {24'd0, ps2_rx_data};
...
```

一旦中断处理程序发起读取，ps2_rd_ack_reg 也会在下一个周期拉高：

```Verilog

always @(posedge clk) begin
    if (!resetn) begin
        ps2_ctrl_reg   <= 32'h6;  // Default: enable=1, int_enable=1, clear_err=0
        ps2_rd_ack_reg <= 1'b0;
    end
    else begin
        // Read acknowledge: pulse when reading PS2_DATA
        ps2_rd_ack_reg <= conf_ren & (conf_raddr[15:0] == `PS2_DATA_ADDR);

        // Write PS2_CTRL register
        if (write_ps2_ctrl) begin
            ps2_ctrl_reg <= conf_wdata[31:0];
        end
        else begin
            // Auto-clear bit0 (clear_err) after one cycle
            ps2_ctrl_reg[0] <= 1'b0;
        end
    end
end
```
拉高之后，此时就应该在 `ps2_ctrl` 中下拉中断信号了（必须在中断处理程序返回之前拉低，不然会重复中断）：

```Verilog
        // 清除 rx_valid 当 CPU 读取
        if (rd_ack) begin
            rx_valid_reg <= 1'b0;
        end
```

从而 ps2 的 `interrupt` 信号拉低。

从以上过程可以看到，这是不完全的模拟，因为无法真正模拟和验证 `ps2_clk` 的功能，只是 bypass 了两个现成的信号。仿真的最终目的是初次验证，因此为了确保 ps2 控制器正确，数字化仿真是必须的。

### 完全数字化的模拟

完全数字化的模拟，就是让仿真环境中的 soc_top 发射真正的 `ps_clk`、`ps2_data` 信号。

思路也很简单，就是遵守键盘通码的约定，在 `ps2_data` 中编码信号，每一个bit 保持 2000 个周期，同时产生 `ps2_clk` 信号。


### ps2_ctrl 的设计

这个 IP 不像 ddr 控制器、hdmi 控制器、sd 控制器那样复杂。因此完全可以自主控制，因此这里打算使用大量的篇幅来详细讨论 `ps2_ctrl` 的设计。

#### 同步器
由于 `ps2` 控制器的频率工作在 20MHz，而 `ps2_clk` 的时钟频率为 10-16.7kHz，因此这里涉及到稳态的问题。

思路是使用 3 级同步器防止亚稳态：

```Verilog
  // 使用 3 级同步器防止亚稳态
  reg [2:0] ps2_clk_sync;
  reg [2:0] ps2_data_sync;

  always @(posedge clk or negedge resetn) begin
      if (!resetn) begin
          ps2_clk_sync  <= 3'b111;
          ps2_data_sync <= 3'b111;
      end else begin
          ps2_clk_sync  <= {ps2_clk_sync[1:0], ps2_clk_in};
          ps2_data_sync <= {ps2_data_sync[1:0], ps2_data_in};
      end
  end
```

工作原理是，当 ps2_clk_in 到来的时候，由于它的时钟很慢，主 clk 过 1000 个周期左右，ps2_clk_in 才会拉高一次。因此，ps2_clk_sync 很轻松就会达到 3'b111 的状态。此时我们挑选最高位作为 ps2_clk_in 的采样信号：

```Verilog
wire ps2_clk_s  = ps2_clk_sync[2];   // 同步后的时钟
wire ps2_data_s = ps2_data_sync[2];  // 同步后的数据
```

换句话说，就是再赌 `ps2_clk` 在 `clk` 三个周期后趋于稳定，但这并不盲目。

#### 亚稳态的物理特性

亚稳态持续时间是指数衰减的，当触发器进入亚稳态时，它不会永远保持不稳定状态。根据物理规律，亚稳态持续时间超过 t 的概率:

```math
  P(T > t) = e^(-t/τ)
```
其中:
- - τ 是时间常数（通常是几十到几百皮秒）
- - t 是等待时间


对于现代 FPGA/ASIC：τ ≈ 100 ps (皮秒)：

- 等待 1 个时钟周期 (50ns @ 20MHz)：P(亚稳态仍存在) = e^(-50000/100) = e^(-500) ≈ 10^(-217)， 几乎不可能！

- 等待 2 个时钟周期 (100ns)：P(亚稳态仍存在) = e^(-1000) ≈ 10^(-434)

- 等待 3 个时钟周期 (150ns)：P(亚稳态仍存在) = e^(-1500) ≈ 10^(-651)


因此，三级消除亚稳态的同步器已经相当可靠了，也是工业表准。

#### 边缘检测

这个很简单：

```Verilog
reg ps2_clk_prev;

always @(posedge clk or negedge resetn) begin
    if (!resetn)
        ps2_clk_prev <= 1'b1;
    else
        ps2_clk_prev <= ps2_clk_s;
end

// 检测下降沿: 之前为高，现在为低
wire ps2_clk_negedge = ps2_clk_prev && !ps2_clk_s;
```

#### 脉冲滤波器

虽然 PS2 信号通常比较干净，但可能存在：
- 毛刺（glitch）：短暂的电平跳变
- 抖动（jitter）：信号边沿不稳定
- 噪声：电磁干扰

这是一个数字低通滤波器，使用计数器实现：
```Verilog
  reg [3:0] clk_filter;        // 4位计数器 (0-15)
  reg       ps2_clk_filtered;  // 滤波后的输出

  always @(posedge clk or negedge resetn) begin
      if (!resetn) begin
          clk_filter <= 4'hF;           // 初始值15（高电平）
          ps2_clk_filtered <= 1'b1;
      end else begin
          // 移位滤波：根据输入调整计数器
          if (ps2_clk_s)
              clk_filter <= (clk_filter < 4'hF) ? clk_filter + 1'b1 :
  4'hF;
          else
              clk_filter <= (clk_filter > 4'h0) ? clk_filter - 1'b1 :
  4'h0;

          // 施密特触发效果：只有计数器到达极值才改变输出
          if (clk_filter == 4'hF)
              ps2_clk_filtered <= 1'b1;
          else if (clk_filter == 4'h0)
              ps2_clk_filtered <= 1'b0;
      end
  end
```

关键特性
1. 惯性：计数器需要连续多个周期才能从 15 变到 0（或反向）
2. 施密特触发：
 - 输出只在计数器 = 15 时变为 1
 - 输出只在计数器 = 0 时变为 0
 - 中间值（1-14）不改变输出
3. 抗毛刺：短暂的跳变不会影响输出

施密特触发器有两个阈值，这样可以防止信号在阈值附近反复跳变。
  - 上阈值（15）：超过这个值才输出 1
  - 下阈值（0）：低于这个值才输出 0
  - 滞回区（1-14）：在这个区间内保持原来的输出

#### 滤波后的始终下降沿检测

使用滤波后的时钟检测下降沿：
```Verilog
reg ps2_clk_filtered_prev;
always @(posedge clk or negedge resetn) begin
    if (!resetn)
        ps2_clk_filtered_prev <= 1'b1;
    else
        ps2_clk_filtered_prev <= ps2_clk_filtered;
end

wire clk_falling = ps2_clk_filtered_prev && !ps2_clk_filtered && enable;
```


通过以上分析可知，其实更加稳妥的做法是使用脉冲滤波器以及施密特触发效果，是最好的。

### 软件层次

接下来就是仿真系统的修改了，和之前一样，大体的流程不变，我们需要改变的是 `update()` 函数：

```C
void ps2_update2(void) {
    if (ps2_valid_signal) {
        ps2_clear_scancode();
    }
    if (ps2_cooldown > 0) {
        ps2_cooldown--;
        return;
    }
    // If queue has scancodes and we're not currently injecting
    if (!ps2_scancode_queue.empty()) {
        // Get next scancode from queue
        // printf("\n[SIM] ps2_current_scancode pop!\n");
        ps2_current_scancode = ps2_scancode_queue.front();
        ps2_scancode_queue.pop();

        // Set valid signal for a short pulse
        ps2_valid_signal = 1;
        ps2_cooldown = 60000;
    }
}
```
这里面取消了 `ps2_valid_signal` 信号的保持，思考一下为什么。一旦取出一个 scancode，在下一个周期清除信号防止重复编码，并且进入冷静期。

编码：

```C
void ps2_encode_scancode(void) {
    if (bit_index != -1) {
        return; // 正在发送数据，不能重新编码
    }

    if (!ps2_valid_signal) {
        return; // 没有有效的 scancode 需要编码
    }

    // 构造 11 bit 的数据流
    // 起始位：0
    // 数据位：8 bit，最低位先发
    // 奇偶校验位：奇校验
    // 停止位：1
    ps2_data_bits[0] = 0; // 起始位

    // 数据位
    for (int i = 0; i < 8; i++) {
        ps2_data_bits[i + 1] = (ps2_current_scancode >> i) & 0x1;
    }
    // 计算奇偶校验位（奇校验）
    int one_count = 0;
    for (int i = 1; i <= 8; i++) {
        if (ps2_data_bits[i] == 1) {
            one_count++;
        }
    }
    ps2_data_bits[9] = (one_count % 2 == 0) ? 1 : 0; // 奇校验

    ps2_data_bits[10] = 1; // 停止位
    ps2_data_state = ps2_data_bits[0];
    bit_index = 0;           // 开始发送
    ps2_clk_cycle_count = 0; // 重置时钟计数器
}
```
接着就是控制相应的信号，需要注意的是，由于 ps2_ctrl 在时钟下降沿采集信号，因此当上升沿的时候，再进行数据发送，这是 ps2 键盘的协议：

```C
uint8_t ps2_info() {
    if (bit_index != -1) {
        // 正在发送数据
        if (ps2_clk_cycle_count >= 2000) {
            // 切换时钟状态
            ps2_clk_state = !ps2_clk_state;
            ps2_clk_cycle_count = 0;

            if (ps2_clk_state == 1) {
                // 时钟上升沿，准备下一个 bit 的数据
                // 这样在下一个下降沿时，数据已经稳定
                bit_index++;
                if (bit_index < 11) {
                    // 准备下一个 bit 的数据
                    ps2_data_state = ps2_data_bits[bit_index];
                } else {
                    bit_index = -1;
                    ps2_data_state = 1;
                }
            }
        } else {
            ps2_clk_cycle_count++;
        }
    } else {
        // 空闲状态，保持高电平
        ps2_clk_state = 1;
        ps2_data_state = 1;
        ps2_clk_cycle_count = 0;
    }
    return (ps2_clk_state << 1) | ps2_data_state;
}
```

最终经过 debug，顺利启动：

```bash
xos> help
Available commands:
  %-10s - help
  %-10s - echo
  %-10s - clear
  %-10s - info
xos>
```


## 总结

以上讨论了 ps2 控制器的硬件设计到软件仿真验证功能。

