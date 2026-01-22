---
title: 毕设(7)：uart 的测试
date: 2026-01-19 10:03:13
tags: 毕设
categories: SoC
mermaid: true
---

## 仿真系统中的测试

因为之前的 UART 仿真测试思路是捕获 soc_top 中的信号：

```Verilog
    // APB UART simulation interface (directly from AXI slave mux to APB bridge)
    output wire                      apb_uart_awvalid,  // 写地址有效
    output wire [`Lawaddr     -1 :0] apb_uart_awaddr,   // 地址
    output wire [`Lwdata      -1 :0] apb_uart_wdata,    // 数据
    output wire [`Lwstrb      -1 :0] apb_uart_wstrb,    // mask
    output wire                      apb_uart_wvalid,   // 写有效
    input  wire                      apb_uart_wready_in,// slave ready
```

然后在 simulator 中捕获：

```C
void uart_read_handling() {
    if (top->apb_uart_wvalid && !last_apb_uart_wvalid) {
        uint32_t uart_addr = top->apb_uart_awaddr;
        uint8_t uart_char = top->apb_uart_wdata & 0xFF;
        if (uart_addr == 0x1fe001e0) { // THR register
            if ((uart_char >= 32 && uart_char <= 126) || uart_char == '\n' ||
                uart_char == '\r' || uart_char == '\t') {
                putchar(uart_char);
                fflush(stdout);
            }
        }
    }
}
```

这种方式**绕过了 UART 模块的验证**，直接在 APB 总线层面捕获数据，无法测试 UART 串行通信的时序正确性。更完整的测试方法是在 soc_top 捕获 UART_TX 信号，这样获取的是经过 UART 模块完整处理后的串行信号：

```Verilog
    ,output wire        debug_uart_tx      // UART TX 串行信号
```

这里需要明白一个事实。用户程序在编译的时候，需要确认波特率比如 9600、需要确认时钟频率比如 20MHz。用户程序编译后，执行的时候，就会用这些数据初始化 UART 控制器：

```C
int bsp_uart_init(uint8_t uart_id, uint32_t baudrate) {
    uint16_t divisor;

    /* 计算波特率除数: divisor = CLK / (16 * baudrate) */
    divisor = UART_CLK / (16 * baudrate);
    ...
    UART_REG(UART_DLM) = (divisor >> 8) & 0xFF; /* 高字节 */
    UART_REG(UART_DLL) = divisor & 0xFF;        /* 低字节 */
    ...
}
```

这样我们的 UART 就能以特定的频率将数据从 UART_TX 上传送出来。

那么在模拟器中呢？此时模拟器需要扮演 UART 接收设备的角色，按照 UART 协议解析串行信号。因此必须在仿真环境中配置正确的时钟频率和波特率参数，才能准确解码 UART_TX 上的串行数据：

```C
// Clock frequency for UART baud rate calculation
#define CLK_FREQ_HZ 20000000 // 20MHz
#define UART_BAUD 9600

// UART 串行信号解码器配置
#define CYCLES_PER_BIT (CLK_FREQ_HZ / UART_BAUD)  // 每位的时钟周期数：5208
#define CYCLES_HALF_BIT (CYCLES_PER_BIT / 2)      // 半位时间（用于采样点）：2604
```

这些参数对于正确解码串行信号至关重要。模拟器需要知道每个比特持续多少个时钟周期（CYCLES_PER_BIT），才能在正确的时刻采样数据。这也解释了为什么 UART 通信双方必须使用相同的波特率 - 只有采样时机一致，才能准确读取数据：

```C

