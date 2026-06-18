---
title: "毕业设计记录（15）：JIT 技术调研与设计"
date: 2026-01-29 10:03:13
tags:
  - "graduation thesis"
  - "JIT"
  - "emulator"
categories:
  - "Thesis Project"
mermaid: true
---

## 问题
通过 NES 模拟器，运行 mario 的效率太低了，主要的时间花费在了图像渲染上：

```C
void am_gpu_fbdraw(int x, int y, uint32_t *pixels, int w, int h, bool sync) {
    volatile uint16_t *fb = hdmi_get_fb_pointer();

    // 边界检查
    if (x < 0 || y < 0 || x + w > 1920 || y + h > 1080) {
        return;
    }

    // 预计算基地址（y * 1920 使用移位）
    int y_offset = (y << 10) + (y << 9) + (y << 8) + (y << 7);
    volatile uint16_t *fb_row = fb + y_offset + x;
    uint32_t *src = pixels;

    // 预加载掩码到寄存器
    uint32_t mask_r = 0xF800;
    uint32_t mask_g = 0x07E0;
    uint32_t mask_b = 0x001F;

    // 使用内联汇编优化内循环
    for (int j = 0; j < h; j++) {
        uint32_t *src_ptr = src;
        volatile uint16_t *dst_ptr = fb_row;
        int count = w;

        // 内联汇编：直接从寄存器写入，避免栈操作
        __asm__ volatile("1:\n\t"
                         "ld.w    $t0, %3, 0\n\t"    // 加载 ARGB
                         "addi.w  %3, %3, 4\n\t"     // src++
                         "srli.w  $t1, $t0, 8\n\t"   // argb >> 8
                         "srli.w  $t2, $t0, 5\n\t"   // argb >> 5
                         "srli.w  $t3, $t0, 3\n\t"   // argb >> 3
                         "and     $t1, $t1, %6\n\t"  // R: & mask_r
                         "and     $t2, $t2, %7\n\t"  // G: & mask_g
                         "and     $t3, $t3, %8\n\t"  // B: & mask_b
                         "or      $t1, $t1, $t2\n\t" // R | G
                         "or      $t1, $t1, $t3\n\t" // R | G | B
                         "st.h    $t1, %4, 0\n\t"    // 直接写入16位
                         "addi.w  %4, %4, 2\n\t"     // dst++
                         "addi.w  %5, %5, -1\n\t"    // count--
                         "bne     $zero, %5, 1b\n\t" // 循环
                         : "=r"(src_ptr), "=r"(dst_ptr), "=r"(count)
                         : "0"(src_ptr), "1"(dst_ptr), "2"(count), "r"(mask_r),
                           "r"(mask_g), "r"(mask_b)
                         : "t0", "t1", "t2", "t3", "memory");
        src += w;
        fb_row += 1920;
    }
    if (sync) {
        hdmi_swap_buffers();
    }
}
```

就算是这样，还是很慢，差不多 15 秒一帧，这样是无法玩游戏的，通过反汇编发现，NES 模拟器中的指令模拟，每一条指令差不多都被模拟成了 20～30 条指令。而且本来 PPU 和 CPU 并行执行，模拟器是通过穿插交替执行来模拟并行执行，这样 CPU 推荐的就更慢了。

这里 JIT 技术就闪亮登场（可能会加快，但是也不要迷信 JIT）。

## JIT

JIT (Just-In-Time) 编译是一种在运行时将字节码或中间代码编译成本机机器码的技术。与传统解释器相比，JIT 可以显著提升执行效率。

### 解释器 vs JIT

| 方式 | 执行过程 | 优点 | 缺点 |
|------|----------|------|------|
| 解释器 | 取指令→解码→执行→循环 | 实现简单，启动快 | 每条指令都要重复解码 |
| JIT | 编译→执行本机代码 | 执行速度快 | 编译有开销，实现复杂 |

---

## JIT Demo：一个简单的例子

为了理解 JIT 的核心原理，我们先实现一个简单的虚拟机和对应的 JIT 编译器。

### 虚拟机定义

定义一个极简虚拟机：
- 1 个寄存器 A（累加器）
- 4 条指令

```C
#define OP_HALT  0x00  // 停止执行，返回 A
#define OP_LOAD  0x01  // LOAD n  : A = n
#define OP_ADD   0x02  // ADD n   : A = A + n
#define OP_JMP   0x03  // JMP addr: PC = addr
```

### 解释器实现

传统解释器通过 switch-case 循环执行：

```C
int vm_interpret(uint8_t *code, int max_cycles) {
    int pc = 0;
    int reg_a = 0;
    int cycles = 0;

    while (cycles < max_cycles) {
        uint8_t op = code[pc];

        switch (op) {
        case OP_HALT:
            return reg_a;

        case OP_LOAD:
            reg_a = code[pc + 1];
            pc += 2;
            break;

        case OP_ADD:
            reg_a += code[pc + 1];
            pc += 2;
            break;

        case OP_JMP:
            pc = code[pc + 1];
            break;
        }
        cycles++;
    }
    return reg_a;
}
```

每次执行都要：**取指令 → switch 判断 → 执行 → 循环**，开销很大。

### JIT 编译器实现

JIT 的核心思想：**把虚拟机代码翻译成本机代码，然后直接执行**。

#### LoongArch32R 指令编码

首先需要了解目标平台的指令编码：

