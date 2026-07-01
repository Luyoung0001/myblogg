---
title: "编译与链接实验：在 QEMU 上运行 RISC-V 裸机程序"
date: 2025-10-27 15:03:13
tags:
  - "RISC-V"
  - "QEMU"
  - "linker script"
  - "bare metal"
categories:
  - "Computer Architecture"
---

## 前言

昨天面试，有一个问题问到了链接脚本以及 elf 文件结构。感觉回答得不怎么样，很多概念的细节都想不起来了，面试官也没深问。于是我立即决定做一个实验好好回顾一下这一块儿的知识：
- 写一个比较复杂的测试程序，最好能包含各个段；
- 写一个简单的启动代码；
- 写一个简单的链接脚本；
- 写一个简单的库，包含类似于 printf、memcopy；
- 然后编译、链接，放在裸机上执行。

## 项目结构

```bash
tree .
.
├── compile_commands.json
├── debug.gdb
├── include
│   ├── hello.h
│   ├── mylib.h
│   └── world.h
├── linker.ld
├── main.c
├── Makefile
├── qemu.log
├── README.md
└── src
    ├── hello.c
    ├── mylib.c
    ├── startup.s
    ├── trm.c
    └── world.c
```

include/ 中放的是各个头文件，src/ 中放的是函数的实现。其它文件比如链接脚本 linker.ld、main.c 放在根目录。

## 部分代码以及说明

mylib.h：

```C
#ifndef _MYLIB_H_
#define _MYLIB_H_
#include <stdint.h>
void* memcpy(void* out, const void* in, uint32_t n);
void print_str(const char *s);
void uart_putc(char c);
#endif
```

hello.h、world.h:

```C
#ifndef _HELLO_H_
#define _HELLO_H_
void say_hello();
#endif


#ifndef _WORLD_H_
#define _WORLD_H_
void say_world();
#endif
```

接着是启动代码：

```S
.section entry, "ax"
.globl _start
.type _start, @function

_start:
  mv s0, zero
  la sp, _stack_pointer
  jal _trm_init
```

_stack_pointer 来自于 链接脚本中定义的符号：

```c
OUTPUT_FORMAT("elf32-littleriscv")
OUTPUT_ARCH(riscv)
ENTRY(_start)

MEMORY {
  flash (rx) : ORIGIN = 0x80000000, LENGTH = 128M
  sram (rwx) : ORIGIN = 0x80040000, LENGTH = 128M - 256K
}

SECTIONS {
  .entry :ALIGN(4){
    *(entry)
  } > flash

  .text :ALIGN(4){
    *(.text*)
  } > flash

  .rodata :ALIGN(4) {
    *(.rodata*)
    *(.srodata*)
  } > flash

  .data :ALIGN(4) {
    *(.data*)
    *(.sdata*)
  } > sram AT> flash

  _sdata = ADDR(.data);
  _edata = ADDR(.data) + SIZEOF(.data);
  _sidata = LOADADDR(.data);

  .bss :ALIGN(4) {
    *(.bss*)
    *(.sbss*)
    *(.scommon*)
  } > sram

  _sbss = ADDR(.bss);
  _ebss = ADDR(.bss) + SIZEOF(.bss);

  _heap_end = ORIGIN(sram) + LENGTH(sram);
  _stack_pointer = _heap_end;
}
```

本来我的想在 0x20000000 处定义 flash，在 0x0f000000 处定义 sram。但是 qemu-system-riscv32 默认从 0x1000 处执行代码：

```gdb
0x00001000 in ?? ()
(gdb) x/20wx 0x1000
0x1000:	0x00000297	0x02828613	0xf1402573	0x0202a583
0x1010:	0x0182a283	0x00028067	0x80000000	0x00000000
0x1020:	0x87e00000	0x00000000	0x4942534f	0x00000002
0x1030:	0x80000000	0x00000001	0x00000000	0x00000000
0x1040:	0x00000000	0x00000000	0x00000000	0x00000000
```

也就是这些：

```asm
0:   00000297          auipc   t0,0x0
4:   02828613          addi    a2,t0,40
8:   f1402573          csrr    a0,mhartid
c:   0202a583          lw      a1,32(t0)
10:  0182a283          lw      t0,24(t0)
14:  00028067          jr      t0
```

