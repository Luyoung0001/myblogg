---
title: "毕业设计记录（16）：用 JIT 运行 NES 指令"
date: 2026-01-31 10:03:13
tags:
  - "graduation thesis"
  - "JIT"
  - "NES"
categories:
  - "Thesis Project"
mermaid: true
---

## 把 JIT 接入 NES 指令执行路径

上一篇先把 JIT 的基本原理和一个小 demo 跑通了。这一篇开始把它真正接到毕业设计项目里：让 `xOS Pro Max` 上移植的 LiteNES 模拟器，在 LoongArch32R 软核上运行 NES 指令时，可以从传统解释器路径切换到动态生成的本机代码路径。

这件事看起来像是“给模拟器加一个优化开关”，但实际做下来更像是在系统里加一条新的执行通道。NES 的 CPU 指令不是孤立计算，读写地址会牵出 PPU 寄存器、OAM DMA、手柄输入、Mapper、调色板等副作用。如果 JIT 只追求把 `switch(opcode)` 翻译成机器码，很容易跑得更快，也错得更快。

所以本项目当前采用的是一个比较稳的中间形态：**单指令直译 JIT**。每个 6502 PC 编译成一段 LoongArch32R 机器码，能安全原生执行的指令直接生成本机代码，复杂或暂未覆盖的指令则生成一个调用解释器单步函数的 fallback stub。

## 代码位置

软件侧 NES/JIT 主要在 `software/xos_pro_max` 中：

| 路径 | 作用 |
|------|------|
| `software/xos_pro_max/Makefile` | 把 LiteNES 源码和 `jit_simple.c` 纳入 `xos` 构建 |
| `software/xos_pro_max/src/litenes/cpu.c` | 6502 解释器、指令表、`cpu_run()` 主循环 |
| `software/xos_pro_max/src/litenes/jit_simple.c` | 当前 JIT 主实现 |
| `software/xos_pro_max/src/litenes/la_emit.h` | LoongArch32R 指令编码助手 |
| `software/xos_pro_max/src/litenes/memory.c` | NES CPU 地址空间读写路由 |
| `software/xos_pro_max/src/shell.c` | `jitmode`、`difftest` 等 shell 命令入口 |

`Makefile` 中的 LiteNES 源文件列表已经包含了 `src/litenes/jit_simple.c`，因此它不是独立 demo，而是直接进入系统镜像：

```makefile
LITENES_SRCS := src/litenes/common.c \
                src/litenes/cpu.c \
                src/litenes/cpu-addressing.c \
                src/litenes/fce.c \
                src/litenes/memory.c \
                src/litenes/mmc.c \
                src/litenes/ppu.c \
                src/litenes/psg.c \
                src/litenes/mario-rom.c \
                src/litenes/am_adapter.c \
                src/litenes/jit_simple.c
```

## 原来的解释器路径

解释器入口在 `cpu.c` 的 `cpu_run(long cycles)`。PPU 每条扫描线会分段调用 `cpu_run()`，模拟 NES 中 CPU 与 PPU 的比例关系。当前代码里先把传入周期除以 3，然后在循环里不断执行 6502 指令：

```c
void cpu_run(long cycles) {
  cycles /= 3;
  while (cycles > 0) {
    int used = 0;
    if (jit_enabled) {
      used = jit_run(cpu.PC);
    }
    if (!jit_enabled || used <= 0) {
      op_code = memory_readb(cpu.PC++);
      if (cpu_op_address_mode[op_code] != NULL) {
        cpu_op_address_mode[op_code]();
        cpu_op_handler[op_code]();
      }
      used = cpu_op_cycles[op_code] + op_cycles;
      op_cycles = 0;
    }
    cycles -= used;
    cpu_cycles -= used;
  }
}
```

这段代码里最关键的变化是：解释器执行之前先尝试 `jit_run(cpu.PC)`。如果 JIT 返回了正的周期数，就说明这条 6502 指令已经通过 JIT 路径完成执行；如果返回 0 或失败，再落回解释器。

