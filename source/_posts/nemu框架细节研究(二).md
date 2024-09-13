---
title: nemu框架细节研究（二）
date: 2024-09-12 11:22:33
tags:
    - NEMU
    - PA2
categories:
    - RISCV
    - PA
    - 虚拟机
---

<!--more-->

## 一、前言
前文细致地研究了命令 `make ARCH=native ALL=add run` 的运行过程，对于Makefile中的行为有了很准确地把握（几乎没有疑点，可以随意修改 Makefile 且保证不会出错）。这篇文章继续研究命令`make ARCH=riscv32-nemu ALL=add run`的行为。

## 二、riscv32-nemu 开始的地方

由于 `ARCH=riscv32-nemu`，因此 `ISA=riscv32`，`PLATFORM=nemu`。我们来看看它新包含了哪些 可以包含到Makefile中的文件:

```Makefile
-include $(AM_HOME)/scripts/$(ARCH).mk
```

这里的 `ARCH=riscv32-nemu`，因此它会包含`$(AM_HOME)/scripts/riscv32-nemu.mk`:

```Makefile
include $(AM_HOME)/scripts/isa/riscv.mk
include $(AM_HOME)/scripts/platform/nemu.mk
CFLAGS  += -DISA_H=\"riscv/riscv.h\"
COMMON_CFLAGS += -march=rv32im_zicsr -mabi=ilp32   # overwrite
LDFLAGS       += -melf32lriscv                     # overwrite

AM_SRCS += riscv/nemu/start.S \
           riscv/nemu/cte.c \
           riscv/nemu/trap.S \
           riscv/nemu/vme.c

```
这个文件就复杂多了，看起来它将 `isa` 和 `platform` 解耦了，单独包含了两个可包含文件；还定义了一个可以传递到.c 文件中的宏`ISA_H`，看来它引入了 riscv 的一些符号。

后面，它更新了 `COMMON_CFLAGS`、`LDFLAGS` 等变量。

## 三、目标构建

执行`make ARCH=riscv32-nemu ALL=add run`。

```Makefile
run: image
	$(MAKE) -C $(NEMU_HOME) ISA=$(ISA) run ARGS="$(NEMUFLAGS)" IMG=$(IMAGE).bin
```

依赖 `image` 的依赖：`$(IMAGE).elf`、`image-dep`，这两个依赖依次执行。


### 一、$(IMAGE).elf

```Makefile
$(IMAGE).elf: $(OBJS) $(LIBS)
	@echo + LD "->" $(IMAGE_REL).elf
	@$(LD) $(LDFLAGS) -o $(IMAGE).elf --start-group $(LINKAGE) --end-group
```

这里将会编译 `add.c`、以及 `kilb.a`、`am.a`。

之后，就会链接成`$(IMAGE).elf`。

### 二、image-dep
```Makefile
image-dep: $(OBJS) $(LIBS)
	@echo \# Creating image [$(ARCH)]
```

依赖在上面已经编译结束了，这里直接打印一个信息就结束了。这样 image 的依赖就解决完了。接着：

```Makefile
image: $(IMAGE).elf
	@$(OBJDUMP) -d $(IMAGE).elf > $(IMAGE).txt
	@echo + OBJCOPY "->" $(IMAGE_REL).bin
	@$(OBJCOPY) -S --set-section-flags .bss=alloc,contents -O binary $(IMAGE).elf $(IMAGE).bin
```
这里是将 $(IMAGE).elf 的二进制代码抽取出来，以供在 nemu 上运行。

### 做了什么

看看这做了什么：