作用就是：
- 初始化 t0 和 a2：计算后续数据地址;
- 读取 Hart ID：检查当前 CPU 核心（多核场景下可能需要区分）;
- 加载配置数据：从 0x1020 和 0x1018 加载参数到 a1 和 t0;
- 跳转到主程序：通过 jr t0 跳转到 0x80000000（用户代码入口）。

既然它能跳转到 0x80000000，那么我就有办法来完成这个实验了：我打算在 0x8000000 处当做 flash、0x80040000 当做 sram 来完成实验。

## 链接 & 链接脚本

链接脚本非常重要，我刚开始并没有很好的理解链接脚本中定义的变量，事实上它不是变量，它被称为符号更贴切。

```c
  _sdata = ADDR(.data);
  _edata = ADDR(.data) + SIZEOF(.data);
  _sidata = LOADADDR(.data);

  _sbss = ADDR(.bss);
  _ebss = ADDR(.bss) + SIZEOF(.bss);

  _heap_end = ORIGIN(sram) + LENGTH(sram);
  _stack_pointer = _heap_end;
```

比如对于 _sdata，我之前的理解的是有一个变量 _sdata，它的值是 .data 段的开始地址。后来发现并不是这样，不然无法解释以下代码：
trm.c:
```C
#include <mylib.h>

extern char _sidata, _sdata, _edata, _sbss, _ebss;
extern int main();

// 简单串口输出实现
void uart_putc(char c) {
    // 实现向串口发送单个字符
    volatile char *uart = (volatile char*)0x10000000;
    *uart = c;
}

/* 初始化 .data 段 */
static void init_data(char* sidata, char* sdata, char* edata) {
    uintptr_t size = (uintptr_t)edata - (uintptr_t)sdata;
    if (size > 0) {
        memcpy((void*)sdata, (void*)sidata, size);
    }
}
/* 清零 .bss 段 */
static void zero_bss(char* sbss, char* ebss) {
    while (sbss < ebss) {
        *sbss = 0;
        sbss++;
    }
}

void _trm_init() {
    init_data(&_sidata, &_sdata, &_edata);
    zero_bss(&_sbss, &_ebss);
    (void) main();
    while (1);
}
```

这个 C 代码中使用了 extern 关键字来声明char 类型的 _sidata 等来告知编译器外部定义了一个 _sidata 变量。而且这里有一个巧妙的用法：_trm_init() 并没有使用这些变量的值，它使用了这些变量的地址。于是编译器编译好这些代码后会空着这些参数，链接的时候才会去回填这些变量的地址。

但是事实是，这个变量根本不存在，怎么办呢？于是后面在链接的时候，借助链接脚本，链接器从符号表中找到了符号 _sidata，并且它有地址，这时候就会完成对上述目标文件的回填。

从 trm.o 文件的信息也可以看到：

```bash
rvojbdump -d ./build/trm.o
...
000000e0 <_trm_init>:
  e0:   ff010113                addi    sp,sp,-16
  e4:   00112623                sw      ra,12(sp)
  e8:   00812423                sw      s0,8(sp)
  ec:   01010413                addi    s0,sp,16
  f0:   000007b7                lui     a5,0x0
  f4:   00078613                mv      a2,a5
  f8:   000007b7                lui     a5,0x0
  fc:   00078593                mv      a1,a5
 100:   000007b7                lui     a5,0x0
 104:   00078513                mv      a0,a5
 108:   00000097                auipc   ra,0x0
 10c:   000080e7                jalr    ra # 108 <_trm_init+0x28>
 110:   000007b7                lui     a5,0x0
 114:   00078593                mv      a1,a5
 118:   000007b7                lui     a5,0x0
 11c:   00078513                mv      a0,a5
 120:   00000097                auipc   ra,0x0
 124:   000080e7                jalr    ra # 120 <_trm_init+0x40>
 128:   00000097                auipc   ra,0x0
 12c:   000080e7                jalr    ra # 128 <_trm_init+0x48>
```

关键点在于符号如 `_sidata`, `_sdata`, `_edata` 等目前是未解析的（地址为0），这将在链接阶段通过链接脚本和符号表解决。

### 当前未链接状态的分析
在反汇编中可以看到：
```assembly
000000e0 <_trm_init>:
  ...
  f0:   000007b7                lui     a5,0x0       # 加载 _sidata 高20位 (当前为0)
  f4:   00078613                mv      a2,a5        # a2 = _sidata
  f8:   000007b7                lui     a5,0x0       # 加载 _sdata 高20位
  fc:   00078593                mv      a1,a5        # a1 = _sdata
 100:   000007b7                lui     a5,0x0       # 加载 _edata 高20位
 104:   00078513                mv      a0,a5        # a0 = _edata
 108:   00000097                auipc   ra,0x0       # 调用 init_data
```

