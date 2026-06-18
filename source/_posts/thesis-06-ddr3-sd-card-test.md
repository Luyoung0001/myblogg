---
title: "毕业设计记录（6）：DDR3 与 SD 卡测试"
date: 2026-01-18 10:03:13
tags:
  - "graduation thesis"
  - "DDR3"
  - "SD card"
categories:
  - "Thesis Project"
mermaid: true
---

## 前言

SoC 搭建好了后，就可以测试 ddr3 了。我觉得这种测试思路很值得借鉴，尤其是在早期 SoC 的开发过程中。

首先我的设计目标是：
- 上电并且 reset 释放后，CPU 的 PC 直接访问 Boot ROM；
- Boot ROM 中放着一个bootloader，它的作用是从 sd 卡中搬运程序数据到 ddr3 中；
- 搬运完后，CPU 跳转到 ddr3 中执行目标程序。

以上要确保：
- 硬件上： Boot ROM 访问正常、sd 卡扇区读取正常、ddr3 访问正常，axi 等各种数据通道正常；
- 软件上：bootloader 要工作正常。

## bootloader 测试

目前只有两个 led 可以用来协助 debug。只需要写一个电灯的 bootloader 就可以了。这个可以根据 led 的反应来确认 CPU 是启动正常、 Boot ROM 读取正常。

## ddr3 测试

上面的测试通过后，就可以测试 ddr3 了。思路是将 bootloader 中的代码搬运到 ddr3 然后跳转到 ddr3：

```asm
#==============================================================================
# DDR 代码段（会被复制到 DDR 0x00000000 执行）
# 注意：这段代码必须是位置无关的，使用绝对地址访问外设
#==============================================================================
.align 4
ddr_code_start:
    # 重新初始化 LED 寄存器基址（因为在 DDR 中执行，不能依赖寄存器值）
    lu12i.w $t0, 0x1FD0F
    ori     $t0, $t0, 0x000         # 0x1FD0F000

ddr_blink_loop:
    # LED1 亮
    li.w    $t1, 0x2
    st.w    $t1, $t0, 0

    # 延时 0.1s
    lu12i.w $t2, 0x7A
    ori     $t2, $t2, 0x120
ddr_delay_on:
    addi.w  $t2, $t2, -1
    bne     $t2, $zero, ddr_delay_on

    # LED1 灭
    li.w    $t1, 0x0
    st.w    $t1, $t0, 0

    # 延时 0.1s
    lu12i.w $t2, 0x7A
    ori     $t2, $t2, 0x120
ddr_delay_off:
    addi.w  $t2, $t2, -1
    bne     $t2, $zero, ddr_delay_off

    # 无限循环
    b       ddr_blink_loop

.align 4
ddr_code_end:

.end
```

只要观察到 ddr3 的代码对应的 led 闪烁情况就可以认为 ddr3 访问写、读正常。

## sd 测试

这个测试很复杂，因为一旦测试失败，就得更加精细化被测试的信号：
```txt
# SD Card Test Program for LoongArch32R SoC
#
# 功能：
#   1. 等待 SD 卡初始化完成（LED0 闪烁，0.1s间隔）
#   2. 初始化完成后，两灯都亮 2 秒
#   3. 读取扇区 0 的 512 字节（MBR）
#   4. 校验偏移 510-511 字节是否为 0x55, 0xAA（MBR 签名）
#   5. 显示结果：
#      - 成功 (0x55AA) → LED1 闪烁（0.1s间隔）
#      - 失败 → 两灯快闪（0.05s间隔）
#
# 测试判据：
#   - LED0 一直闪烁 → SD 卡未初始化（硬件问题/无卡）
#   - LED0 闪 → 两灯亮 → 两灯快闪 → SD 卡读取失败
#   - LED0 闪 → 两灯亮 → LED1 闪 → SD 卡完全正常！
```

测试失败后，需要检测内部状态：