解释器本身仍然非常重要。它既是正确性基准，也是 JIT 暂未覆盖指令的后备实现。项目里还额外提供了 `cpu_step_interp(uint16_t pc)`，用于从指定 PC 执行一条解释器指令，这正好给 JIT fallback stub 使用。

## 当前 JIT 的设计取舍

最初设想过块级 JIT：从某个 PC 开始连续翻译一个 basic block，直到分支、跳转、JSR/RTS 或 I/O 边界为止。这个方向性能潜力更大，但调试成本也更高，尤其是在 NES 这种大量依赖内存映射副作用的系统里。

当前实现选择先跑通一个更保守的版本：

1. 每个 PC 单独缓存一段机器码；
2. 每段机器码只负责一条 6502 指令；
3. 原生模板统一通过 `memory_readb()` / `memory_writeb()` 访问 NES 地址空间；
4. 未覆盖 opcode 编译成 `cpu_step_interp(pc)` 调用；
5. 每次生成代码后调用 `__builtin___clear_cache()` 刷新 I-cache。

对应数据结构在 `jit_simple.c` 顶部：

```c
#define JIT_CODE_PER_ENTRY 64

typedef int (*jit_func_t)(void);

static jit_func_t pc_func[1 << 16] LITENES_NOINIT;
static uint8_t pc_len[1 << 16] LITENES_NOINIT;
static uint32_t code_pool[1 << 16][JIT_CODE_PER_ENTRY] LITENES_NOINIT;
```

这里没有采用哈希表，而是直接用 16 位 PC 做下标。好处是查找极其简单：`pc_func[pc]` 要么是已经编译好的函数指针，要么为空。代价是代码池比较大，所以这些数组放在 `.noinit.litenes` 段里，避免普通 `.bss` 初始化拖慢启动。

## JIT 执行流程

整体流程如下：

```mermaid
flowchart TD
  A[PPU/frame loop] --> B[cpu_run cycles]
  B --> C{jit_enabled?}
  C -->|否| I[解释器取指/译码/执行]
  C -->|是| D[jit_run cpu.PC]
  D --> E{pc_func[PC] hit?}
  E -->|miss| F[jit_compile PC]
  E -->|hit| G[调用已生成机器码]
  F --> H[写入 code_pool 并刷新 I-cache]
  H --> G
  G --> J{返回 cycles > 0?}
  J -->|是| K[扣减周期]
  J -->|否| I
  I --> K
```

`jit_run()` 负责命中检查、编译、统计和调用：

```c
int jit_run(uint16_t pc) {
    if (!jit_enabled)
        return 0;

    stat_calls++;
    jit_func_t fn = pc_func[pc];
    if (!fn) {
        stat_misses++;
        uint8_t len = 0;
        fn = jit_compile(pc, &len);
        if (!fn)
            return 0;
        pc_func[pc] = fn;
    } else {
        stat_hits++;
    }

    jit_cur_pc = pc;
    return fn();
}
```

这段代码还有一个很实用的调试价值：可以通过 `stat_calls`、`stat_hits`、`stat_misses`、`stat_invalid` 观察 JIT 是否真的被执行，而不是只是 shell 里打开了开关。

## 如何生成 LoongArch32R 机器码

JIT 不依赖汇编器，而是直接把 LoongArch32R 指令编码写进 `uint32_t` 数组。`la_emit.h` 封装了常用指令，例如：

```c
static inline uint32_t la_ori(int rd, int rj, int imm12) {
    return 0x03800000 | ((imm12 & 0xFFF) << 10) | (rj << 5) | rd;
}

static inline uint32_t la_jirl(int rd, int rj, int imm16) {
    return 0x4C000000 | ((imm16 & 0xFFFF) << 10) | (rj << 5) | rd;
}

static inline int emit_li32(uint32_t *buf, int pos, int rd, uint32_t imm) {
    buf[pos++] = la_lu12i_w(rd, imm >> 12);
    buf[pos++] = la_ori(rd, rd, imm & 0xFFF);
    return pos;
}
```