所有符号地址当前都是0，因为：
- 这些符号在 `trm.c` 中是 `extern` 声明
- 它们的实际定义在链接脚本中
- 链接器会在链接阶段解析这些符号的真实地址

### 链接阶段的关键处理
链接器会执行以下关键操作：

#### 符号解析与重定位
- **符号表合并**：链接器收集所有目标文件（.o）的符号表
- **地址分配**：根据链接脚本的布局确定每个段的基地址
- **重定位**：修改代码中的地址引用（如 `lui a5,0x0` → `lui a5,%hi(_sdata)`）

#### 链接脚本的核心作用
链接脚本 linker.ld:
```ld
OUTPUT_FORMAT("elf32-littleriscv")
OUTPUT_ARCH(riscv)
ENTRY(_start)

MEMORY {
  flash (rx) : ORIGIN = 0x80000000, LENGTH = 128M
  sram (rwx) : ORIGIN = 0x80040000, LENGTH = 128M - 256K
}

SECTIONS {
  .entry :ALIGN(4){
    *(entry)
  } > flash

  .text :ALIGN(4){
    *(.text*)
  } > flash

  .rodata :ALIGN(4) {
    *(.rodata*)
    *(.srodata*)
  } > flash

  .data :ALIGN(4) {
    *(.data*)
    *(.sdata*)
  } > sram AT> flash

  _sdata = ADDR(.data);
  _edata = ADDR(.data) + SIZEOF(.data);
  _sidata = LOADADDR(.data);

  .bss :ALIGN(4) {
    *(.bss*)
    *(.sbss*)
    *(.scommon*)
  } > sram

  _sbss = ADDR(.bss);
  _ebss = ADDR(.bss) + SIZEOF(.bss);

  _heap_end = ORIGIN(sram) + LENGTH(sram);
  _stack_pointer = _heap_end;
}
```

它会创建特殊的符号表。

#### 重定位过程详解
先看看链接后的代码：

```assembly
800001f0 <_trm_init>:
800001f0:	ff010113          	addi	sp,sp,-16
800001f4:	00112623          	sw	ra,12(sp)
800001f8:	00812423          	sw	s0,8(sp)
800001fc:	01010413          	addi	s0,sp,16
80000200:	800407b7          	lui	a5,0x80040
80000204:	01078613          	addi	a2,a5,16 # 80040010 <_heap_end+0xf8040010>
80000208:	800407b7          	lui	a5,0x80040
8000020c:	00078593          	mv	a1,a5
80000210:	800007b7          	lui	a5,0x80000
80000214:	31c78513          	addi	a0,a5,796 # 8000031c <_heap_end+0xf800031c>
80000218:	f35ff0ef          	jal	ra,8000014c <init_data>
8000021c:	800407b7          	lui	a5,0x80040
80000220:	01478593          	addi	a1,a5,20 # 80040014 <_heap_end+0xf8040014>
80000224:	800407b7          	lui	a5,0x80040
80000228:	01078513          	addi	a0,a5,16 # 80040010 <_heap_end+0xf8040010>
8000022c:	f79ff0ef          	jal	ra,800001a4 <zero_bss>
80000230:	03c000ef          	jal	ra,8000026c <main>
80000234:	0000006f          	j	80000234 <_trm_init+0x44>
```
链接器会：
1. 计算 `_sidata`, `_sdata`, `_edata` 的实际地址
2. 修改指令中的地址占位符：
   ```diff
   - lui a5,0x0         # 原始未链接代码
   + lui a5,%hi(_sdata) # 链接后：加载地址高20位
   + addi a5,a5,%lo(_sdata) # 加载地址低12位
   ```

