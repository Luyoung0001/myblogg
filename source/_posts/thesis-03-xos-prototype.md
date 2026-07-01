---
title: "毕业设计记录（3）：xOS 雏形与启动流程"
date: 2026-01-11 10:03:13
tags:
  - "graduation thesis"
  - "xOS"
  - "boot"
categories:
  - "Thesis Project"
---

## xOS
上节仿真系统顺利从 uart 截获数据并顺利打印出字符，这节依赖于此，我要设计 xOS。

目前的 xOS 只支持栈区，也就是说它不支持堆区内存的分配。

## 启动代码

启动函数做 4 件事情：
- 设置栈指针
- 清零 BSS 段
- 跳到 main
- 从 main 返回（实际上没必要返回，因为 main 返回之前可以设置一个死循环）

因此，启动代码大致如下：
```asm
    .section .text.start
    .globl _start
    .type _start, @function

_start:
    /* ======== 1. 设置栈指针 ======== */
    la.local    $sp, __stack_top

    /* ======== 2. 清零 BSS 段 ======== */
    la.local    $t0, __bss_start
    la.local    $t1, __bss_end

.L_clear_bss:
    beq         $t0, $t1, .L_bss_done
    st.w        $zero, $t0, 0
    addi.w      $t0, $t0, 4
    b           .L_clear_bss

.L_bss_done:
    /* ======== 3. 调用 main ======== */
    bl          main

    /* ======== 4. main 返回后死循环 ======== */
.L_halt:
    b           .L_halt
    .size _start, . - _start
```

这段启动代码很好理解。

## main
main 函数是 xOS 的核心，它会初始化有一些配置，比如 uart、中断注册等，之后就会进入到 shell。大致如下：

```C
int main(void) {
    myprintf("now in %s\n", __func__);
    system_init();
    shell_run();
    while (1) {
    }
    return 0;
}
```

## 系统初始化

### uart
首先是 uart 的初始化，因为我们要给 uart 输送各种调试信息，因此它要最先初始化。上一节已经讨论过，这节不在赘述。

### trap 初始化
首先这是一段 myCPU 的前端的代码：
```verilog
    assign nextpc =
           refetch_sign_i ? refetch_pc_i :
           excp_tlbrefill_flush ? csr_tlbrentry :
           excp_flush ? csr_eentry:
           ertn_flush ? inst_flush_pc:
           br_taken && br_true ? br_target :
           seq_pc;
```
由于xOS 不支持虚拟存储系统，因为 tlb 异常永远不会触发。这样看来，不管是异常还是中断，走的都是 `csr_eentry`。这里可以看出不管是中断还是异常，其实都是 trap。

### trap_entry 的设计
当程序在执行的时候，突然来了一个中断或者异常，会怎么办呢？思路就是，保存被中断打断的程序的所有的相关的状态。当trap 的时候，硬件会自动将被打断的指令的 pc 保存到 `csr_era` 中，并且同时会关闭中断：