```Makefile
CROSS_COMPILE := riscv64-linux-gnu-
COMMON_CFLAGS := -fno-pic -march=rv64g -mcmodel=medany -mstrict-align
CFLAGS        += $(COMMON_CFLAGS) -static
ASFLAGS       += $(COMMON_CFLAGS) -O0
LDFLAGS       += -melf64lriscv

# overwrite ARCH_H defined in $(AM_HOME)/Makefile
ARCH_H := arch/riscv.h
...
AM_SRCS := platform/nemu/trm.c \
           platform/nemu/ioe/ioe.c \
           platform/nemu/ioe/timer.c \
           platform/nemu/ioe/input.c \
           platform/nemu/ioe/gpu.c \
           platform/nemu/ioe/audio.c \
           platform/nemu/ioe/disk.c \
           platform/nemu/mpe.c

CFLAGS    += -fdata-sections -ffunction-sections
LDFLAGS   += -T $(AM_HOME)/scripts/linker.ld \
             --defsym=_pmem_start=0x80000000 --defsym=_entry_offset=0x0
LDFLAGS   += --gc-sections -e _start
NEMUFLAGS += -l $(shell dirname $(IMAGE).elf)/nemu-log.txt
NEMUFLAGS += -e $(IMAGE).elf # elf
NEMUFLAGS += -b # 自动执行


CFLAGS += -DMAINARGS=\"$(mainargs)\"
CFLAGS += -I$(AM_HOME)/am/src/platform/nemu/include
.PHONY: $(AM_HOME)/am/src/platform/nemu/trm.c

image: $(IMAGE).elf
	@$(OBJDUMP) -d $(IMAGE).elf > $(IMAGE).txt
	@echo + OBJCOPY "->" $(IMAGE_REL).bin
	@$(OBJCOPY) -S --set-section-flags .bss=alloc,contents -O binary $(IMAGE).elf $(IMAGE).bin

run: image
	$(MAKE) -C $(NEMU_HOME) ISA=$(ISA) run ARGS="$(NEMUFLAGS)" IMG=$(IMAGE).bin

gdb: image
	$(MAKE) -C $(NEMU_HOME) ISA=$(ISA) gdb ARGS="$(NEMUFLAGS)" IMG=$(IMAGE).bin

...

CFLAGS  += -DISA_H=\"riscv/riscv.h\"
COMMON_CFLAGS += -march=rv32im_zicsr -mabi=ilp32   # overwrite
LDFLAGS       += -melf32lriscv                     # overwrite

AM_SRCS += riscv/nemu/start.S \
           riscv/nemu/cte.c \
           riscv/nemu/trap.S \
           riscv/nemu/vme.c


```

可以看到，由于要编译出跨平台的文件，我们在编译目标的时候，要将 image 编译成能在 riscv32 的平台能跑的程序，因此它定义了 CROSS_COMPILE。

另外还加了一些需要设置 riscv 的参数 ，并把它并入到了 CFLAGS、LDFLAGS。


- **`-fno-pic`**：
  禁用生成位置无关代码（Position Independent Code, PIC）。PIC 通常用于动态链接库，因为它允许代码在内存中的任何位置运行。在某些情况下，如果你不需要动态链接，或者出于性能考虑，你可能会禁用它。

- **`-march=rv64g`**：
  设置目标架构为 RISC-V 64位，`G` 表示支持标准整数指令集（`I`）、乘法和除法指令集（`M`）、原子指令集（`A`）、单精度浮点（`F`）、双精度浮点（`D`）和压缩指令集（`C`）。这告诉编译器生成针对此架构优化的代码。

- **`-mcmodel=medany`**：
  这个选项定义了代码和数据的地址模型，适用于 RISC-V。`medany` 模型适合中等大小的可执行文件，允许代码和数据位于任意位置，但假设所有符号的地址相对于程序计数器（PC）的偏移适中。

- **`-mstrict-align`**：
  强制数据对齐，不允许编译器生成未对齐的内存访问指令。这在一些严格要求对齐的架构上可以避免性能损失或程序错误。



- **`-static`**：
  生成完全静态链接的可执行文件。静态链接的程序包含了所有必需的库函数的副本，不需要依赖系统中的共享库，提高了程序的移植性。



- **`-O0`**：
  禁用优化。这个选项告诉编译器不进行任何优化，通常用于调试，确保编译的代码直接对应源代码，便于跟踪和调试。


- **`-melf64lriscv`**：
  指定链接器输出格式为 RISC-V 64位 ELF 格式。ELF（可执行与链接格式）是一种广泛使用的文件格式，用于定义不同类型的文件如可执行文件、目标文件、共享库等。

总体来说，这些编译和链接选项针对 RISC-V 64位架构进行了特定的优化和设置，适用于生成高效且独立的程序。

- **`-fdata-sections`**：
  这个选项指导编译器将不同的数据（变量）放置在不同的段（sections）中。这有助于链接器在链接时进行优化，比如删除未使用的数据段，从而减小最终可执行文件的大小。

- **`-ffunction-sections`**：
  与 `-fdata-sections` 类似，这个选项指导编译器将每个函数放置在独立的段中。这样做可以让链接器有机会在最终的可执行文件中移除未被引用的函数，尤其是在结合链接器的 `--gc-sections` 选项时。


- **`-T $(AM_HOME)/scripts/linker.ld`**：
  指定链接器脚本的位置，这里使用的是一个环境变量 `AM_HOME`。链接器脚本用于控制链接过程，定义了各种内存布局、段到地址的映射等。这是为嵌入式系统或操作系统内核等需要精确控制内存布局的场合常用的做法。