### 3. **链接后的预期结果**
链接完成后，代码会变成类似这样：
```diff
- e0:   ff010113                addi    sp,sp,-16
+ 800001f0:	ff010113            add     sp,sp,-16

- e4:   00112623                sw      ra,12(sp)
+ 800001f4:	00112623            sw      ra,12(sp)

- e8:   00812423                sw      s0,8(sp)
+ 800001f8:	00812423            sw      s0,8(sp)

- ec:   01010413                addi    s0,sp,16
+ 800001fc:	01010413            addi    s0,sp,16

- f0:   000007b7                lui     a5,0x0
+ 80000200:	800407b7            lui     a5,0x80040

- f4:   00078613                mv      a2,a5
+ 80000204:	01078613            addi    a2,a5,16 # 80040010 <_heap_end+0xf8040010>

- f8:   000007b7                lui     a5,0x0
+ 80000208:	800407b7            lui     a5,0x80040

- fc:   00078593                mv      a1,a5
+ 8000020c:	00078593            mv      a1,a5

- 100:   000007b7                lui     a5,0x0
+ 80000210:	800007b7             lui     a5,0x80000

- 104:   00078513                mv      a0,a5
+ 80000214:	31c78513             addi    a0,a5,796 # 8000031c <_heap_end+0xf800031c>

- 108:   00000097                auipc   ra,0x0
- 10c:   000080e7                jalr    ra # 108 <_trm_init+0x28>
+ 80000218:	f35ff0ef             jal     ra,8000014c <init_data>

- 110:   000007b7                lui     a5,0x0
+ 8000021c:	800407b7             lui     a5,0x80040

- 114:   00078593                mv      a1,a5
+ 80000220:	01478593             addi    a1,a5,20 # 80040014 <_heap_end+0xf8040014>

- 118:   000007b7                lui     a5,0x0
+ 80000224:	800407b7             lui     a5,0x80040

- 11c:   00078513                mv      a0,a5
+ 80000228:	01078513             addi    a0,a5,16 # 80040010 <_heap_end+0xf8040010>

- 120:   00000097                auipc   ra,0x0
- 124:   000080e7                jalr    ra # 120 <_trm_init+0x40>
+ 8000022c:	f79ff0ef             jar     ra,800001a4 <zero_bss>

- 128:   00000097                auipc   ra,0x0
- 12c:   000080e7                jalr    ra # 128 <_trm_init+0x48>
+ 80000230:	03c000ef             jal     ra,8000026c <main>

```
从上面的 diff 中可以看到一个很清晰的回填过程。

因此可以暂时有这些结论：

1. **符号而非变量**：
   - `_sdata`, `_edata`, `_sidata` 不是程序中的真实变量
   - 它们是链接器在链接过程中创建的**符号标签**
   - 这些符号不占用任何内存空间，只是地址的别名

2. **地址值的载体**：
   ```ld
   _sdata = ADDR(.data);        // .data段的运行时地址(VMA)
   _sidata = LOADADDR(.data);   // .data段的加载地址(LMA)
   ```
   - 等号不是赋值，而是**地址绑定**
   - 链接器会计算这些表达式的值并存储在符号表中


### 具体实现机制

1. **符号表创建**：
   - 链接器解析链接脚本时，会创建特殊符号表项：
     ```
     符号名   类型    值
     _sdata   D    0x80040000
     _edata   B    0x80040010
     _sidata  A    0x8000031c
     ```

2. **重定位过程**：
   - 当遇到 `lui a5,0x0` 这样的指令：
   - 检查重定位条目：`偏移0xf0，类型R_RISCV_HI20，符号_sdata`
   - 计算：`(0x80040000 >> 12) & 0xFFFFF`
   - 修改指令：`lui a5, 0x80040`

3. **地址回填示例**：
   原始指令：
   ```asm
   f0:   000007b7                lui     a5,0x0   # 加载_sdata
   ```
   链接后变为：
   ```asm
   800000f0:   800207b7                lui     a5,0x80020
   ```

### 关键特性

1. **零存储开销**：
   - 这些符号不占用.data/.bss空间
   - 只存在于符号表中，链接后地址直接编码到指令里

2. **强类型地址**：
   - 在C代码中声明为`extern char _sdata;`
   - 实际使用的是`&_sdata`获取地址值
   - 类型`char`只是占位符，实际可以是任意类型

3. **链接时计算**：
   - `_edata = _sdata + SIZEOF(.data)`
   - 链接器在布局完成后计算具体值
   - 所有表达式在链接阶段解析为常量


### 验证方法

1. **查看符号表**：
   ```bash
    riscv64-linux-gnu-nm bin/firmware.elf | grep -E '_sdata|_edata|_sidata|_sbss|_ebss|_stack'
    80040014 B _ebss
    80040010 D _edata
    80040010 B _sbss
    80040000 D _sdata
    8000031c A _sidata
    88000000 A _stack_pointer
   ```

2. **反汇编验证**：
   ```bash
    80000200:	800407b7          	lui	a5,0x80040
    80000204:	01078613          	addi	a2,a5,16 # 80040010 <_heap_end+0xf8040010>
   ```