```C
// ori rd, rj, imm12 : rd = rj | imm12
// 编码: 0x03800000 | (imm12 << 10) | (rj << 5) | rd
static uint32_t encode_ori(int rd, int rj, int imm12) {
    return 0x03800000 | ((imm12 & 0xFFF) << 10) | (rj << 5) | rd;
}

// addi.w rd, rj, imm12 : rd = rj + imm12
// 编码: 0x02800000 | (imm12 << 10) | (rj << 5) | rd
static uint32_t encode_addi_w(int rd, int rj, int imm12) {
    return 0x02800000 | ((imm12 & 0xFFF) << 10) | (rj << 5) | rd;
}

// jirl rd, rj, offset : rd = pc + 4; pc = rj + offset
static uint32_t encode_jirl(int rd, int rj, int offset) {
    return 0x4C000000 | (((offset >> 2) & 0xFFFF) << 10) | (rj << 5) | rd;
}

// b offset : pc = pc + offset
static uint32_t encode_b(int offset) {
    int off = offset >> 2;
    return 0x50000000 | ((off & 0xFFFF) | ((off >> 16) << 16));
}
```

#### 寄存器分配

```C
#define REG_ZERO  0   // $r0 = 0 (恒为零)
#define REG_RA    1   // $r1 = 返回地址
#define REG_T0    12  // $r12 = $t0 (虚拟机的 A 寄存器)
#define REG_A0    4   // $r4 = $a0 (函数返回值)
```

#### JIT 编译函数

```C
// JIT 代码缓冲区
static uint32_t jit_code_buffer[256];
static int jit_code_size = 0;

// 生成一条指令
static void emit(uint32_t instruction) {
    jit_code_buffer[jit_code_size++] = instruction;
}

typedef int (*jit_func_t)(void);
```

核心编译逻辑：

```C
jit_func_t jit_compile(uint8_t *code, int code_len) {
    jit_code_size = 0;

    // 记录虚拟机地址到本机代码位置的映射
    int addr_map[256];
    memset(addr_map, -1, sizeof(addr_map));

    // 需要回填的跳转指令
    struct {
        int native_pos;   // 本机代码位置
        int target_addr;  // 目标虚拟机地址
    } fixups[64];
    int num_fixups = 0;

    // 第一遍：生成代码
    int pc = 0;
    while (pc < code_len) {
        addr_map[pc] = jit_code_size;  // 记录映射
        uint8_t op = code[pc];

        switch (op) {
        case OP_HALT:
            // 返回：$a0 = $t0; return
            emit(encode_ori(REG_A0, REG_T0, 0));
            emit(encode_jirl(REG_ZERO, REG_RA, 0));
            pc += 1;
            break;

        case OP_LOAD:
            // A = n  =>  ori $t0, $zero, n
            emit(encode_ori(REG_T0, REG_ZERO, code[pc + 1]));
            pc += 2;
            break;

        case OP_ADD:
            // A += n  =>  addi.w $t0, $t0, n
            emit(encode_addi_w(REG_T0, REG_T0, code[pc + 1]));
            pc += 2;
            break;

        case OP_JMP:
            // 跳转：先占位，稍后回填
            fixups[num_fixups].native_pos = jit_code_size;
            fixups[num_fixups].target_addr = code[pc + 1];
            num_fixups++;
            emit(0);  // 占位
            pc += 2;
            break;
        }
    }

    // 第二遍：回填跳转地址
    for (int i = 0; i < num_fixups; i++) {
        int native_pos = fixups[i].native_pos;
        int target_native = addr_map[fixups[i].target_addr];
        int offset = (target_native - native_pos) * 4;
        jit_code_buffer[native_pos] = encode_b(offset);
    }

    // 返回函数指针
    return (jit_func_t)jit_code_buffer;
}
```

#### JIT 编译的关键点

1. **两遍扫描**：第一遍生成代码，第二遍回填跳转地址；
2. **地址映射**：`addr_map[vm_pc] = native_pos` 记录虚拟机地址到本机代码的对应关系；
3. **函数指针转换**：`(jit_func_t)jit_code_buffer` 将数据数组转为可执行函数。

---

### 编译示例

对于程序 `LOAD 1; ADD 2; ADD 3; HALT`：

```
虚拟机字节码:
  [0] 0x01 0x01    LOAD 1
  [2] 0x02 0x02    ADD 2
  [4] 0x02 0x03    ADD 3
  [6] 0x00         HALT
```

JIT 生成的 LoongArch32R 代码：

```
0x00:  ori   $t0, $zero, 1    ; A = 1
0x04:  addi.w $t0, $t0, 2     ; A = A + 2
0x08:  addi.w $t0, $t0, 3     ; A = A + 3
0x0c:  ori   $a0, $t0, 0      ; 返回值 = A
0x10:  jirl  $zero, $ra, 0    ; return
```

---

### 性能对比

```C
void jit_demo(void) {
    uint8_t prog[] = {
        OP_LOAD, 1,     // A = 1
        OP_ADD,  2,     // A = 3
        OP_ADD,  3,     // A = 6
        OP_HALT
    };

    // 编译一次
    jit_func_t fn = jit_compile(prog, sizeof(prog));

    // 测试 1000 万次迭代
    #define TEST_ITERATIONS 10000000

    // 解释器
    uint32_t start = timer_get_uptime_us();
    for (int i = 0; i < TEST_ITERATIONS; i++) {
        vm_interpret(prog, 100);
    }
    uint32_t time_interp = timer_get_uptime_us() - start;

    // JIT
    start = timer_get_uptime_us();
    for (int i = 0; i < TEST_ITERATIONS; i++) {
        fn();
    }
    uint32_t time_jit = timer_get_uptime_us() - start;

    printf("Interpreter: %u us\n", time_interp);
    printf("JIT: %u us\n", time_jit);
    printf("Speedup: ~%ux\n", time_interp / time_jit);
}
```

实际测试结果（在 LoongArch32R 上）：

```
Interpreter: 约 5000000 us
JIT:         约 500000 us
Speedup:     ~10x
```

JIT 版本快约 **10 倍**！