- **`--defsym=_pmem_start=0x80000000`** 和 **`--defsym=_entry_offset=0x0`**：
  这两个选项定义了链接时的符号。`--defsym` 用于创建一个符号并为其分配一个值。在这个例子中，`_pmem_start` 被设置为 `0x80000000` 和 `_entry_offset` 被设置为 `0x0`。这些符号在链接脚本中可能会被引用，用于指定程序的某些特定内存地址或偏移。

- **`--gc-sections`**：
  这个链接器选项与前面的 `-ffunction-sections` 和 `-fdata-sections` 一起工作，用来在链接阶段去除未使用的函数和数据段，从而减少生成的二进制文件大小。这对于内存受限的系统特别有用。

- **`-e _start`**：
  设置入口点为 `_start`。这是程序开始执行的地方，`-e` 选项用于指定入口点符号名。这在很多系统软件中非常重要，特别是在操作系统或裸机程序中，需要精确控制程序的启动过程。


这里添加到 `COMMON_CFLAGS` 的参数用于覆盖或指定编译时的架构和二进制接口（ABI）设置：

- **`-march=rv32im_zicsr`**：
  - **`-march=rv32im_zicsr`**：这个选项指定目标架构为 RISC-V 32位，具体到支持整数 (`I`)、乘法 (`M`) 指令和 CSR 寄存器 (`Zicsr`)。`Zicsr` 表示支持 CSR (Control and Status Register) 指令扩展，用于控制和状态寄存器的操作，这对于系统级编程特别重要。

- **`-mabi=ilp32`**：
  - **`-mabi=ilp32`**：设置使用的 ABI（Application Binary Interface）为 `ilp32`，这意味着整数和指针都是 32位，符合 32位系统的处理方式。这个设置确保生成的代码在32位 RISC-V 架构上能正确执行。



- **`-melf32lriscv`**：
  - **`-melf32lriscv`**：这个链接器选项指定生成的可执行文件格式为 32位 RISC-V ELF 格式。ELF（Executable and Linkable Format）是一种广泛使用的文件格式，用于定义可执行文件、目标代码、共享库等。这里的设置是为了确保链接器输出的是符合 32位 RISC-V 架构的 ELF 文件。

这些选项通常用于交叉编译环境，其中开发环境（如你的电脑）和目标环境（如一个嵌入式设备）的架构不同。例如，如果你正在为一个基于 RISC-V 32位架构的嵌入式系统开发软件，就需要配置这些选项以确保代码能在目标硬件上正确运行。

这样配置的编译器和链接器选项允许开发者精确控制生成代码的架构特性和二进制格式，以符合特定硬件和操作系统的要求。

这样设置，就生成了适合在 nemu 上运行的 bin 文件了。



这时候，继续执行：

```Makefile
run: image
	$(MAKE) -C $(NEMU_HOME) ISA=$(ISA) run ARGS="$(NEMUFLAGS)" IMG=$(IMAGE).bin
```

这个命令表明，接下来将会继续编译 nemu，并且传入了一些参数。

## 四、编译 nemu

```Makefile
#***************************************************************************************
# Copyright (c) 2014-2022 Zihao Yu, Nanjing University
#
# NEMU is licensed under Mulan PSL v2.
# You can use this software according to the terms and conditions of the Mulan PSL v2.
# You may obtain a copy of Mulan PSL v2 at:
#          http://license.coscl.org.cn/MulanPSL2
#
# THIS SOFTWARE IS PROVIDED ON AN "AS IS" BASIS, WITHOUT WARRANTIES OF ANY KIND,
# EITHER EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO NON-INFRINGEMENT,
# MERCHANTABILITY OR FIT FOR A PARTICULAR PURPOSE.
#
# See the Mulan PSL v2 for more details.
#**************************************************************************************/

# Sanity check
ifeq ($(wildcard $(NEMU_HOME)/src/nemu-main.c),)
  $(error NEMU_HOME=$(NEMU_HOME) is not a NEMU repo)
endif

# Include variables and rules generated by menuconfig
-include $(NEMU_HOME)/include/config/auto.conf
-include $(NEMU_HOME)/include/config/auto.conf.cmd

remove_quote = $(patsubst "%",%,$(1))

# Extract variabls from menuconfig
GUEST_ISA ?= $(call remove_quote,$(CONFIG_ISA))
ENGINE ?= $(call remove_quote,$(CONFIG_ENGINE))
NAME    = $(GUEST_ISA)-nemu-$(ENGINE)

# Include all filelist.mk to merge file lists
FILELIST_MK = $(shell find -L ./src -name "filelist.mk")
include $(FILELIST_MK)

# Filter out directories and files in blacklist to obtain the final set of source files
DIRS-BLACKLIST-y += $(DIRS-BLACKLIST)
SRCS-BLACKLIST-y += $(SRCS-BLACKLIST) $(shell find -L $(DIRS-BLACKLIST-y) -name "*.c")
SRCS-y += $(shell find -L $(DIRS-y) -name "*.c")
SRCS = $(filter-out $(SRCS-BLACKLIST-y),$(SRCS-y))

# Extract compiler and options from menuconfig
CC = $(call remove_quote,$(CONFIG_CC))
CFLAGS_BUILD += $(call remove_quote,$(CONFIG_CC_OPT))
CFLAGS_BUILD += $(if $(CONFIG_CC_LTO),-flto,)
CFLAGS_BUILD += $(if $(CONFIG_CC_DEBUG),-Og -ggdb3,)
CFLAGS_BUILD += $(if $(CONFIG_CC_ASAN),-fsanitize=address,)
CFLAGS_TRACE += -DITRACE_COND=$(if $(CONFIG_ITRACE_COND),$(call remove_quote,$(CONFIG_ITRACE_COND)),true)
CFLAGS  += $(CFLAGS_BUILD) $(CFLAGS_TRACE) -D__GUEST_ISA__=$(GUEST_ISA)
LDFLAGS += $(CFLAGS_BUILD)

# Include rules for menuconfig
include $(NEMU_HOME)/scripts/config.mk

ifdef CONFIG_TARGET_AM
include $(AM_HOME)/Makefile
LINKAGE += $(ARCHIVES)
else
# Include rules to build NEMU
include $(NEMU_HOME)/scripts/native.mk
endif

```