## 遇到的问题

此时，有一个关建的编译参数和链接参数出现了：-fno-pic、-Wl,-no-pie。为什么我不加这两个参数，编译出来的代码就很奇怪，这个两个选项到底有什么用？

对于启动代码：
```s
    _start:
    mv s0, zero
    la sp, _stack_pointer
    jal _trm_init
```
取消 -fno-pic、-Wl,-no-pie 选项前：
```s
80000000 <_start>:
80000000:	00000413          	li	s0,0
80000004:	08000117          	auipc	sp,0x8000
80000008:	ffc10113          	addi	sp,sp,-4 # 88000000 <_heap_end>
8000000c:	1e4000ef          	jal	ra,800001f0 <_trm_init>
```
取消 -fno-pic、-Wl,-no-pie 选项后：
```s
80000000:	00000413          	li	s0,0
80000004:	00040117          	auipc	sp,0x40
80000008:	09812103          	lw	sp,152(sp) # 8004009c <_GLOBAL_OFFSET_TABLE_+0x4>
8000000c:	1e8000ef          	jal	ra,800001f4 <_trm_init>
```
显然，对于裸机程序，前面的代码是我们想要的，而后面的代码，错误的编译成了 lw 指令，为什么呢？

DeepSeekR1 解答：

您观察到的现象非常关键，这涉及到位置无关代码(PIC)和位置无关可执行文件(PIE)的概念。让我详细解释为什么在裸机程序中需要禁用这些特性。

### 问题核心：位置无关代码(PIC) vs 固定地址代码

#### 1. **`-fno-pic` 编译选项**
- **作用**：禁用位置无关代码生成
- **默认行为**：GCC默认启用`-fpic`（位置无关代码）
- **裸机程序问题**：
  - PIC代码假设会被加载到任意地址运行
  - 使用相对偏移访问全局变量/函数
  - 需要运行时重定位支持（**裸机没有**），这是重点！！！

#### 2. **`-Wl,-no-pie` 链接选项**
- **作用**：禁用位置无关可执行文件
- **默认行为**：现代Linux发行版默认启用PIE
- **裸机程序问题**：
  - PIE可执行文件没有固定加载地址
  - 需要动态链接器支持（**裸机没有**），这是重点！！！

### 代码对比分析

#### 取消选项前（正确）：
```assembly
80000000:	00000413          	li	s0,0
80000004:	08000117          	auipc	sp,0x8000   # PC相对加载
80000008:	ffc10113          	addi	sp,sp,-4    # 调整得到绝对地址
8000000c:	1e4000ef          	jal	ra,800001f0 <_trm_init>
```
- `auipc sp,0x8000`：计算PC + 0x8000000
- `addi sp,sp,-4`：微调得到精确地址
- **直接访问符号地址**，适合裸机

#### 取消选项后（错误）：
```assembly
80000000:	00000413          	li	s0,0
80000004:	00040117          	auipc	sp,0x40     # 计算GOT位置
80000008:	09812103          	lw	sp,152(sp) # 从GOT加载地址
8000000c:	1e8000ef          	jal	ra,800001f4 <_trm_init>
```
- `auipc sp,0x40`：计算全局偏移表(GOT)位置
- `lw sp,152(sp)`：从GOT加载实际地址，此时什么都加载不到！！！
- **间接访问**，需要运行时重定位

### 为什么PIC/PIE不适合裸机程序

1. **缺少动态链接器**：
   - PIC/PIE需要运行时重定位
   - 裸机环境没有动态链接器（如ld.so）
   - GOT表无法初始化

2. **固定地址需求**：
   - 裸机启动地址固定（如0x80000000）
   - 中断向量表等必须位于固定位置
   - PIC的灵活性反而成为负担

3. **性能开销**：
   - PIC需要额外的间接访问
   - 每条全局访问都需要GOT查找
   - 裸机程序通常追求极致性能

4. **启动代码特殊性**：
   - 启动时内存系统未初始化
   - GOT表可能位于未初始化的内存区域
   - 无法处理重定位前的访问

### 解决方案：禁用PIC/PIE

```bash
# 编译选项
riscv64-unknown-elf-gcc -fno-pic -c start.S

# 链接选项
riscv64-unknown-elf-ld -Wl,-no-pie -T link.ld -o firmware.elf
```

### 选项详解