每个 JIT 模板都是往 `code_pool[pc]` 里追加 32 位机器码。模板开头会保存返回地址和 `s0`，并把 `s0` 固定用作 `cpu` 结构体指针：

```c
#define REG_CPU 23

static inline int emit_prologue(uint32_t *buf, int p) {
    buf[p++] = la_addi_w(3, 3, -24);
    buf[p++] = la_st_w(1, 3, 0);
    buf[p++] = la_st_w(REG_CPU, 3, 4);
    p = emit_li32(buf, p, REG_CPU, (uint32_t)(uintptr_t)&cpu);
    return p;
}
```

模板末尾把周期数放进 `$a0` 返回：

```c
static inline int emit_ret_cycles(uint32_t *buf, int p, int cycles) {
    buf[p++] = la_ori(4, 0, cycles);
    return p;
}
```

## 一个指令模板：`LDA #imm`

以 `LDA #imm` 为例，6502 语义是把立即数放进 A 寄存器，并更新 Z/N 标志。JIT 模板会在编译时读取立即数，把它直接编码进本机代码：

```c
case 0xA9: { // LDA #imm
    uint8_t imm = memory_readb(pc + 1);
    p = emit_li32(buf, p, 12, (uint32_t)(uintptr_t)&cpu);
    buf[p++] = la_ori(4, 0, imm);
    buf[p++] = la_st_b(4, REG_CPU, OFF_A);
    p = emit_set_zn(buf, p);
    p = emit_pc_advance(buf, p, 2);
    p = emit_ret_cycles(buf, p, 2);
    len = 2;
    break;
}
```

执行时这段机器码不再经过解释器的 opcode switch，也不再查询寻址模式表，直接完成：

```text
a0 = imm
cpu.A = a0
更新 Z/N
cpu.PC += 2
return 2 cycles
```

这里的 `emit_set_zn()` 会清掉 P 寄存器里的 N/Z 位，然后根据结果是否为 0、最高位是否为 1 写回标志。这样 JIT 与解释器的 `cpu_update_zn_flags()` 保持同一类语义。

## 内存访问必须保留副作用

NES 模拟器最容易出错的地方不是算术，而是内存访问。`memory.c` 里按地址区间把访问分到 CPU RAM、PPU、APU/输入和 Mapper：

```c
byte memory_readb(word address) {
  switch (address >> 13) {
    case 0: return cpu_ram_read(address & 0x07FF);
    case 1: return ppuio_read(address);
    case 2: return psgio_read(address);
    case 3: return cpu_ram_read(address & 0x1FFF);
    default: return mmc_read(address);
  }
}
```

写路径还额外处理 `$4014` 的 OAM DMA：

```c
if (address == 0x4014) {
  for (i = 0; i < 256; i++) {
    ppu_sprram_write(cpu_ram_read((0x100 * data) + i));
  }
  return;
}
```

因此 JIT 模板中只要涉及 NES 地址空间读写，就尽量调用 `memory_readb()` / `memory_writeb()`，而不是直接访问某个数组。比如 `STA abs` 的模板会生成对 `memory_writeb` 的调用：

```c
case 0x8D: { // STA abs
    uint16_t addr = memory_readb(pc + 1) | (memory_readb(pc + 2) << 8);
    p = emit_li32(buf, p, 19, (uint32_t)(uintptr_t)memory_writeb);
    p = emit_li32(buf, p, 4, addr);
    buf[p++] = la_ld_bu(5, REG_CPU, OFF_A);
    buf[p++] = la_jirl(1, 19, 0);
    p = emit_pc_advance(buf, p, 3);
    p = emit_ret_cycles(buf, p, 4);
    len = 3;
    break;
}
```

这样虽然牺牲了一部分性能，但能保证 PPU 寄存器写、OAM DMA、Mapper 写等副作用仍然走原来的模拟器路径。这也是当前版本“正确性优先”的核心。

## 覆盖范围与 fallback

当前 `jit_simple.c` 对常见指令分三类处理：