由于我们的 nemu 依然要在 native 上跑，因此这里依然要 native.mk 来干这事。编译好了之后，设置 IMG 为刚才编译的 bin，就好了:
```bash
$ make ARCH=riscv32-nemu  ALL=add run
# Building add-run [riscv32-nemu]
+ CC tests/add.c
# Building am-archive [riscv32-nemu]
+ CC src/platform/nemu/trm.c
+ CC src/platform/nemu/ioe/ioe.c
+ CC src/platform/nemu/ioe/timer.c
+ CC src/platform/nemu/ioe/input.c
+ CC src/platform/nemu/ioe/gpu.c
+ CC src/platform/nemu/ioe/audio.c
+ CC src/platform/nemu/ioe/disk.c
+ CC src/platform/nemu/mpe.c
+ AS src/riscv/nemu/start.S
+ CC src/riscv/nemu/cte.c
+ AS src/riscv/nemu/trap.S
+ CC src/riscv/nemu/vme.c
+ AR -> build/am-riscv32-nemu.a
# Building klib-archive [riscv32-nemu]
+ CC src/stdlib.c
+ CC src/string.c
+ CC src/int64.c
+ CC src/stdio.c
+ CC src/cpp.c
+ AR -> build/klib-riscv32-nemu.a
+ LD -> build/add-riscv32-nemu.elf
# Creating image [riscv32-nemu]
+ OBJCOPY -> build/add-riscv32-nemu.bin
+ CC src/device/device.c
+ CC src/device/alarm.c
+ CC src/device/intr.c
+ CC src/device/serial.c
+ CC src/device/timer.c
+ CC src/device/keyboard.c
+ CC src/device/vga.c
+ CC src/device/audio.c
+ CC src/device/disk.c
+ CC src/nemu-main.c
+ CC src/engine/interpreter/init.c
+ CC src/engine/interpreter/hostcall.c
+ CC src/isa/riscv32/system/mmu.c
+ CC src/isa/riscv32/system/intr.c
+ CC src/isa/riscv32/inst.c
+ CC src/isa/riscv32/init.c
+ CC src/isa/riscv32/difftest/dut.c
+ CC src/isa/riscv32/logo.c
+ CC src/isa/riscv32/reg.c
+ CC src/device/io/port-io.c
+ CC src/device/io/mmio.c
+ CC src/device/io/map.c
+ CC src/cpu/difftest/dut.c
+ CC src/cpu/difftest/ref.c
+ CC src/cpu/cpu-exec.c
+ CC src/monitor/sdb/expr.c
+ CC src/monitor/sdb/watchpoint.c
+ CC src/monitor/sdb/sdb.c
+ CC src/monitor/monitor.c
+ CC src/utils/timer.c
+ CC src/utils/trace.c
+ CC src/utils/log.c
+ CC src/utils/state.c
+ CC src/memory/vaddr.c
+ CC src/memory/paddr.c
+ LD /home/luyoung/ysyx-workbench/nemu/build/riscv32-nemu-interpreter
```

这样的流程很符合预期。

## 五、总结

通过分析了两种不同的目标构建方式，了解了整个项目的架构设计。




