| 选项 | 作用 | 裸机必要性 |
|------|------|------------|
| `-fno-pic` | 禁用位置无关代码 | 必须 |
| `-Wl,-no-pie` | 禁用位置无关可执行文件 | 必须 |
| `-nostdlib` | 禁用标准库 | 推荐 |
| `-nostartfiles` | 禁用默认启动文件 | 推荐 |
| `-static` | 静态链接 | 推荐 |

### 深入技术细节

#### 1. PIC如何工作
![PIC 与非 PIC 的全局变量访问路径](/images/mermaid-svg/riscv-qemu-compilation-linking/pic-pie-access-path.svg)

#### 2. 启动代码的正确处理
裸机启动代码需要：
```c
// 正确的启动代码示例
void _start() {
    // 1. 直接设置栈指针
    asm volatile("la sp, _stack_top");

    // 2. 初始化数据段
    init_data();

    // 3. 清零BSS段
    zero_bss();

    // 4. 调用主函数
    main();

    // 5. 无限循环
    while(1);
}
```

### 总结

在裸机程序开发中：
1. **必须使用`-fno-pic`**：避免生成依赖GOT的代码
2. **必须使用`-Wl,-no-pie`**：生成固定地址的可执行文件
3. **避免间接访问**：启动代码应直接操作硬件地址
4. **简化内存模型**：裸机没有虚拟内存/动态链接支持

基本上解决了我的疑问。

## 运行结果
```bash
make qemu_run
128+0 records in
128+0 records out
134217728 bytes (134 MB, 128 MiB) copied, 0.28816 s, 466 MB/s
1+1 records in
1+1 records out
812 bytes copied, 0.000487525 s, 1.7 MB/s
hello hello hello world
world
world
world
world
hello world
hello world
hello world
hello world
hello world
hello world
hello world
```

## 依然存在的疑问：elf 文件分析

```bash
rvobjdump -h ./bin/firmware.elf

./bin/firmware.elf:     file format elf32-littleriscv

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .entry        00000010  80000000  80000000  00001000  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .text         000002fc  80000010  80000010  00001010  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .rodata       0000000f  8000030c  8000030c  0000130c  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .data         00000010  80040000  8000031c  00002000  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  4 .bss          00000004  80040010  8000032c  00002010  2**2
                  ALLOC
  5 .debug_info   0000050a  00000000  00000000  00002010  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
  6 .debug_abbrev 000003a3  00000000  00000000  0000251a  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
  7 .debug_aranges 000000c0  00000000  00000000  000028c0  2**3
                  CONTENTS, READONLY, DEBUGGING, OCTETS
  8 .debug_line   00000400  00000000  00000000  00002980  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
  9 .debug_str    000001b2  00000000  00000000  00002d80  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 10 .debug_line_str 000000bf  00000000  00000000  00002f32  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 11 .comment      0000002b  00000000  00000000  00002ff1  2**0
                  CONTENTS, READONLY
 12 .riscv.attributes 0000002e  00000000  00000000  0000301c  2**0
                  CONTENTS, READONLY
 13 .debug_frame  000001a0  00000000  00000000  0000304c  2**2
                  CONTENTS, READONLY, DEBUGGING, OCTETS
```

这里抛出的问题是：**一个 elf 文件有那么多的段，bss、data、rodata 分别放的是什么数据？**


### 各数据段详解

| 段名     | 大小      | VMA        | LMA        | 文件偏移   | 作用                                                                 | 是否占用磁盘空间 |
|----------|----------|------------|------------|------------|----------------------------------------------------------------------|------------------|
| **.rodata** | 0x0f (15)| 0x8000030c | 0x8000030c | 0x0000130c | **只读数据**：字符串常量、const全局变量、跳转表等                     | ✅ 是            |
| **.data**   | 0x10 (16)| 0x80040000 | 0x8000031c | 0x00002000 | **已初始化数据**：非零初始值的全局/静态变量                          | ✅ 是            |
| **.bss**    | 0x04 (4) | 0x80040010 | 0x8000032c | 0x00002010 | **未初始化数据**：零初始值或未初始化的全局/静态变量                   | ❌ 否            |

### 关键特性分析

1. **.rodata (Read-Only Data)**：
   - **内容**：程序中的常量数据
     ```c
     const int version = 1;
     const char* msg = "Hello, World!";
     ```
   - **内存映射**：VMA = LMA = 0x8000030c
     - 直接存储在ROM/Flash中
     - 运行时无需复制，直接从加载地址访问
   - **文件占用**：有实际内容（15字节），存储在文件偏移0x130c处