```Verilog
        else if (excp_flush) begin
            csr_crmd[ `PLV] <=  2'b0; // 例外发生时，将权限打到最高，关闭中断
            csr_crmd[  `IE] <=  1'b0;
            if (excp_tlbrefill) begin
                csr_crmd [`DA] <= 1'b1;
                csr_crmd [`PG] <= 1'b0;
            end
        end
```

核心问题是，处理器会跳转到 trap_entry。此时所有通用寄存器都保持着异常发生前的值，我们需要保存这些寄存器到栈上，但是我们需要临时寄存器来完成保存操作本身。这就产生了一个"鸡生蛋"的问题：
- 要保存寄存器，需要用到临时寄存器（比如 t0）
- 但使用 t0 会破坏它原来的值
- 我们又需要保存 t0 的原始值

解决方案就是使用 CSR 作为临时存储：

```asm
  csrwr   $t0, CSR_SAVE0      /* 先把 t0 的原始值保存到 CSR_SAVE0 */
  csrwr   $sp, CSR_SAVE1      /* 同时保存 sp 的原始值到 CSR_SAVE1 */
```
现在 t0 和 sp 可以自由使用了，因为它们的原始值已经安全保存在 CSR 中。

既然 sp、t0 已经保存，接下来就可以使用它们来完成保存上下文的工作了。现在产生了一个新的问题要保存哪些寄存器？
- CSR_ERA     /* 异常返回地址 - 必须保存，防止处理中断的时候再次修改 */
- CSR_PRMD    /* 异常前的特权级和中断使能状态 - 必须保存，因为处理中断的时候是关中断的，要修改它，处理完得恢复 */
- CSR_ESTAT   /* 异常状态（异常类型、中断pending）- 建议保存 */

其它 csr 可以不用管，因为这是裸机系统，接下来建议保存所有的通用寄存器：

```asm
    /* Allocate trap frame on stack */
    addi.w  $sp, $sp, -TF_SIZE

    /* Save general purpose registers */
    st.w    $ra,  $sp, TF_RA
    st.w    $tp,  $sp, TF_TP
    /* sp will be saved later from SAVE1 */
    st.w    $a0,  $sp, TF_A0
    st.w    $a1,  $sp, TF_A1
    st.w    $a2,  $sp, TF_A2
    st.w    $a3,  $sp, TF_A3
    st.w    $a4,  $sp, TF_A4
    st.w    $a5,  $sp, TF_A5
    st.w    $a6,  $sp, TF_A6
    st.w    $a7,  $sp, TF_A7
    /* t0 will be saved later from SAVE0 */
    st.w    $t1,  $sp, TF_T1
    st.w    $t2,  $sp, TF_T2
    st.w    $t3,  $sp, TF_T3
    st.w    $t4,  $sp, TF_T4
    st.w    $t5,  $sp, TF_T5
    st.w    $t6,  $sp, TF_T6
    st.w    $t7,  $sp, TF_T7
    st.w    $t8,  $sp, TF_T8
    st.w    $r21, $sp, TF_R21
    st.w    $fp,  $sp, TF_FP
    st.w    $s0,  $sp, TF_S0
    st.w    $s1,  $sp, TF_S1
    st.w    $s2,  $sp, TF_S2
    st.w    $s3,  $sp, TF_S3
    st.w    $s4,  $sp, TF_S4
    st.w    $s5,  $sp, TF_S5
    st.w    $s6,  $sp, TF_S6
    st.w    $s7,  $sp, TF_S7
    st.w    $s8,  $sp, TF_S8

    /* Restore and save t0 from SAVE0 */
    csrrd   $t0, CSR_SAVE0
    st.w    $t0,  $sp, TF_T0

    /* Restore and save original sp from SAVE1 */
    csrrd   $t0, CSR_SAVE1
    st.w    $t0,  $sp, TF_SP

    /* Save CSR values */
    csrrd   $t0, CSR_ERA
    st.w    $t0,  $sp, TF_ERA

    csrrd   $t0, CSR_PRMD
    st.w    $t0,  $sp, TF_PRMD

    csrrd   $t0, CSR_ESTAT
    st.w    $t0,  $sp, TF_ESTAT
```

接着就可以跳转到 trap_dispatch 中进行处理了：

```asm
    /*
     * Call C trap handler
     * Argument: $a0 = pointer to trap frame (current sp)
     */
    move    $a0, $sp
    bl      trap_dispatch
```
### trap_dispatch()

a0 作为 trap_dispatch 的第一个参数，在 trap_dispatch 中进行处理的时候，利用 C 语言强大的指针转义直接可以访问刚刚保存的 trap_frame：

```C
typedef struct {
    uint32_t ra;        /* $r1  - Return Address */
    uint32_t tp;        /* $r2  - Thread Pointer */
    uint32_t sp;        /* $r3  - Stack Pointer */
    uint32_t a0;        /* $r4  - Argument 0 */
    uint32_t a1;        /* $r5  - Argument 1 */
    uint32_t a2;        /* $r6  - Argument 2 */
    uint32_t a3;        /* $r7  - Argument 3 */
    uint32_t a4;        /* $r8  - Argument 4 */
    uint32_t a5;        /* $r9  - Argument 5 */
    uint32_t a6;        /* $r10 - Argument 6 */
    uint32_t a7;        /* $r11 - Argument 7 */
    uint32_t t0;        /* $r12 - Temporary 0 */
    uint32_t t1;        /* $r13 - Temporary 1 */
    uint32_t t2;        /* $r14 - Temporary 2 */
    uint32_t t3;        /* $r15 - Temporary 3 */
    uint32_t t4;        /* $r16 - Temporary 4 */
    uint32_t t5;        /* $r17 - Temporary 5 */
    uint32_t t6;        /* $r18 - Temporary 6 */
    uint32_t t7;        /* $r19 - Temporary 7 */
    uint32_t t8;        /* $r20 - Temporary 8 */
    uint32_t r21;       /* $r21 - Reserved */
    uint32_t fp;        /* $r22 - Frame Pointer */
    uint32_t s0;        /* $r23 - Saved 0 */
    uint32_t s1;        /* $r24 - Saved 1 */
    uint32_t s2;        /* $r25 - Saved 2 */
    uint32_t s3;        /* $r26 - Saved 3 */
    uint32_t s4;        /* $r27 - Saved 4 */
    uint32_t s5;        /* $r28 - Saved 5 */
    uint32_t s6;        /* $r29 - Saved 6 */
    uint32_t s7;        /* $r30 - Saved 7 */
    uint32_t s8;        /* $r31 - Saved 8 */
    /* CSR values */
    uint32_t era;       /* Exception Return Address */
    uint32_t prmd;      /* Pre-exception Mode */
    uint32_t estat;     /* Exception Status */
} trap_frame_t;
```

其中保存的顺序要和 trap_frame 顺序严格一直，不然会给访问徒增难度：

```C
#define ESTAT_IS_MASK   0x1FFF    /* Interrupt Status [12:0] */
#define ESTAT_ECODE_SHIFT 16
#define ESTAT_ECODE_MASK  0x3F    /* Exception Code [21:16] */

void trap_dispatch(trap_frame_t *tf) {
    myprintf("now in %s\n", __func__);
    uint32_t ecode = (tf->estat >> ESTAT_ECODE_SHIFT) & ESTAT_ECODE_MASK;
    if (ecode == ECODE_INT) {
        interrupt_handler(tf);
    } else {
        exception_handler(tf);
    }
}
```
ecode 在硬件中是这样定义的：

```verilog
//ESTAT
`define IS        12:0
`define ECODE     21:16
`define ESUBCODE  30:22

    if (excp_flush) begin
        // 这里很关键，记录着各个中断的类型
        // 包括时钟中断等等
        csr_estat[`ECODE] <= csr_ecode;
        csr_estat[`ESUBCODE] <= csr_esubcode;
    end
```

所以我们通过移位+掩码运算就能解析出 ecode。在 loongarch32r 中，如果是中断 ecode 就是 0，异常的话各不相同：

```verilog
    // 检测异常
    assign {csr_ecode,
            va_error,
            bad_va,
            csr_esubcode,
            excp_tlbrefill,
            excp_tlb,
            excp_tlb_vppn} =
           ws_excp_num[7] || (inst_idle_i_r && has_int) ? {`ECODE_INT , 1'b0, 32'b0, 9'b0, 1'b0, 1'b0, 19'b0}:
           ws_excp_num[0] ? {`ECODE_ADEF, wbu_valid, wire_pc, `ESUBCODE_ADEF, 1'b0, 1'b0, 19'b0} :
           ws_excp_num[1] ? {`ECODE_TLBR, wbu_valid, wire_pc, 9'b0, wbu_valid, wbu_valid, wire_pc[31:13]} :
           ws_excp_num[2] ? {`ECODE_PIF , wbu_valid, wire_pc, 9'b0, 1'b0, wbu_valid, wire_pc[31:13]} :
           ws_excp_num[3] ? {`ECODE_PPI , wbu_valid, wire_pc, 9'b0, 1'b0, wbu_valid, wire_pc[31:13]} :
           ws_excp_num[4] ? {`ECODE_SYS, 1'b0, 32'b0, 9'b0, 1'b0, 1'b0, 19'b0}:
           ws_excp_num[6] ? {`ECODE_BRK , 1'b0, 32'b0, 9'b0, 1'b0, 1'b0, 19'b0} :
           ws_excp_num[5] ? {`ECODE_INE , 1'b0, 32'b0, 9'b0, 1'b0, 1'b0, 19'b0} :
           ws_excp_num[8] ? {`ECODE_IPE , 1'b0, 32'b0, 9'b0, 1'b0, 1'b0, 19'b0} :
           ws_excp_num[9] ? {`ECODE_ALE , wbu_valid, wire_error_va, 9'b0, 1'b0, 1'b0, 19'b0} :
           ws_excp_num[10] ? {`ECODE_TLBR, wbu_valid, wire_error_va, 9'b0, wbu_valid, wbu_valid, wire_error_va[31:13]} :
           ws_excp_num[14] ? {`ECODE_PME , wbu_valid, wire_error_va, 9'b0, 1'b0, wbu_valid, wire_error_va[31:13]} :
           ws_excp_num[13] ? {`ECODE_PPI , wbu_valid, wire_error_va, 9'b0, 1'b0, wbu_valid, wire_error_va[31:13]} :
           ws_excp_num[12] ? {`ECODE_PIS , wbu_valid, wire_error_va, 9'b0, 1'b0, wbu_valid, wire_error_va[31:13]} :
           ws_excp_num[11] ? {`ECODE_PIL , wbu_valid, wire_error_va, 9'b0, 1'b0, wbu_valid, wire_error_va[31:13]} :
           69'b0;
```

因此只需要根据 ecode 就能区分中断和各个异常。假设是一个中断后，进入中断处理函数。这里先说一下中断处理相关的几个寄存器。

#### csr_estat
ESTAT（Exception Status）寄存器包含：
- IS 字段（Interrupt Status）：哪些中断正在 pending（待处理）
- Ecode 字段：异常类型编码
- EsubCode 字段：异常子类型

这里主要关心 IS 字段，它是一个位图，每一位代表一个中断源：
- bit 0: 软件中断 0
- bit 1: 软件中断 1
- bit 2-9: 硬件中断 0-7
- bit 10: 定时器中断
- bit 11: 性能计数器中断
- bit 12: IPI 中断
- ...

#### csr_ecfg

LIE 字段（Local Interrupt Enable）：本地中断使能位图。局部中断使能位，高有效。这些局部中断使能位与 CSR.ESTAT 中 IS[12:0]域记录的 13 个中断源一一对应，每一位控制一个中断源。

### 中断管理

这是本系统设计的 IRQ 标号：
```C
#define IRQ_SWI0        0       /* Software Interrupt 0 */
#define IRQ_SWI1        1       /* Software Interrupt 1 */
#define IRQ_HWI0        2       /* Hardware Interrupt 0 (unused) */
#define IRQ_UART        3       /* UART interrupt (HWI1, bit 3) */
#define IRQ_PS2         4       /* PS2 keyboard interrupt (HWI2, bit 4) */
#define IRQ_HWI3        5       /* Hardware Interrupt 3 */
#define IRQ_HWI4        6       /* Hardware Interrupt 4 */
#define IRQ_HWI5        7       /* Hardware Interrupt 5 */
#define IRQ_HWI6        8       /* Hardware Interrupt 6 */
#define IRQ_HWI7        9       /* Hardware Interrupt 7 */
#define IRQ_PMI         10      /* Performance Monitor Interrupt */
#define IRQ_TIMER       11      /* Timer interrupt */
#define IRQ_IPI         12      /* Inter-Processor Interrupt */
```
可以看到，IRQ_PS2 是 4 号中断。在注册的时候：
```C
void irq_register(int irq, irq_handler_t handler) {
    myprintf("now in %s\n", __func__);
    if (irq >= 0 && irq < IRQ_MAX) {
        irq_handlers[irq] = handler ? handler : default_irq_handler;
    }
}
```
会把 `handler` 放到对应的 `irq_handlers[]`，这样 interrupt_handler 就会执行到：

```C
static void interrupt_handler(trap_frame_t *tf) {
    myprintf("now in %s\n", __func__);
    uint32_t estat = tf->estat;
    uint32_t ecfg = read_csr_ecfg();
    /* find out pending IRQ */
    uint32_t pending = estat & ecfg & ESTAT_IS_MASK;

    /* Check each interrupt source */
    for (int i = 0; i < IRQ_MAX; i++) {
        if (pending & (1 << i)) {
            if (irq_handlers[i]) {
                irq_handlers[i]();
            }
        }
    }
}
```

假设这里是一个 `ps2_irq_handler`，

```C
static void ps2_irq_handler(void) {
    myprintf("now in %s\n", __func__);
    /* Read all available scancodes */
    while (bsp_ps2_data_available()) {
        uint8_t scancode = (uint8_t)bsp_ps2_read();
        int next = (kb_head + 1) % KB_BUF_SIZE;

        /* Store if buffer not full */
        if (next != kb_tail) {
            kb_buffer[kb_head] = scancode;
            kb_head = next;
        }
    }
}
```

这个 handler 会在 `bsp_ps2_data_available()` 的时候，将 ps2 的数据全部读取到 kb_buffer。然后就返回并恢复现场：

```asm

    /*
     * Restore CSR values
     */
    ld.w    $t0,  $sp, TF_ERA
    csrwr   $t0, CSR_ERA

    ld.w    $t0,  $sp, TF_PRMD
    csrwr   $t0, CSR_PRMD

    /*
     * Restore general purpose registers
     */
    ld.w    $ra,  $sp, TF_RA
    ld.w    $tp,  $sp, TF_TP
    /* Don't restore sp yet - we need it for the frame */
    ld.w    $a0,  $sp, TF_A0
    ld.w    $a1,  $sp, TF_A1
    ld.w    $a2,  $sp, TF_A2
    ld.w    $a3,  $sp, TF_A3
    ld.w    $a4,  $sp, TF_A4
    ld.w    $a5,  $sp, TF_A5
    ld.w    $a6,  $sp, TF_A6
    ld.w    $a7,  $sp, TF_A7
    ld.w    $t0,  $sp, TF_T0
    ld.w    $t1,  $sp, TF_T1
    ld.w    $t2,  $sp, TF_T2
    ld.w    $t3,  $sp, TF_T3
    ld.w    $t4,  $sp, TF_T4
    ld.w    $t5,  $sp, TF_T5
    ld.w    $t6,  $sp, TF_T6
    ld.w    $t7,  $sp, TF_T7
    ld.w    $t8,  $sp, TF_T8
    ld.w    $r21, $sp, TF_R21
    ld.w    $fp,  $sp, TF_FP
    ld.w    $s0,  $sp, TF_S0
    ld.w    $s1,  $sp, TF_S1
    ld.w    $s2,  $sp, TF_S2
    ld.w    $s3,  $sp, TF_S3
    ld.w    $s4,  $sp, TF_S4
    ld.w    $s5,  $sp, TF_S5
    ld.w    $s6,  $sp, TF_S6
    ld.w    $s7,  $sp, TF_S7
    ld.w    $s8,  $sp, TF_S8

    /* Restore sp (deallocate trap frame) */
    addi.w  $sp, $sp, TF_SIZE

    /* Return from exception */
    ertn
```

然后就回到被中断的指令处：

```C
    /* Main loop - keyboard input comes from interrupt buffer */
    while (1) {
        int scancode = kb_get_scancode();
        // myprintf("0x%x ",scancode);
        if (scancode >= 0) {
            process_scancode((uint8_t)scancode);
        }
    }
```
这样，就完成了被中断、中断处理、中断返回并继续执行被打断的指令的过程。

### 执行效果

可以看到，xOS 工作正常，可以通过 ps2 进行输入，并且中断处理、中断退出等都能正常工作：
```txt
========================================

Type 'help' for available commands.

xos> s
Unknown command: s
Type 'help' for available commands.
xos> help
Available commands:
  %-10s - help
  %-10s - echo
  %-10s - clear
  %-10s - info
xos> info
xOS System Information
----------------------
CPU:      LoongArch32R
Board:    A7-Lite FPGA
RAM:      256MB
Keyboard: PS2 (Interrupt-driven)
Serial:   UART 115200 8N1
xos>
```

注：由于这个系列文章是我的毕业设计，我之所以可以在答辩前大胆暴露很多代码，是因为我在关键之处埋了大坑但是丝毫不影响知识的传播。