| 类别 | 当前策略 |
|------|----------|
| `LDA/LDX/LDY`、`STA/STX/STY` | 生成较完整的原生模板 |
| `AND/ORA/EOR` | 生成原生逻辑运算模板 |
| `ADC/SBC/CMP/CPX/CPY` | 生成到 C 级 helper 的短跳转 |
| 其他 opcode | 生成 `cpu_step_interp(pc)` fallback stub |

也就是说，当前版本并不是“所有 6502 指令都已经纯原生化”。更准确地说，它已经把 JIT 执行框架接入了模拟器，并覆盖了一批高频、容易验证的指令；其余指令仍由解释器保证功能正确。

fallback 的实现很直接：

```c
default:
    p = emit_li32(buf, p, 4, pc);
    p = emit_li32(buf, p, 12, (uint32_t)(uintptr_t)cpu_step_interp);
    buf[p++] = la_jirl(1, 12, 0);
    break;
```

这个设计有一个好处：即使遇到未覆盖指令，也不会把整个模拟器切回纯解释器模式，而是只在这个 PC 上缓存一个“解释器单步调用”。下一次再执行同一个 PC，就不需要重新编译 fallback 了。

## Shell 控制入口

JIT 通过 xOS shell 控制：

```text
jitmode on
jitmode off
jitmode stats
jitmode reset
jitmode dump
```

对应实现位于 `software/xos_pro_max/src/shell.c`。打开 JIT 时会先调用 `jit_init()` 清空缓存：

```c
if (strcmp(argv[1], "on") == 0) {
    jit_init();
    jit_enabled = true;
    printf("JIT mode enabled\n");
}
```

`jitmode stats` 会打印类似：

```text
[JIT] calls=... hits=... misses=... invalid=... hit-rate=...%
```

这对现场调试非常重要。之前调试系统时，经常会遇到“以为 JIT 开了，但实际主路径没走 JIT”的情况。有了命中率和调用次数，至少可以先确认执行通道是否真的切过去了。

## 与第 15 篇 demo 的区别

第 15 篇里的 JIT demo 是一个极简虚拟机：`LOAD/ADD/JMP/HALT`，所有状态都在一两个寄存器里，内存没有复杂副作用。

这篇接入 LiteNES 后，问题完全不一样：

1. 6502 有 A/X/Y/P/SP/PC 等完整状态；
2. 指令周期要返回给 PPU 时序模拟；
3. 地址空间读写可能触发 PPU、APU、OAM DMA、Mapper；
4. 生成的代码要和 C 代码共享 `cpu` 全局结构；
5. 生成代码后必须刷新 I-cache，否则可能执行到旧指令；
6. 未覆盖指令必须可回退，而不是直接报错。

可以把当前版本理解为从 demo 到工程实现之间的第一道桥：它先把 JIT 的生命周期、代码缓存、机器码生成、解释器协同都接进了系统，之后再逐步扩大原生模板覆盖范围。

## 后续优化路线

当前版本的瓶颈也很明显：单指令 JIT 仍然有较多函数调用开销，访存模板大量调用 `memory_readb()` / `memory_writeb()`，分支和跳转也没有跨块链接。因此后续可以按下面顺序演进：

1. **扩大原生覆盖范围**：优先实现 `INC/DEC/ASL/LSR/ROL/ROR`、分支和栈相关指令；
2. **块级 JIT**：把顺序执行的多条 6502 指令合成一个 basic block；
3. **I/O 白名单**：普通 RAM/ROM 访问走更短路径，PPU/APU/Mapper 访问继续走安全路径；
4. **回退原因统计**：区分 opcode 未覆盖、I/O 保守回退、缓存失效等原因；
5. **热点块优化**：对高频 PC 段做块链接或指令融合；
6. **自动回归**：用 `jitmode on/off` 对同一 ROM 跑若干帧，做截图或状态对比。

软件 JIT 并不是本毕业设计里唯一的 NES 加速路线。它适合展示“在自研 LoongArch SoC 上动态生成并执行本机代码”的系统能力；而第 18 篇会继续记录另一条更硬核的路线：把 NES 核作为硬件外设接入 SoC，让 6502、PPU 和像素输出直接在 FPGA 逻辑中运行。