2. **.data (Initialized Data)**：
   - **内容**：初始值非零的全局/静态变量
     ```c
     int global_counter = 42;
     static float pi = 3.14;
     ```
   - **双重地址**：
     - LMA (0x8000031c)：ROM中的存储位置
     - VMA (0x80040000)：RAM中的运行时位置
   - **启动过程**：需要从ROM复制到RAM
   - **文件占用**：有实际内容（16字节），存储在文件偏移0x2000处

3. **.bss (Block Started by Symbol)**：
   - **内容**：初始值为零或未初始化的变量
     ```c
     int uninitialized_var;
     static char buffer[1024] = {0};
     ```
   - **零初始化**：启动时需要清零
   - **关键特性**：
     - 在ELF文件中**不占用空间**（文件偏移0x2010后无内容）
     - 运行时在RAM中分配空间（此处4字节）
     - LMA (0x8000032c) 仅表示加载结束位置，无实际数据

### 内存布局图示

```
ROM/Flash 区域 (LMA)          RAM 区域 (VMA)
┌───────────────────┐        ┌───────────────────┐
│ 0x80000000: .text │        │                   │
│ ...               │        │                   │
├───────────────────┤        ├───────────────────┤
│ 0x8000030c:       │        │ 0x80040000:       │
│   .rodata (15B)   │        │   .data (16B)     │
├───────────────────┤        ├───────────────────┤
│ 0x8000031c:       │        │ 0x80040010:       │
│   .data (LMA)     │───────>│   .bss (4B)       │
├───────────────────┤        └───────────────────┘
│ 0x8000032c:       │
│   (LMA end)       │
└───────────────────┘
```

### 启动代码的处理

在`_trm_init`函数中需要：
```c
/* 复制.data段 */
memcpy(&_sdata, &_sidata, (size_t)(&_edata - &_sdata));

/* 清零.bss段 */
memset(&_sbss, 0, (size_t)(&_ebss - &_sbss));
```

### 为什么.bss不占磁盘空间？

1. **效率优化**：
   - 所有.bss变量初始值都是0
   - 存储大量零值会浪费磁盘空间
   - 改为记录"此区域需要清零"的元数据

2. **ELF实现**：
   - 在Program Header中标记为`PT_LOAD`
   - `p_memsz > p_filesz` 表示需要额外清零的内存
   - 查看详细信息：
     ```bash
     riscv64-unknown-elf-readelf -l firmware.elf
     ```

### 验证方法

**查看各段大小**：
   ```bash
    riscv64-unknown-elf-size bin/firmware.elf
    text    data     bss     dec     hex filename
    795      16       4     815     32f bin/firmware.elf
   ```
   这里的 4 意味着在 sram 中将会占用 4 字节空间。，但在 flash 中体积为 0。


### 总结

- **.rodata**：只读常量，存储在ROM，直接访问
- **.data**：已初始化变量，存储在ROM，需复制到RAM
- **.bss**：未初始化变量，**不占磁盘空间**，启动时清零
- **关键区别**：
  - .data有实际磁盘内容
  - .bss只有内存分配需求
  - .rodata有内容但不可写

elf 中的 bss 只有描述的元数据，比如在链接脚本中：

```s
 .bss :ALIGN(4) {
    *(.bss*)
    *(.sbss*)
    *(.scommon*)
  } > sram
  _sbss = ADDR(.bss);
  _ebss = ADDR(.bss) + SIZEOF(.bss);
```

链接后形成的 elf 文件中 bss 并没有 size，但是如果有 boatloader，我们必须利用 _sbss、_ebss 信息来为 bss 在 sram 中开辟内存空间并初始化：

```c
/* 清零 .bss 段 */
static void zero_bss(char* sbss, char* ebss) {
    while (sbss < ebss) {
        *sbss = 0;
        sbss++;
    }
}
```


### 1. .bss段在ELF文件中的本质

```lds
.bss :ALIGN(4) {
    *(.bss*)
    *(.sbss*)
    *(.scommon*)
} > sram

_sbss = ADDR(.bss);
_ebss = ADDR(.bss) + SIZEOF(.bss);
```

