---
title: nemu框架细节研究（二）
date: 2024-09-08 11:22:33
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
前文细致地研究了命令 `make ARCH=native ALL=add run` 的运行过程，对于Makefile中的行为有了很准确地把握（没有任何疑点，可以随意修改 Makefile 且保证不会出错）。这篇文章继续研究命令`make ARCH=riscv32-nemu ALL=add run`的行为。

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

##  目标构建

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

这里将会编译 `string.c`、以及 `kilb.a`、`am.a`。

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

这时候，继续执行：

```Makefile
run: image
	$(MAKE) -C $(NEMU_HOME) ISA=$(ISA) run ARGS="$(NEMUFLAGS)" IMG=$(IMAGE).bin
```

这个命令表明，接下来将会继续编译 nemu，并且传入了一些参数。
