```txt
# SD Card Diagnostic Program
#
# 测试流程：
#   阶段1: LED0 闪 2 次（确认启动）
#   阶段2: 检查 SD 初始化状态
#          - init_done=0 → LED0 常亮（SD 未初始化）
#          - init_done=1 → 继续
#   阶段3: 检查 SD busy 状态
#          - busy=1 → LED1 常亮（SD 忙）
#          - busy=0 → 继续
#   阶段4: 发送读取命令，检查 busy 变化
#          - busy 没变成 1 → 两灯交替闪（命令没被接收）
#          - busy 变成 1 → 继续
#   阶段5: 等待 data_ready
#          - 超时无数据 → LED0 快闪（没收到数据）
#          - 有数据 → 继续
#   阶段6: 读取第一个字节并显示
#          - LED 显示数据的低 2 位
#   阶段7: 等待 read_done
#          - 成功 → 两灯常亮
#          - 超时 → 两灯快闪
#
# LED 含义：
#   - LED0 常亮: SD 卡未初始化
#   - LED1 常亮: SD 卡一直忙
#   - 两灯交替闪: 读取命令没被接收
#   - LED0 快闪: 等待数据超时
#   - 两灯常亮: 测试完成
#   - 两灯快闪: read_done 超时
```
测试 2：

```txt
# SD Card Diagnostic Program V2
#
# 测试流程：
#   1. LED0 闪 2 次（确认启动）
#   2. 等待 init_done=1（最多 2 秒）
#   3. 检查 busy=0
#   4. 写入 SD_CTRL=1，然后立即读回 SD_CTRL 检查 sd_rd_start_reg
#   5. 多次读取 SD_STATUS，检查 busy 是否变化
#   6. 显示结果
#
# LED 含义：
#   - LED0 常亮: SD 卡未初始化（Phase 2 超时）
#   - LED1 常亮: SD 卡一直忙（Phase 3 失败）
#   - LED0 快闪: sd_rd_start_reg 没有被设置（写入失败）
#   - LED1 快闪: sd_rd_start_reg 设置了但 busy 没变（sd_rd 没响应）
#   - 两灯交替闪: busy 变成 1 了，继续等待 data_ready
#   - 两灯常亮: 测试完成（成功读到数据）
```

如果继续错误，还要检查 FIFO：

```txt
# SD Card FIFO Test - 测试 FIFO 读取功能
#
# 测试流程：
#   1. 等待 SD 初始化完成
#   2. 发送读取命令
#   3. 等待 read_done
#   4. 用 LED 显示 FIFO 中的字节数（sd_fifo_count 的低 2 位）
#   5. 读取字节 510 和 511，用 LED 显示
#
# LED 含义：
#   - LED0 闪烁: 等待初始化
#   - 两灯都亮 2 秒: 初始化完成
#   - LED 显示数字: 显示数据
```

每一次修改完代码（bootloader、verilog）后，都得重新综合，因此比较费时间。

## 从 sd 卡中加载 xOS
测试结束后，就可以尝试从 sd 卡中加载 xOS 了。最终会跳转到 shell 中，只要观察到led 轮流点亮，就可以认为执行成功了：

```C
void shell_run(void) {
    uart_display_printf(0,0,"\n");
    uart_display_printf(0,0,"========================================\n");
    uart_display_printf(0,0,"  xOS - Simple Operating System\n");
    uart_display_printf(0,0,"  for LoongArch32R SoC\n");
    uart_display_printf(0,0,"========================================\n");
    uart_display_printf(0,0,"\n");
    uart_display_printf(0,0,"Type 'help' for available commands.\n");
    uart_display_printf(0,0,"\n");

    shell_print_prompt();

    while (1) {
        blink_led(LED0, 1);  /* LED0 亮1下 */
        blink_led(LED1, 1);  /* LED1 亮1下 */
        int scancode = kb_get_scancode();
        // printf("0x%x ",scancode);
        if (scancode >= 0) {
            process_scancode((uint8_t)scancode);
        }
    }
}
```
最终也是成功！