- **ELF文件中的.bss**：
  - 只有描述性元数据（起始地址、大小、内存属性）
  - **没有实际内容**（因为所有初始值都是0）
  - 在section header中标记为`NOBITS`类型
  - 查看验证：
    ```bash
    riscv64-unknown-elf-readelf -S bin/firmware.elf
    There are 18 section headers, starting at offset 0x36a0:

    Section Headers:
    [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
    [ 0]                   NULL            00000000 000000 000000 00      0   0  0
    [ 1] .entry            PROGBITS        80000000 001000 000010 00  AX  0   0  4
    [ 2] .text             PROGBITS        80000010 001010 0002fc 00  AX  0   0  4
    [ 3] .rodata           PROGBITS        8000030c 00130c 00000f 00   A  0   0  4
    [ 4] .data             PROGBITS        80040000 002000 000010 00  WA  0   0  4
    [ 5] .bss              NOBITS          80040010 002010 000004 00  WA  0   0  4
    [ 6] .debug_info       PROGBITS        00000000 002010 00050a 00      0   0  1
    [ 7] .debug_abbrev     PROGBITS        00000000 00251a 0003a3 00      0   0  1
    [ 8] .debug_aranges    PROGBITS        00000000 0028c0 0000c0 00      0   0  8
    [ 9] .debug_line       PROGBITS        00000000 002980 000400 00      0   0  1
    [10] .debug_str        PROGBITS        00000000 002d80 0001b2 01  MS  0   0  1
    [11] .debug_line_str   PROGBITS        00000000 002f32 0000bf 01  MS  0   0  1
    [12] .comment          PROGBITS        00000000 002ff1 00002b 01  MS  0   0  1
    [13] .riscv.attributes RISCV_ATTRIBUTE 00000000 00301c 00002e 00      0   0  1
    [14] .debug_frame      PROGBITS        00000000 00304c 0001a0 00      0   0  4
    [15] .symtab           SYMTAB          00000000 0031ec 000310 10     16  29  4
    [16] .strtab           STRTAB          00000000 0034fc 0000f0 00      0   0  1
    [17] .shstrtab         STRTAB          00000000 0035ec 0000b3 00      0   0  1
    Key to Flags:
    W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
    L (link order), O (extra OS processing required), G (group), T (TLS),
    C (compressed), x (unknown), o (OS specific), E (exclude),
    p (processor specific)
    ```
    注意`Type=NOBITS`和`Off=02010`（文件偏移）这个偏移被.debug_info 使用了！因此实际文件无内容。

### 2. 启动代码的必要处理

```c
/* 清零 .bss 段 */
void zero_bss(char* sbss, char* ebss) {
    while (sbss < ebss) {
        *sbss = 0;  // 逐字节清零
        sbss++;
    }
}
```

### 3. 完整启动序列

在bootloader或启动代码中必须包含：

```c
// 声明链接脚本提供的符号
extern char _sbss[], _ebss[];

void _start() {
    // 1. 初始化栈指针
    asm volatile("la sp, _stack_top");

    // 2. 复制.data段（从ROM到RAM）
    extern char _sdata[], _edata[], _sidata[];
    memcpy(_sdata, _sidata, (size_t)(_edata - _sdata));

    // 3. 清零.bss段（关键步骤！）
    zero_bss(_sbss, _ebss);

    // 4. 调用主程序
    main();

    // 5. 退出处理
    while(1);
}
```

### 4. 为什么必须手动清零？

1. **C语言规范要求**：
   - C标准规定全局变量必须初始化为0
   - 未初始化变量行为是未定义的

2. **硬件特性**：
   - RAM上电后内容随机
   - 不保证为0值

3. **安全考虑**：
   - 防止敏感信息泄漏
   - 确保程序行为可预测

4. **优化意义**：
   - 避免存储大量零值节省Flash空间
   - 启动时一次性清零效率更高


### 5. 特殊情况处理

#### 5.1 部分初始化的.bss变量
```c
// 显式初始化为0 -> 仍放入.bss
uint8_t buffer[1024] = {0};
```

#### 5.2 非零初始化的"伪.bss"
```c
// 实际放入.data段（占用Flash空间）
uint8_t non_zero_buffer[1024] = {1};
```


### bss 总结

1. **.bss段在ELF中只有元数据**，没有实际内容
2. **必须使用`_sbss`和`_ebss`**来定位内存区域
3. **启动代码必须手动清零**.bss区域
4. `zero_bss()`函数是实现这一点的标准方式

这种设计是嵌入式系统的核心优化之一，它：
- 显著减小可执行文件大小
- 符合C语言规范要求
- 允许灵活的内存布局
- 确保程序启动状态的确定性






