void uart_read_handling_tx() {
    // 读取 UART TX 信号（来自 soc_top 的 debug_uart_tx）
    uint8_t uart_tx = top->debug_uart_tx;

    switch (uart_rx_state) {
        case RX_IDLE:
            // 等待起始位：检测 TX 从高到低的下降沿
            if (last_uart_tx == 1 && uart_tx == 0) {
                // 检测到起始位，切换到 START 状态
                uart_rx_state = RX_START;
                uart_rx_cycle_count = 0;
                uart_rx_data_byte = 0;
                uart_rx_bit_index = 0;
#ifdef DEBUG_UART_RX
                printf("[UART RX] Start bit detected at time %lu\n", main_time);
#endif
            }
            break;

        case RX_START:
            uart_rx_cycle_count++;
            // 在起始位的中点采样（半位时间）
            if (uart_rx_cycle_count >= CYCLES_HALF_BIT) {
                if (uart_tx == 0) {
                    // 起始位有效，准备接收数据位
                    uart_rx_state = RX_DATA;
                    uart_rx_cycle_count = 0;
                } else {
                    // 起始位无效（可能是毛刺），返回 IDLE
                    uart_rx_state = RX_IDLE;
                }
            }
            break;

        case RX_DATA:
            uart_rx_cycle_count++;
            // 每隔一位时间采样数据位
            if (uart_rx_cycle_count >= CYCLES_PER_BIT) {
                // 采样当前数据位（LSB first）
                if (uart_tx) {
                    uart_rx_data_byte |= (1 << uart_rx_bit_index);
                }
                uart_rx_bit_index++;
                uart_rx_cycle_count = 0;

                // 8个数据位采样完成
                if (uart_rx_bit_index >= 8) {
                    uart_rx_state = RX_STOP;
                }
            }
            break;

        case RX_STOP:
            uart_rx_cycle_count++;
            // 在停止位的中点采样
            if (uart_rx_cycle_count >= CYCLES_PER_BIT) {
                if (uart_tx == 1) {
                    // 停止位有效，接收完成，输出字符
                    uint8_t received_char = uart_rx_data_byte;
                    uart_rx_byte_count++;

                    // 过滤控制字符（除了换行、回车、制表符）
                    if ((received_char >= 32 && received_char <= 126) ||
                        received_char == '\n' ||
                        received_char == '\r' ||
                        received_char == '\t') {
                        putchar(received_char);
                        fflush(stdout);
                    }
#ifdef DEBUG_UART_RX
                    printf("[UART RX] Byte #%lu: 0x%02x ('%c') at time %lu\n",
                           uart_rx_byte_count, received_char,
                           (received_char >= 32 && received_char <= 126) ? received_char : '?',
                           main_time);
#endif
                } else {
                    // 停止位无效（帧错误）
                    uart_rx_error_count++;
                    printf("[UART RX ERROR] Frame error #%lu at time %lu (data=0x%02x, stop_bit=%d, cycles=%lu)\n",
                           uart_rx_error_count, main_time, uart_rx_data_byte, uart_tx, uart_rx_cycle_count);
                }
                // 返回 IDLE 状态，准备接收下一个字节
                uart_rx_state = RX_IDLE;
            }
            break;
    }

    // 保存当前 TX 状态用于下降沿检测
    last_uart_tx = uart_tx;
}

```

这样就能获取到 UART_TX 发来的数据了。

## 关于 UART 的几个问题

### 在 qemu 中，写 uart 的时候为什么不用涉及到波特率、频率的设置？

```C
/*
 * 初始化 UART
 * QEMU 已经配置好了，我们只需启用
 */
void bsp_uart_init(void) {
    /* QEMU virt 的 UART 已经初始化，我们不需要做太多 */
    UART_FCR = 0x07;  /* 使能 FIFO，清空 FIFO */
    UART_LCR = 0x03;  /* 8位数据，无校验，1停止位 */
    UART_IER = 0x00;  /* 禁用中断 (轮询模式) */
}
```

因为 QEMU 是高层次的软件模拟器，它抽象掉了底层的时序细节。QEMU 中的 UART 模拟直接工作在字节级别，不需要模拟实际的串行信号时序（起始位、数据位、停止位等）。用户程序向 UART 写数据时，QEMU 直接将字节传输到主机的标准输出或终端，省略了波特率和时钟频率相关的硬件细节。这种方式大大简化了模拟，但也意味着无法测试实际的串行通信时序问题。

### 波特率除数很关键，设置 divisor 的时候，究竟在设置什么？

这里需要理解一个关键点：**UART 硬件寄存器中存储的是除数，而不是波特率本身**。

在 `bsp_uart_init()` 函数中：

```C
int bsp_uart_init(uint8_t uart_id, uint32_t baudrate) {
    uint16_t divisor;

    /* 软件层面：接收波特率参数（如 9600） */
    divisor = UART_CLK / (16 * baudrate);  // 将波特率转换为除数

    /* 硬件层面：写入除数到寄存器 */
    UART_REG(UART_DLM) = (divisor >> 8) & 0xFF; /* 高字节 */
    UART_REG(UART_DLL) = divisor & 0xFF;        /* 低字节 */
}
```

**为什么只写除数而不直接写波特率？**

因为 UART 硬件控制器**不理解"9600 bps"这个概念**，它只能根据系统时钟进行分频。硬件的工作原理是：

```
串行信号速率 = 系统时钟频率 / (除数 × 16)
```

因此：
1. **软件的任务**：将用户想要的波特率（如 9600）转换成除数
2. **硬件的任务**：根据除数对系统时钟进行分频，生成对应速率的串行信号

**除数的作用**：

设置波特率除数本质上是配置 UART 硬件控制器内部的时钟分频器。UART 内部会用 16 倍的波特率进行过采样，以提高接收的可靠性和抗干扰能力。因此：

- **发送端**：除数决定了每个比特在 TX 线上保持的时间
- **接收端**：除数决定了采样 RX 线的时间间隔

**类比**：这就像你想让汽车以 60 km/h 行驶（波特率），但你需要调整油门踏板的位置（除数）来实现这个速度。你的目标是速度，但你操作的是踏板。

只有双方的实际波特率一致（或误差在允许范围内），才能保证数据正确传输。

### 那为什么 UART 设备（比如串口屏）只需要设置波特率，不用设置时钟频率？

因为 UART 设备的系统时钟频率在出厂时已经固定，其内部的 UART 控制器会根据设定的波特率自动计算并配置除数。用户只需要关心通信的波特率，设备内部会自动完成：

```
除数 = 系统时钟频率 / (波特率 × 16)
```

这个计算过程对用户透明。因此，用户只需要确保通信双方的波特率参数一致即可，不需要关心各自的内部时钟频率。

### 那如果两个设备的时钟频率不一样，但是波特率设置完全一样，能相互通信吗？

**完全可以通信！** 这里需要理解一个关键点：**波特率是最终的通信速率，与时钟频率无关**。

波特率定义的是串行线上每秒传输的比特数（bps），是实际的物理信号速率。不同设备可以使用不同的时钟频率，只要它们能够产生相同的波特率，就能够正常通信。

例如：

**设备 A（MCU）：**
- 系统时钟：20 MHz
- 波特率设置：9600 bps
- 除数计算：20000000 / (9600 × 16) = 130.208 ≈ 130
- 实际波特率：20000000 / (130 × 16) ≈ 9615 bps

**设备 B（串口屏）：**
- 系统时钟：16 MHz（与设备 A 不同）
- 波特率设置：9600 bps
- 除数计算：16000000 / (9600 × 16) = 104.167 ≈ 104
- 实际波特率：16000000 / (104 × 16) ≈ 9615 bps

虽然两个设备的时钟频率不同（20MHz vs 16MHz），但只要实际波特率接近（都在 9615 bps 左右），它们就能正常通信。

**需要注意的是**：由于除数只能是整数，实际波特率可能与设置值存在误差。只要误差在允许范围内（通常 ±2% 以内），通信就不会出现问题。这也是为什么标准波特率（9600、115200 等）的选择很重要，它们能在大多数时钟频率下产生较小的误差。




