---
title: "NEMU 框架细节研究（一）：native 构建入口"
date: 2024-09-07 11:22:33
tags:
  - "NEMU"
  - "build system"
  - "ysyx"
categories:
  - "Computer Architecture"
---

<!--more-->

## 一、前言

现在做到了 PA2.3后面，遇到的问题有点多，感觉总在棉花上踩。遇到了一个 bug 迟迟无法解决：用 native 且没有定义 `__NATIVE_USE_KLIB__` 跑 Bad Apples 会失败，但是用 nemu 跑竟然会成功。这说明我的 klib 比 glibc 还完备吗？看了两个小时，也没看出什么。思考良久，感觉还是对项目构建过程缺乏细节上的了解，这成了学习进度上的瓶颈。于是决定好好写几篇文章细致地研究一下这个项目的构建细节，计划花 5 天左右来做这件事。

本文会横向比较在 `/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/` 中两种架构 native 和 riscv32-nemu 的构建差异。


## 二、native 开始的地方
在 `/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/` 下 有四个文件：
- tests/
- include/
- Makefile
- .gitignore


`native` 的宗旨是把 `tests/`中的文件编译一个，或者多份，这取决于 `ALL` 变量的的定义情况，可以在 `Makefile` 中看到：

```Makefile
...
ALL = $(basename $(notdir $(shell find tests/. -name "*.c")))
...
```

如果在编译命令：`make ARCH=native ALL=add run` 中定义了，显然这个变量具有最高优先级，它会覆盖 `Makefile` 中的定义，因此 `ALL=add`。

接下来，仔细研究`Makefile`:

```Makefile
.PHONY: all run gdb clean latest $(ALL)

RESULT = .result
$(shell > $(RESULT))

COLOR_RED   = \033[1;31m
COLOR_GREEN = \033[1;32m
COLOR_NONE  = \033[0m

ALL = $(basename $(notdir $(shell find tests/. -name "*.c")))

all: $(addprefix Makefile., $(ALL))
	@echo "test list [$(words $(ALL)) item(s)]:" $(ALL)

$(ALL): %: Makefile.%



Makefile.%: tests/%.c latest
	@/bin/echo -e "NAME = $*\nSRCS = $<\ninclude $${AM_HOME}/Makefile" > $@
	@if make -s -f $@ ARCH=$(ARCH) $(MAKECMDGOALS); then \
		printf "[%14s] $(COLOR_GREEN)PASS$(COLOR_NONE)\n" $* >> $(RESULT); \
	else \
		printf "[%14s] $(COLOR_RED)***FAIL***$(COLOR_NONE)\n" $* >> $(RESULT); \
	fi
#-@rm -f Makefile.$*

run: all
	@cat $(RESULT)
	@rm $(RESULT)

gdb: all

clean:
	rm -rf Makefile.* build/

latest:

```

当我执行 `make ARCH=native ALL=add run`的时候，由于目标是 `run`，因此 `make` 会找 `run` 的依赖 `all`:

```Makefile
run: all
	@cat $(RESULT)
	@rm $(RESULT)
```

在 `Makefile` 被解析的时候，下面的语句就会被执行：
```Makefile
RESULT = .result
$(shell > $(RESULT))
```

这里需要注意的是，并不是需要查找一个变量的时候，这个变量才会被赋值。正确的模型是：`make` 执行的时候，生成目标之前，整个 `Makefile` 自上而下，线性地都会被解析一遍，包括变量赋值、`$(shell ...)` 执行，就像是预处理。之后，才开始分析依赖路径，一步一步执行。

比如：
```Makefile
A=1
$(info $(A))

A=2
$(info $(A))

all:
	@echo $(A)+$(A)

A=3
$(info $(A))


```

在 `make all` 的时候，它会 从上而下解析整个 `Makefile` 文件，然后才会执行 all 目标，因此它首先会打印：

```bash
$ make all
1
2
3
3+3
```

搞清楚这些之后，继续研究这个`Makefile`。

显然，这个文件没有其它的解析的东西了(`ALL` 已经通过 `make` 参数传进来了，它具有最高优先级，会覆盖 本文件的`ALL`)。这下开始构建目标`run`:
- `run`的依赖是 `all`
- 接着构建 `all`,`all`的依赖是 `$(addprefix Makefile., $(ALL))`，也就是`Makefile.add`:
- `Makefile.add`的依赖为：`tests/add.c latest`；
- `tests/add.c latest` 在本地存在；因此开始构建`Makefile.add`：


`@/bin/echo -e "NAME = $*\nSRCS = $<\ninclude $${AM_HOME}/Makefile" > $@` 的含义是将`NAME = $*\nSRCS = $<\ninclude $${AM_HOME}/Makefile` 创建到目标 `Makefile.add`，这显然就生成了一个文件：`Makefile.add`：

```Makefile
NAME = add
SRCS = tests/add.c
include /home/luyoung/ysyx-workbench/abstract-machine/Makefile

```

接着执行剩下命令：
```Makefile
@if make -s -f $@ ARCH=$(ARCH) $(MAKECMDGOALS); then \
		printf "[%14s] $(COLOR_GREEN)PASS$(COLOR_NONE)\n" $* >> $(RESULT); \
	else \
		printf "[%14s] $(COLOR_RED)***FAIL***$(COLOR_NONE)\n" $* >> $(RESULT); \
	fi
```

含义是目标（-f） `Makefile` 为 `Makefile.add`，相当于对在 `Makefile.add` 执行 `make ARCH=native run`。$(MAKECMDGOALS)这个参数是 make 的内建变量，指得是目标 `run`。

接着就来到了`Makefile.add`：

```Makefile
NAME = add
SRCS = tests/add.c
include /home/luyoung/ysyx-workbench/abstract-machine/Makefile

```

这个文件包含了 `/home/luyoung/ysyx-workbench/abstract-machine/Makefile` ，也就是这个文件：

```Makefile
# Makefile for AbstractMachine Kernels and Libraries

### *Get a more readable version of this Makefile* by `make html` (requires python-markdown)
html:
	cat Makefile | sed 's/^\([^#]\)/    \1/g' | markdown_py > Makefile.html
.PHONY: html

## 1. Basic Setup and Checks

### Default to create a bare-metal kernel image
ifeq ($(MAKECMDGOALS),)
  MAKECMDGOALS  = image
  .DEFAULT_GOAL = image
endif

### Override checks when `make clean/clean-all/html`
ifeq ($(findstring $(MAKECMDGOALS),clean|clean-all|html),)

### Print build info message
$(info # Building $(NAME)-$(MAKECMDGOALS) [$(ARCH)])

### Check: environment variable `$AM_HOME` looks sane
ifeq ($(wildcard $(AM_HOME)/am/include/am.h),)
  $(error $$AM_HOME must be an AbstractMachine repo)
endif

### Check: environment variable `$ARCH` must be in the supported list
ARCHS = $(basename $(notdir $(shell ls $(AM_HOME)/scripts/*.mk)))
ifeq ($(filter $(ARCHS), $(ARCH)), )
  $(error Expected $$ARCH in {$(ARCHS)}, Got "$(ARCH)")
endif

### Extract instruction set architecture (`ISA`) and platform from `$ARCH`. Example: `ARCH=x86_64-qemu -> ISA=x86_64; PLATFORM=qemu`
ARCH_SPLIT = $(subst -, ,$(ARCH))
ISA        = $(word 1,$(ARCH_SPLIT))
PLATFORM   = $(word 2,$(ARCH_SPLIT))

### Check if there is something to build
ifeq ($(flavor SRCS), undefined)
  $(error Nothing to build)
endif

### Checks end here
endif

## 2. General Compilation Targets

### Create the destination directory (`build/$ARCH`)
WORK_DIR  = $(shell pwd)
DST_DIR   = $(WORK_DIR)/build/$(ARCH)
$(shell mkdir -p $(DST_DIR))

### Compilation targets (a binary image or archive)
IMAGE_REL = build/$(NAME)-$(ARCH)
IMAGE     = $(abspath $(IMAGE_REL))
ARCHIVE   = $(WORK_DIR)/build/$(NAME)-$(ARCH).a

### Collect the files to be linked: object files (`.o`) and libraries (`.a`)
OBJS      = $(addprefix $(DST_DIR)/, $(addsuffix .o, $(basename $(SRCS))))
LIBS     := $(sort $(LIBS) am klib) # lazy evaluation ("=") causes infinite recursions
LINKAGE   = $(OBJS) $(addsuffix -$(ARCH).a, $(join $(addsuffix /build/, $(addprefix $(AM_HOME)/, $(LIBS))),  $(LIBS) ))

## 3. General Compilation Flags

### (Cross) compilers, e.g., mips-linux-gnu-g++
AS        = $(CROSS_COMPILE)gcc
CC        = $(CROSS_COMPILE)gcc
CXX       = $(CROSS_COMPILE)g++
LD        = $(CROSS_COMPILE)ld
AR        = $(CROSS_COMPILE)ar
OBJDUMP   = $(CROSS_COMPILE)objdump
OBJCOPY   = $(CROSS_COMPILE)objcopy
READELF   = $(CROSS_COMPILE)readelf



### Compilation flags
INC_PATH += $(WORK_DIR)/include $(addsuffix /include/, $(addprefix $(AM_HOME)/, $(LIBS)))

INCFLAGS += $(addprefix -I, $(INC_PATH))

ARCH_H := arch/$(ARCH).h
CFLAGS   += -O2 -MMD -Wall -Werror $(INCFLAGS) \
            -D__ISA__=\"$(ISA)\" -D__ISA_$(shell echo $(ISA) | tr a-z A-Z)__ \
            -D__ARCH__=$(ARCH) -D__ARCH_$(shell echo $(ARCH) | tr a-z A-Z | tr - _) \
            -D__PLATFORM__=$(PLATFORM) -D__PLATFORM_$(shell echo $(PLATFORM) | tr a-z A-Z | tr - _) \
            -DARCH_H=\"$(ARCH_H)\" \
            -fno-asynchronous-unwind-tables -fno-builtin -fno-stack-protector \
            -Wno-main -U_FORTIFY_SOURCE -fvisibility=hidden
CXXFLAGS +=  $(CFLAGS) -ffreestanding -fno-rtti -fno-exceptions
ASFLAGS  += -MMD $(INCFLAGS)
LDFLAGS  += -z noexecstack

## 4. Arch-Specific Configurations

### Paste in arch-specific configurations (e.g., from `scripts/x86_64-qemu.mk`)
-include $(AM_HOME)/scripts/$(ARCH).mk

### Fall back to native gcc/binutils if there is no cross compiler
ifeq ($(wildcard $(shell which $(CC))),)
  $(info #  $(CC) not found; fall back to default gcc and binutils)
  CROSS_COMPILE :=
endif

## 5. Compilation Rules

### Rule (compile): a single `.c` -> `.o` (gcc)
$(DST_DIR)/%.o: %.c
	@mkdir -p $(dir $@) && echo + CC $<
	@$(CC) -std=gnu11 $(CFLAGS) -c -o $@ $(realpath $<)

### Rule (compile): a single `.cc` -> `.o` (g++)
$(DST_DIR)/%.o: %.cc
	@mkdir -p $(dir $@) && echo + CXX $<
	@$(CXX) -std=c++17 $(CXXFLAGS) -c -o $@ $(realpath $<)

### Rule (compile): a single `.cpp` -> `.o` (g++)
$(DST_DIR)/%.o: %.cpp
	@mkdir -p $(dir $@) && echo + CXX $<
	@$(CXX) -std=c++17 $(CXXFLAGS) -c -o $@ $(realpath $<)

### Rule (compile): a single `.S` -> `.o` (gcc, which preprocesses and calls as)
$(DST_DIR)/%.o: %.S
	@mkdir -p $(dir $@) && echo + AS $<
	@$(AS) $(ASFLAGS) -c -o $@ $(realpath $<)

### Rule (recursive make): build a dependent library (am, klib, ...)
$(LIBS): %:
	@$(MAKE) -s -C $(AM_HOME)/$* archive

### Rule (link): objects (`*.o`) and libraries (`*.a`) -> `IMAGE.elf`, the final ELF binary to be packed into image (ld)
$(IMAGE).elf: $(OBJS) $(LIBS)
	@echo + LD "->" $(IMAGE_REL).elf
	@$(LD) $(LDFLAGS) -o $(IMAGE).elf --start-group $(LINKAGE) --end-group

### Rule (archive): objects (`*.o`) -> `ARCHIVE.a` (ar)
$(ARCHIVE): $(OBJS)
	@echo + AR "->" $(shell realpath $@ --relative-to .)
	@$(AR) rcs $(ARCHIVE) $(OBJS)

### Rule (`#include` dependencies): paste in `.d` files generated by gcc on `-MMD`
-include $(addprefix $(DST_DIR)/, $(addsuffix .d, $(basename $(SRCS))))

## 6. Miscellaneous

### Build order control
image: image-dep
archive: $(ARCHIVE)
image-dep: $(OBJS) $(LIBS)
	@echo \# Creating image [$(ARCH)]
.PHONY: image image-dep archive run $(LIBS)

### Clean a single project (remove `build/`)
clean:
	rm -rf Makefile.html $(WORK_DIR)/build/
.PHONY: clean

### Clean all sub-projects within depth 2 (and ignore errors)
CLEAN_ALL = $(dir $(shell find . -mindepth 2 -name Makefile))
clean-all: $(CLEAN_ALL) clean
$(CLEAN_ALL):
	-@$(MAKE) -s -C $@ clean
.PHONY: clean-all $(CLEAN_ALL)

```


这个文件太长，而且里面还包含了很多的 其它可包含文件，因此以下分析按照`文件解析`、`目标构建`分两步。

### 1. 文件包含

上面的文件包含了：
- `-include $(AM_HOME)/scripts/$(ARCH).mk`
- `-include $(addprefix $(DST_DIR)/, $(addsuffix .d, $(basename $(SRCS))))`

由于 `ARCH=native`，因此它会包含: `/home/luyoung/ysyx-workbench/abstract-machine/scripts/native.mk`:

```Makefile
AM_SRCS := native/trm.c \
           native/ioe.c \
           native/cte.c \
           native/trap.S \
           native/vme.c \
           native/mpe.c \
           native/platform.c \
           native/ioe/input.c \
           native/ioe/timer.c \
           native/ioe/gpu.c \
           native/ioe/uart.c \
           native/ioe/audio.c \
           native/ioe/disk.c \

CFLAGS  += -fpie
ASFLAGS += -fpie -pie
comma = ,
LDFLAGS_CXX = $(addprefix -Wl$(comma), $(LDFLAGS))

image:
	@echo + LD "->" $(IMAGE_REL)
	@g++ -pie -o $(IMAGE) -Wl,--whole-archive $(LINKAGE) -Wl,-no-whole-archive $(LDFLAGS_CXX) -lSDL2 -ldl

run: image
	$(IMAGE)

gdb: image
	gdb -ex "handle SIGUSR1 SIGUSR2 SIGSEGV noprint nostop" $(IMAGE)
```

第二个文件为：`/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/native/tests/add.d`，这个文件由编译命令 `gcc on` `-MMD` 生成，也是一个可包含文件，他就相当于依赖：

```Makefile
/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/native/tests/add.o: \
 /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/tests/add.c \
 /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include/trap.h \
 /home/luyoung/ysyx-workbench/abstract-machine/am/include/am.h \
 /home/luyoung/ysyx-workbench/abstract-machine/am/include/arch/native.h \
 /home/luyoung/ysyx-workbench/abstract-machine/am/include/amdev.h \
 /home/luyoung/ysyx-workbench/abstract-machine/klib/include/klib.h \
 /home/luyoung/ysyx-workbench/abstract-machine/klib/include/klib-macros.h

```

之后整个文件就变成了如下的样子：

```Makefile
NAME = add
SRCS = tests/add.c
#include /home/luyoung/ysyx-workbench/abstract-machine/Makefile

# Makefile for AbstractMachine Kernels and Libraries

### *Get a more readable version of this Makefile* by `make html` (requires python-markdown)
html:
	cat Makefile | sed 's/^\([^#]\)/    \1/g' | markdown_py > Makefile.html
.PHONY: html

## 1. Basic Setup and Checks

### Default to create a bare-metal kernel image
ifeq ($(MAKECMDGOALS),)
  MAKECMDGOALS  = image
  .DEFAULT_GOAL = image
endif

### Override checks when `make clean/clean-all/html`
ifeq ($(findstring $(MAKECMDGOALS),clean|clean-all|html),)

### Print build info message
$(info # Building $(NAME)-$(MAKECMDGOALS) [$(ARCH)])

### Check: environment variable `$AM_HOME` looks sane
ifeq ($(wildcard $(AM_HOME)/am/include/am.h),)
  $(error $$AM_HOME must be an AbstractMachine repo)
endif

### Check: environment variable `$ARCH` must be in the supported list
ARCHS = $(basename $(notdir $(shell ls $(AM_HOME)/scripts/*.mk)))
ifeq ($(filter $(ARCHS), $(ARCH)), )
  $(error Expected $$ARCH in {$(ARCHS)}, Got "$(ARCH)")
endif

### Extract instruction set architecture (`ISA`) and platform from `$ARCH`. Example: `ARCH=x86_64-qemu -> ISA=x86_64; PLATFORM=qemu`
ARCH_SPLIT = $(subst -, ,$(ARCH))
ISA        = $(word 1,$(ARCH_SPLIT))
PLATFORM   = $(word 2,$(ARCH_SPLIT))

### Check if there is something to build
ifeq ($(flavor SRCS), undefined)
  $(error Nothing to build)
endif

### Checks end here
endif

## 2. General Compilation Targets

### Create the destination directory (`build/$ARCH`)
WORK_DIR  = $(shell pwd)
DST_DIR   = $(WORK_DIR)/build/$(ARCH)
$(shell mkdir -p $(DST_DIR))

### Compilation targets (a binary image or archive)
IMAGE_REL = build/$(NAME)-$(ARCH)
IMAGE     = $(abspath $(IMAGE_REL))
ARCHIVE   = $(WORK_DIR)/build/$(NAME)-$(ARCH).a

### Collect the files to be linked: object files (`.o`) and libraries (`.a`)
OBJS      = $(addprefix $(DST_DIR)/, $(addsuffix .o, $(basename $(SRCS))))
LIBS     := $(sort $(LIBS) am klib) # lazy evaluation ("=") causes infinite recursions
LINKAGE   = $(OBJS) $(addsuffix -$(ARCH).a, $(join $(addsuffix /build/, $(addprefix $(AM_HOME)/, $(LIBS))),  $(LIBS) ))

## 3. General Compilation Flags

### (Cross) compilers, e.g., mips-linux-gnu-g++
AS        = $(CROSS_COMPILE)gcc
CC        = $(CROSS_COMPILE)gcc
CXX       = $(CROSS_COMPILE)g++
LD        = $(CROSS_COMPILE)ld
AR        = $(CROSS_COMPILE)ar
OBJDUMP   = $(CROSS_COMPILE)objdump
OBJCOPY   = $(CROSS_COMPILE)objcopy
READELF   = $(CROSS_COMPILE)readelf



### Compilation flags
INC_PATH += $(WORK_DIR)/include $(addsuffix /include/, $(addprefix $(AM_HOME)/, $(LIBS)))

INCFLAGS += $(addprefix -I, $(INC_PATH))

ARCH_H := arch/$(ARCH).h
CFLAGS   += -O2 -MMD -Wall -Werror $(INCFLAGS) \
            -D__ISA__=\"$(ISA)\" -D__ISA_$(shell echo $(ISA) | tr a-z A-Z)__ \
            -D__ARCH__=$(ARCH) -D__ARCH_$(shell echo $(ARCH) | tr a-z A-Z | tr - _) \
            -D__PLATFORM__=$(PLATFORM) -D__PLATFORM_$(shell echo $(PLATFORM) | tr a-z A-Z | tr - _) \
            -DARCH_H=\"$(ARCH_H)\" \
            -fno-asynchronous-unwind-tables -fno-builtin -fno-stack-protector \
            -Wno-main -U_FORTIFY_SOURCE -fvisibility=hidden
CXXFLAGS +=  $(CFLAGS) -ffreestanding -fno-rtti -fno-exceptions
ASFLAGS  += -MMD $(INCFLAGS)
LDFLAGS  += -z noexecstack

## 4. Arch-Specific Configurations

### Paste in arch-specific configurations (e.g., from `scripts/x86_64-qemu.mk`)
# -include $(AM_HOME)/scripts/$(ARCH).mk
AM_SRCS := native/trm.c \
           native/ioe.c \
           native/cte.c \
           native/trap.S \
           native/vme.c \
           native/mpe.c \
           native/platform.c \
           native/ioe/input.c \
           native/ioe/timer.c \
           native/ioe/gpu.c \
           native/ioe/uart.c \
           native/ioe/audio.c \
           native/ioe/disk.c \

CFLAGS  += -fpie
ASFLAGS += -fpie -pie
comma = ,
LDFLAGS_CXX = $(addprefix -Wl$(comma), $(LDFLAGS))

image:
	@echo + LD "->" $(IMAGE_REL)
	@g++ -pie -o $(IMAGE) -Wl,--whole-archive $(LINKAGE) -Wl,-no-whole-archive $(LDFLAGS_CXX) -lSDL2 -ldl

run: image
	$(IMAGE)

gdb: image
	gdb -ex "handle SIGUSR1 SIGUSR2 SIGSEGV noprint nostop" $(IMAGE)

### Fall back to native gcc/binutils if there is no cross compiler
ifeq ($(wildcard $(shell which $(CC))),)
  $(info #  $(CC) not found; fall back to default gcc and binutils)
  CROSS_COMPILE :=
endif

## 5. Compilation Rules

### Rule (compile): a single `.c` -> `.o` (gcc)
$(DST_DIR)/%.o: %.c
	@mkdir -p $(dir $@) && echo + CC $<
	@$(CC) -std=gnu11 $(CFLAGS) -c -o $@ $(realpath $<)

### Rule (compile): a single `.cc` -> `.o` (g++)
$(DST_DIR)/%.o: %.cc
	@mkdir -p $(dir $@) && echo + CXX $<
	@$(CXX) -std=c++17 $(CXXFLAGS) -c -o $@ $(realpath $<)

### Rule (compile): a single `.cpp` -> `.o` (g++)
$(DST_DIR)/%.o: %.cpp
	@mkdir -p $(dir $@) && echo + CXX $<
	@$(CXX) -std=c++17 $(CXXFLAGS) -c -o $@ $(realpath $<)

### Rule (compile): a single `.S` -> `.o` (gcc, which preprocesses and calls as)
$(DST_DIR)/%.o: %.S
	@mkdir -p $(dir $@) && echo + AS $<
	@$(AS) $(ASFLAGS) -c -o $@ $(realpath $<)

### Rule (recursive make): build a dependent library (am, klib, ...)
$(LIBS): %:
	@$(MAKE) -s -C $(AM_HOME)/$* archive

### Rule (link): objects (`*.o`) and libraries (`*.a`) -> `IMAGE.elf`, the final ELF binary to be packed into image (ld)
$(IMAGE).elf: $(OBJS) $(LIBS)
	@echo + LD "->" $(IMAGE_REL).elf
	@$(LD) $(LDFLAGS) -o $(IMAGE).elf --start-group $(LINKAGE) --end-group

### Rule (archive): objects (`*.o`) -> `ARCHIVE.a` (ar)
$(ARCHIVE): $(OBJS)
	@echo + AR "->" $(shell realpath $@ --relative-to .)
	@$(AR) rcs $(ARCHIVE) $(OBJS)

### Rule (`#include` dependencies): paste in `.d` files generated by gcc on `-MMD`
#-include $(addprefix $(DST_DIR)/, $(addsuffix .d, $(basename $(SRCS))))

/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/native/tests/add.o: \
 /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/tests/add.c \
 /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include/trap.h \
 /home/luyoung/ysyx-workbench/abstract-machine/am/include/am.h \
 /home/luyoung/ysyx-workbench/abstract-machine/am/include/arch/native.h \
 /home/luyoung/ysyx-workbench/abstract-machine/am/include/amdev.h \
 /home/luyoung/ysyx-workbench/abstract-machine/klib/include/klib.h \
 /home/luyoung/ysyx-workbench/abstract-machine/klib/include/klib-macros.h

## 6. Miscellaneous

### Build order control
image: image-dep
archive: $(ARCHIVE)
image-dep: $(OBJS) $(LIBS)
	@echo \# Creating image [$(ARCH)]
.PHONY: image image-dep archive run $(LIBS)

### Clean a single project (remove `build/`)
clean:
	rm -rf Makefile.html $(WORK_DIR)/build/
.PHONY: clean

### Clean all sub-projects within depth 2 (and ignore errors)
CLEAN_ALL = $(dir $(shell find . -mindepth 2 -name Makefile))
clean-all: $(CLEAN_ALL) clean
$(CLEAN_ALL):
	-@$(MAKE) -s -C $@ clean
.PHONY: clean-all $(CLEAN_ALL)

```

### 2. 文件解析

首先自上而下解析文件。

- `NAME` = `add`

- `SRCS` = `test/add.c`

```Makefile

# 判断目标是否为空，如果为空，将目标定为 image, 默认目标也定为 image
ifeq ($(MAKECMDGOALS),)
  MAKECMDGOALS  = image
  .DEFAULT_GOAL = image
endif

# 做一些安全检查
### Override checks when `make clean/clean-all/html`
ifeq ($(findstring $(MAKECMDGOALS),clean|clean-all|html),)

### Print build info message
$(info # Building $(NAME)-$(MAKECMDGOALS) [$(ARCH)])

### Check: environment variable `$AM_HOME` looks sane
ifeq ($(wildcard $(AM_HOME)/am/include/am.h),)
  $(error $$AM_HOME must be an AbstractMachine repo)
endif

### Check: environment variable `$ARCH` must be in the supported list
ARCHS = $(basename $(notdir $(shell ls $(AM_HOME)/scripts/*.mk)))
ifeq ($(filter $(ARCHS), $(ARCH)), )
  $(error Expected $$ARCH in {$(ARCHS)}, Got "$(ARCH)")
endif

### Extract instruction set architecture (`ISA`) and platform from `$ARCH`. Example: `ARCH=x86_64-qemu -> ISA=x86_64; PLATFORM=qemu`

# 这里就会解析传入的 native，ARCH=native
# 解析完后就是 ARCH_SPLIT=native
# ISA=native
# PLATFORM= (为空)
ARCH_SPLIT = $(subst -, ,$(ARCH))
ISA        = $(word 1,$(ARCH_SPLIT))
PLATFORM   = $(word 2,$(ARCH_SPLIT))

### Check if there is something to build
ifeq ($(flavor SRCS), undefined)
  $(error Nothing to build)
endif

### Checks end here
endif
```

这里继续解析，需要注意的是，shell pwd 是包含别的文件的目录，这里Makefile.add 包含了别的文件，因此 pwd 的结果是：`/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests`。

```Makefile
## 2. General Compilation Targets

### Create the destination directory (`build/$ARCH`)

# WORK_DIR  =/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests
WORK_DIR  = $(shell pwd)

# DST_DIR = /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/native
DST_DIR   = $(WORK_DIR)/build/$(ARCH)

# 创建目录/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/native，-p 参数确保父目录存在，如果不存在它会自动创建；若存在就忽略
$(shell mkdir -p $(DST_DIR))

### Compilation targets (a binary image or archive)

# IMAGE_REL = build/add-native
IMAGE_REL = build/$(NAME)-$(ARCH)

# IMAGE = /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/add-native
IMAGE     = $(abspath $(IMAGE_REL))

# ARCHIVE = /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/add-native.a
ARCHIVE   = $(WORK_DIR)/build/$(NAME)-$(ARCH).a

### Collect the files to be linked: object files (`.o`) and libraries (`.a`)

# OBJS = /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/native/add.o
OBJS      = $(addprefix $(DST_DIR)/, $(addsuffix .o, $(basename $(SRCS))))

# LIBS = (am klib)
LIBS     := $(sort $(LIBS) am klib) # lazy evaluation ("=") causes infinite recursions

# LINKAGE = /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/native/add.o /home/luyoung/ysyx-workbench/abstract-machine/am/build/am-native.a /home/luyoung/ysyx-workbench/abstract-machine/klib/build/klib-native.a
LINKAGE   = $(OBJS) $(addsuffix -$(ARCH).a, $(join $(addsuffix /build/, $(addprefix $(AM_HOME)/, $(LIBS))),  $(LIBS) ))

## 3. General Compilation Flags

### (Cross) compilers, e.g., mips-linux-gnu-g++

# 目前 CROSS_COMPILE 为空，因此使用默认工具链，也就是 native 机器上的工具链
AS        = $(CROSS_COMPILE)gcc
CC        = $(CROSS_COMPILE)gcc
CXX       = $(CROSS_COMPILE)g++
LD        = $(CROSS_COMPILE)ld
AR        = $(CROSS_COMPILE)ar
OBJDUMP   = $(CROSS_COMPILE)objdump
OBJCOPY   = $(CROSS_COMPILE)objcopy
READELF   = $(CROSS_COMPILE)readelf




### Compilation flags

# INC_PATH = /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include /home/luyoung/ysyx-workbench/abstract-machine/klib/include/ /home/luyoung/ysyx-workbench/abstract-machine/am/include/
INC_PATH += $(WORK_DIR)/include $(addsuffix /include/, $(addprefix $(AM_HOME)/, $(LIBS)))

# INCFLAGS = -I/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include -I/home/luyoung/ysyx-workbench/abstract-machine/klib/include/ -I/home/luyoung/ysyx-workbench/abstract-machine/am/include/
INCFLAGS += $(addprefix -I, $(INC_PATH))


# ARCH_H := arch/native.h
ARCH_H := arch/$(ARCH).h

#CFLAGS   = -O2 -MMD -Wall -Werror -I/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include -I/home/luyoung/ysyx-workbench/abstract-machine/klib/include/ -I/home/luyoung/ysyx-workbench/abstract-machine/am/include/ \
            -D__ISA__=native -D__ISA_NATIVE__ \
            -D__ARCH__=NATIVE -D__ARCH_NATIVE \
            -D__PLATFORM__ -D__PLATFORM_ \
            -DARCH_H=arch/native.h \
            -fno-asynchronous-unwind-tables -fno-builtin -fno-stack-protector \
            -Wno-main -U_FORTIFY_SOURCE -fvisibility=hidden

CFLAGS   += -O2 -MMD -Wall -Werror $(INCFLAGS) \
            -D__ISA__=\"$(ISA)\" -D__ISA_$(shell echo $(ISA) | tr a-z A-Z)__ \
            -D__ARCH__=$(ARCH) -D__ARCH_$(shell echo $(ARCH) | tr a-z A-Z | tr - _) \
            -D__PLATFORM__=$(PLATFORM) -D__PLATFORM_$(shell echo $(PLATFORM) | tr a-z A-Z | tr - _) \
            -DARCH_H=\"$(ARCH_H)\" \
            -fno-asynchronous-unwind-tables -fno-builtin -fno-stack-protector \
            -Wno-main -U_FORTIFY_SOURCE -fvisibility=hidden

# CXXFLAGS = -O2 -MMD -Wall -Werror -I/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include -I/home/luyoung/ysyx-workbench/abstract-machine/klib/include/ -I/home/luyoung/ysyx-workbench/abstract-machine/am/include/ \
            -D__ISA__=native -D__ISA_NATIVE__ \
            -D__ARCH__=NATIVE -D__ARCH_NATIVE \
            -D__PLATFORM__ -D__PLATFORM_ \
            -DARCH_H=arch/native.h \
            -fno-asynchronous-unwind-tables -fno-builtin -fno-stack-protector \
            -Wno-main -U_FORTIFY_SOURCE -fvisibility=hidden -ffreestanding -fno-rtti -fno-exceptions
CXXFLAGS +=  $(CFLAGS) -ffreestanding -fno-rtti -fno-exceptions

# ASFLAGS = -MMD -I/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include -I/home/luyoung/ysyx-workbench/abstract-machine/klib/include/ -I/home/luyoung/ysyx-workbench/abstract-machine/am/include/
ASFLAGS  += -MMD $(INCFLAGS)

# LDFLAGS = -z noexecstack
LDFLAGS  += -z noexecstack

```

```Makefile
## 4. Arch-Specific Configurations

### Paste in arch-specific configurations (e.g., from `scripts/x86_64-qemu.mk`)
# -include $(AM_HOME)/scripts/$(ARCH).mk


AM_SRCS := native/trm.c \
           native/ioe.c \
           native/cte.c \
           native/trap.S \
           native/vme.c \
           native/mpe.c \
           native/platform.c \
           native/ioe/input.c \
           native/ioe/timer.c \
           native/ioe/gpu.c \
           native/ioe/uart.c \
           native/ioe/audio.c \
           native/ioe/disk.c \

# CFLAGS 又加了些东西
# CFLAGS = -O2 -MMD -Wall -Werror -I/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include -I/home/luyoung/ysyx-workbench/abstract-machine/klib/include/ -I/home/luyoung/ysyx-workbench/abstract-machine/am/include/ \
            -D__ISA__=native -D__ISA_NATIVE__ \
            -D__ARCH__=NATIVE -D__ARCH_NATIVE \
            -D__PLATFORM__ -D__PLATFORM_ \
            -DARCH_H=arch/native.h \
            -fno-asynchronous-unwind-tables -fno-builtin -fno-stack-protector \
            -Wno-main -U_FORTIFY_SOURCE -fvisibility=hidden -fpie


CFLAGS  += -fpie

# ASFLAGS = -MMD -I/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include -I/home/luyoung/ysyx-workbench/abstract-machine/klib/include/ -I/home/luyoung/ysyx-workbench/abstract-machine/am/include/ -fpie -pie

ASFLAGS += -fpie -pie
comma = ,

# LDFLAGS_CXX = -Wl,-z noexecstack
LDFLAGS_CXX = $(addprefix -Wl$(comma), $(LDFLAGS))
```


以上的变量分析完了，接下来就开始进行目标构建。




### 3. 目标构建

```Makefile
image:
	@echo + LD "->" $(IMAGE_REL)
	@g++ -pie -o $(IMAGE) -Wl,--whole-archive $(LINKAGE) -Wl,-no-whole-archive $(LDFLAGS_CXX) -lSDL2 -ldl

run: image
	$(IMAGE)

gdb: image
	gdb -ex "handle SIGUSR1 SIGUSR2 SIGSEGV noprint nostop" $(IMAGE)

### Fall back to native gcc/binutils if there is no cross compiler
ifeq ($(wildcard $(shell which $(CC))),)
  $(info #  $(CC) not found; fall back to default gcc and binutils)
  CROSS_COMPILE :=
endif

## 5. Compilation Rules

### Rule (compile): a single `.c` -> `.o` (gcc)
$(DST_DIR)/%.o: %.c
	@mkdir -p $(dir $@) && echo + CC $<
	@$(CC) -std=gnu11 $(CFLAGS) -c -o $@ $(realpath $<)

### Rule (compile): a single `.cc` -> `.o` (g++)
$(DST_DIR)/%.o: %.cc
	@mkdir -p $(dir $@) && echo + CXX $<
	@$(CXX) -std=c++17 $(CXXFLAGS) -c -o $@ $(realpath $<)

### Rule (compile): a single `.cpp` -> `.o` (g++)
$(DST_DIR)/%.o: %.cpp
	@mkdir -p $(dir $@) && echo + CXX $<
	@$(CXX) -std=c++17 $(CXXFLAGS) -c -o $@ $(realpath $<)

### Rule (compile): a single `.S` -> `.o` (gcc, which preprocesses and calls as)
$(DST_DIR)/%.o: %.S
	@mkdir -p $(dir $@) && echo + AS $<
	@$(AS) $(ASFLAGS) -c -o $@ $(realpath $<)

### Rule (recursive make): build a dependent library (am, klib, ...)
$(LIBS): %:
	@$(MAKE) -s -C $(AM_HOME)/$* archive

### Rule (link): objects (`*.o`) and libraries (`*.a`) -> `IMAGE.elf`, the final ELF binary to be packed into image (ld)
$(IMAGE).elf: $(OBJS) $(LIBS)
	@echo + LD "->" $(IMAGE_REL).elf
	@$(LD) $(LDFLAGS) -o $(IMAGE).elf --start-group $(LINKAGE) --end-group

### Rule (archive): objects (`*.o`) -> `ARCHIVE.a` (ar)
$(ARCHIVE): $(OBJS)
	@echo + AR "->" $(shell realpath $@ --relative-to .)
	@$(AR) rcs $(ARCHIVE) $(OBJS)

### Rule (`#include` dependencies): paste in `.d` files generated by gcc on `-MMD`
#-include $(addprefix $(DST_DIR)/, $(addsuffix .d, $(basename $(SRCS))))

/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/native/tests/add.o: \
 /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/tests/add.c \
 /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include/trap.h \
 /home/luyoung/ysyx-workbench/abstract-machine/am/include/am.h \
 /home/luyoung/ysyx-workbench/abstract-machine/am/include/arch/native.h \
 /home/luyoung/ysyx-workbench/abstract-machine/am/include/amdev.h \
 /home/luyoung/ysyx-workbench/abstract-machine/klib/include/klib.h \
 /home/luyoung/ysyx-workbench/abstract-machine/klib/include/klib-macros.h

## 6. Miscellaneous

### Build order control
image: image-dep
archive: $(ARCHIVE)
image-dep: $(OBJS) $(LIBS)
	@echo \# Creating image [$(ARCH)]
.PHONY: image image-dep archive run $(LIBS)

### Clean a single project (remove `build/`)
clean:
	rm -rf Makefile.html $(WORK_DIR)/build/
.PHONY: clean

### Clean all sub-projects within depth 2 (and ignore errors)
CLEAN_ALL = $(dir $(shell find . -mindepth 2 -name Makefile))
clean-all: $(CLEAN_ALL) clean
$(CLEAN_ALL):
	-@$(MAKE) -s -C $@ clean
.PHONY: clean-all $(CLEAN_ALL)
```

构建目标 `run`：
- 它依赖于 `image`:
- - `iamge` 依赖于 `image-dep`,`image-dep`依赖于`$(OBJS)` `$(LIBS)`:
- - `$(OBJS)`是：`/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/native/add.o`，它其实就是`$(DST_DIR)/%.o`，而它又依赖于 `%.c`,显然它存在，那就接着构建目标`$(DST_DIR)/%.o`：


```Makefile
$(DST_DIR)/%.o: %.c
	@mkdir -p $(dir $@) && echo + CC $<
	@$(CC) -std=gnu11 $(CFLAGS) -c -o $@ $(realpath $<)
```

在 `Makefile` 中，这两个命令通常位于一个编译规则中，用于编译源代码文件生成对象文件（`.o` 文件）。下面是这两条命令的详细解释：

### 第一条命令：创建目录并输出正在编译的文件名

```makefile
@mkdir -p $(dir $@) && echo + CC $<
```

- `@`: 这个符号用于告诉 `make` 不要在执行命令时在控制台输出该命令本身。通常用于隐藏命令行，让输出更加干净。
- `mkdir -p $(dir $@)`: 这个命令用于创建目标文件所在的目录，确保在文件生成之前该目录存在。`$(dir $@)` 函数取得目标文件（`$@`）的目录部分。`-p` 参数确保了即使目录已经存在，命令也不会失败，并且能够创建所有必需的父目录。
- `&&`: 表示只有当 `mkdir` 命令成功执行后，才会执行后续的 `echo` 命令。
- `echo + CC $<`: 这个命令输出正在被编译的源文件名，`$<` 是 `make` 的自动变量，代表第一个依赖文件。

### 第二条命令：编译源文件生成对象文件

```makefile
@$(CC) -std=gnu11 $(CFLAGS) -c -o $@ $(realpath $<)
```

- `$(CC)`: 这是 `make` 变量，通常在 `Makefile` 开头定义，指定使用的 C 编译器，如 `gcc`。
- `-std=gnu11`: 指定编译器使用 GNU C11 标准编译源代码。这个标准是 C11 标准的一个超集，包含了 GNU 的扩展。
- `$(CFLAGS)`: 这是一个常见的 `make` 变量，包含了应用于编译过程中的编译器标志，如优化级别、警告处理等。
- `-c`: 这个编译器选项告诉编译器生成对象文件，而不是完成链接生成可执行文件。
- `-o $@`: 指定输出文件的名称，`$@` 是 `make` 的自动变量，代表当前规则的目标文件。
- `$(realpath $<)`: `realpath` 函数获取 `$<`（当前依赖项，即源文件）的绝对路径，这有助于确保编译器可以无误地找到文件，特别是在源文件位于不同目录时。

`CFLAGS`在前面已经解析过了，它是：`-O2 -MMD -Wall -Werror -I/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include -I/home/luyoung/ysyx-workbench/abstract-machine/klib/include/ -I/home/luyoung/ysyx-workbench/abstract-machine/am/include/ \
            -D__ISA__=native -D__ISA_NATIVE__ \
            -D__ARCH__=NATIVE -D__ARCH_NATIVE \
            -D__PLATFORM__ -D__PLATFORM_ \
            -DARCH_H=arch/native.h \
            -fno-asynchronous-unwind-tables -fno-builtin -fno-stack-protector \
            -Wno-main -U_FORTIFY_SOURCE -fvisibility=hidden -fpie`


也就是说，它在构建二进制目标文件，而不是可执行文件，完整的命令为：

`gcc -std=gnu11 -O2 -MMD -Wall -Werror -I/home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/include -I/home/luyoung/ysyx-workbench/abstract-machine/klib/include/ -I/home/luyoung/ysyx-workbench/abstract-machine/am/include/ \
            -D__ISA__=native -D__ISA_NATIVE__ \
            -D__ARCH__=NATIVE -D__ARCH_NATIVE \
            -D__PLATFORM__ -D__PLATFORM_ \
            -DARCH_H=arch/native.h \
            -fno-asynchronous-unwind-tables -fno-builtin -fno-stack-protector \
            -Wno-main -U_FORTIFY_SOURCE -fvisibility=hidden -fpie
-c -o /home/luyoung/ysyx-workbench/am-kernels/tests/cpu-tests/build/native/add.o add.c`

`OBJS` 被构建好了，接着构建`LIBS`:

```Makefile
$(LIBS): %:
	@$(MAKE) -s -C $(AM_HOME)/$* archive
```

这段 `Makefile` 规则定义了一个所谓的 "recursive make" 过程，它用于构建依赖的库。让我们逐步分解这个规则的功能和组成：

### 规则解释


1. **目标和模式规则**:
   - `$(LIBS): %:` 这是一个模式规则，其中 `$(LIBS)` 应该是一个变量，列出了所有需要构建的库的名字。`%` 是一个模式匹配符，表示对于 `$(LIBS)` 中的每个元素，都应用这个规则，比如：`klib`、`am`。
   - 这种使用模式的规则允许你为多个目标使用相同的构建命令，而不需要为每个目标单独写规则。

2. **命令执行**:
   - `@$(MAKE) -s -C $(AM_HOME)/$* archive` 这条命令是实际执行的操作：
     - `@`: 表示不在命令行输出这个命令，只显示其输出（如果有）。
     - `$(MAKE)`: 这通常是 `make` 工具本身，用于调用另一个 `make` 进程。
     - `-s`: 表示 "silent" 模式，不显示执行的命令，只显示命令的输出。
     - `-C $(AM_HOME)/$*`: `-C` 选项告诉 `make` 更改到指定的目录再执行 `make` 命令。
     - `$(AM_HOME)/$*` 是目标库所在的目录，其中 `$(AM_HOME)` 是一个环境变量，指向所有库的根目录，`$*` 是当前目标名称的自动变量，匹配到 `$(LIBS)` 中的每个库名。比如`$(AM_HOME)/am`, `$(AM_HOME)/klib`。
     - `archive`: 是要在指定目录下执行的 `make` 目标。这通常指一个特定的 `Makefile` 目标，用于创建静态库（.a 文件）。这里的意思就是在`$(AM_HOME)/klib`目录下用`$(AM_HOME)/klib/Makefile`构建目标`archive`。

我们来看看它是怎么构建目标`archive`的。

### 构建 klib 的目标 archive

我们可以看到，`$(AM_HOME)/klib/Makefile`中又包含了`$(AM_HOME)/Makefile`:

```Makefile
NAME = klib
SRCS = $(shell find src/ -name "*.c")
include $(AM_HOME)/Makefile

```
可以看到这里覆盖了之前的 `NAME`、`SRCS`，其它没有任何变化。换句话说，这里又是一个全新的编译目标的过程，和前面的所有编译 add.c 没有任何区别：

```Makefile
NAME = add
SRCS = tests/add.c
include /home/luyoung/ysyx-workbench/abstract-machine/Makefile
```

区别是，klib 中编译的SRCS 文件比较多，有：

```bash
$ ls /home/luyoung/ysyx-workbench/abstract-machine/klib/src
cpp.c  int64.c  stdio.c  stdlib.c  string.c
```

这里的解析过程和前面完全一致，就不赘述了，只是构建目标换成了 `archive`，而它又依赖于`$(ARCHIVE)`：

```Makefile
$(ARCHIVE): $(OBJS)
	@echo + AR "->" $(shell realpath $@ --relative-to .)
	@$(AR) rcs $(ARCHIVE) $(OBJS)
```

可以看到，`$(ARCHIVE)` 又依赖于 `$(OBJS)`。

`$(OBJS)` 为`$(addprefix $(DST_DIR)/, $(addsuffix .o, $(basename $(SRCS))))`，也就是目标文件`$(DST_DIR)/%.o`。

当然，`$(DST_DIR)/%.o`又依赖于`%.c`，满足后，继续按照编译规则进行编译：

```Makefile
### Rule (compile): a single `.c` -> `.o` (gcc)
$(DST_DIR)/%.o: %.c
	@mkdir -p $(dir $@) && echo + CC $<
	@$(CC) -std=gnu11 $(CFLAGS) -c -o $@ $(realpath $<)
```

之后，就会进行编译目标文件了。

完成后，就会编译`$(ARCHIVE)`=`$(WORK_DIR)/build/$(NAME)-$(ARCH).a`：

```Makefile
$(ARCHIVE): $(OBJS)
	@echo + AR "->" $(shell realpath $@ --relative-to .)
	@$(AR) rcs $(ARCHIVE) $(OBJS)
```

这段代码定义了一个 `Makefile` 规则，用于创建一个静态库文件，通常是 `.a` 文件。每一行的具体作用如下：

### 规则解释


1. **目标和依赖关系**:
   - `$(ARCHIVE): $(OBJS)`：这里，`$(ARCHIVE)` 是目标文件，它依赖于 `$(OBJS)`，即一组对象文件。`$(OBJS)` 变量应该包含了构建该静态库所需的所有对象文件的列表。

2. **命令解释**:
   - `@echo + AR "->" $(shell realpath $@ --relative-to .)`:
     - `@`: 表示该命令在执行时不被显示在终端，只显示其结果。
     - `echo`: 用于输出文本。
     - `+ AR "->"`: 这是一个简单的提示文本，表示正在执行的操作（AR 表示归档）。
     - `$(shell realpath $@ --relative-to .)`: 这是一个 `shell` 函数调用，它调用 `realpath` 命令来获取目标文件 `$(ARCHIVE)` 的相对路径，`$@` 是自动变量，代表当前规则的目标文件。

   - `@$(AR) rcs $(ARCHIVE) $(OBJS)`:
     - `$(AR)`: 通常是 `ar` 工具，用于创建和修改静态库。
     - `rcs`: 这是 `ar` 工具的选项，其中：
       - `r`：插入文件到归档（或替换归档中已存在的文件）。
       - `c`：创建归档文件，如果不存在的话。
       - `s`：写入索引到归档文件中，或更新已有的索引。
     - `$(ARCHIVE)`: 是静态库的文件名，目标文件。
     - `$(OBJS)`: 需要被添加到归档中的对象文件列表。

以上的过程就是为了将 klib 中所有的文件创建成静态库。


### 构建 am 的目标 archive

同样，am 也是一样的，但是 `INC_PATH` 多了一些东西，这是因为 `am` 和 `klib` 是不同的。`am` 中的 `src` 中有一些头文件，而 `klib` 中的 `src` 没有头文件要查找：

```Makefile
NAME     := am
SRCS      = $(addprefix src/, $(AM_SRCS))
INC_PATH += $(AM_HOME)/am/src

include $(AM_HOME)/Makefile

```

同样，又是相同的构建过程。就不累赘了，总之，也是创建了一个静态库`am-native.a`

### 继续编译add.c

完成下面的构建之后：

```Makefile
$(LIBS): %:
	@$(MAKE) -s -C $(AM_HOME)/$* archive
```
`$(LIBS)`就完成了构建，这时候，`image-dep`就不缺乏依赖了：
```Makefile
image-dep: $(OBJS) $(LIBS)
	@echo \# Creating image [$(ARCH)]
```

完成构建之后，`image-dep`、`image`也就不缺乏依赖了，因此会直接执行：

```Makefile
image:
	@echo + LD "->" $(IMAGE_REL)
	@g++ -pie -o $(IMAGE) -Wl,--whole-archive $(LINKAGE) -Wl,-no-whole-archive $(LDFLAGS_CXX) -lSDL2 -ldl
```

然后，就会构建 `run`：

```Makefile
run: image
	$(IMAGE)
```

还没完，接着回到这里：

```Makefile
all: $(addprefix Makefile., $(ALL))
	@echo "test list [$(words $(ALL)) item(s)]:" $(ALL)

$(ALL): %: Makefile.%



Makefile.%: tests/%.c latest
	@/bin/echo -e "NAME = $*\nSRCS = $<\ninclude $${AM_HOME}/Makefile" > $@
	@if make -s -f $@ ARCH=$(ARCH) $(MAKECMDGOALS); then \
		printf "[%14s] $(COLOR_GREEN)PASS$(COLOR_NONE)\n" $* >> $(RESULT); \
	else \
		printf "[%14s] $(COLOR_RED)***FAIL***$(COLOR_NONE)\n" $* >> $(RESULT); \
	fi
#-@rm -f Makefile.$*

run: all
	@cat $(RESULT)
	@rm $(RESULT)
```

假设我们的 `make ARCH=native ALL=add run` 构建成功了，那么就会执行：`printf "[%14s] $(COLOR_GREEN)PASS$(COLOR_NONE)\n" $* >> $(RESULT);`

然后 `Makefile.%` 构建完毕，接着构建 `all`:`@echo "test list [$(words $(ALL)) item(s)]:" $(ALL)`，这个命令执行的结果为：`test list [1 item(s)]: add`。

然后 all 就有了，然后构建 run：

```Makefile
run: all
	@cat $(RESULT)
	@rm $(RESULT)

```

至此，整个命令执行完毕：`make ARCH=native ALL=add run`。


## 三、native 构建流程图

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/native_add_test.png)


## 四、细节分析
这里会分析一下目标在构建的过程中，库是如何链接的，这里会结合相关源代码的调用、宏的控制关系，结合部分 make 输出信息来验证。

### 链接

```Makefile
image:
	@echo + LD "->" $(IMAGE_REL)
	@g++ -pie -o $(IMAGE) -Wl,--whole-archive $(LINKAGE) -Wl,-no-whole-archive $(LDFLAGS_CXX) -lSDL2 -ldl
```
、
命令展开后为：
`g++ -pie -o add-native -Wl,--whole-archive am-native.a klib-native.a -Wl,-no-whole-archive -Wl,-z noexecstack -lSDL2 -ldl`



这条命令是用来链接一个可执行文件 `add-native` 的。让我们逐步解释每个部分的含义：

- `g++`: 这是调用 C++ 编译器 `g++`。
- `-pie`: 表示生成一个位置无关可执行文件（Position Independent Executable）。
- `-o add-native`: 指定输出文件名为 `add-native`。

接下来是链接阶段的选项和库文件：

- `-Wl,--whole-archive am-native.a klib-native.a -Wl,-no-whole-archive`: 这部分使用了链接器选项 `-Wl`，它告诉编译器将 `am-native.a` 和 `klib-native.a` 中的所有目标文件（`.o` 文件）都包含进来。`--whole-archive` 表示整个库文件都被包含，而不是只包含用到的目标文件。在 `-Wl,-no-whole-archive` 之后，链接器将恢复默认行为，即只链接程序中需要的目标文件。

接下来是一些链接选项：

- `-Wl,`: 通常 `-Wl` 用于传递给链接器 `ld` 的选项。
- `-z noexecstack`: 这个选项告诉链接器禁止在可执行文件的栈上执行代码，这是一种增强安全性的措施。
- `-lSDL2`: 这里 `-l` 选项指定链接 `SDL2` 库。`-l` 选项后面跟着的是库名，编译器会在系统库路径下寻找对应的库文件。
- `-ldl`: 这个 `-l` 选项指定链接 `dl` 库，它是动态链接库加载器的库文件。

综合起来，这条命令的作用是编译并链接一个名为 `add-native` 的可执行文件，其中包含了 `am-native.a` 和 `klib-native.a` 中的所有目标文件，以及 `SDL2` 和 `dl` 这两个系统库。

整个执行过程的输出：

```bash
$ make ARCH=native ALL=add run
# Building add-run [native]
+ CC tests/add.c
# Building am-archive [native]
+ CC src/native/trm.c
+ CC src/native/ioe.c
+ CC src/native/cte.c
+ AS src/native/trap.S
+ CC src/native/vme.c
+ CC src/native/mpe.c
+ CC src/native/platform.c
+ CC src/native/ioe/input.c
+ CC src/native/ioe/timer.c
+ CC src/native/ioe/gpu.c
+ CC src/native/ioe/uart.c
+ CC src/native/ioe/audio.c
+ CC src/native/ioe/disk.c
+ AR -> build/am-native.a
# Building klib-archive [native]
+ CC src/stdlib.c
+ CC src/int64.c
+ CC src/cpp.c
+ CC src/string.c
+ CC src/stdio.c
+ AR -> build/klib-native.a
# Creating image [native]
+ LD -> build/add-native
Exit code = 00h
test list [1 item(s)]: add
[           add] PASS
```

整个过程都符合预期，它分别构建 `add.o`、 `am`、`klib`、然后链接成 `add-native`，完成构建。

### 源代码分析

#### 编译 add.c

`add.c` 包含了一个头文件 `trap.h`:

```c
#ifndef __TRAP_H__
#define __TRAP_H__

#include <am.h>
#include <klib.h>
#include <klib-macros.h>

__attribute__((noinline))
void check(bool cond) {
  if (!cond) halt(1);
}

#endif

```

`trap.h`的含义是 `add.c` 中的任何符号与这个头文件相关，编译目标文件的时候，只要能通过递归查找，找到函数声明就能通过编译，因此`add.c` 中的符号都可以使用声明在 `trap.h`中的函数或者符号。但是，`add.c`中目前确实也没啥多余的符号只有一个`check`;

#### 编译 klib

klib 中有 5 个文件需要编译：
```bash
$ ls
cpp.c  int64.c  stdio.c  stdlib.c  string.c
```

比如在 `string.c` 中，它同样包含了一些头文件，这些头文件可以给 `string.c`中的符号提供声明，比如宏、或者一些需要递归查询的头文件。

#### 编译 am

`am` 中有很多的源文件，这个和 `klib` 一样，不在累赘。

### 研究编译参数

这些编译参数用于控制 `gcc` 或 `g++` 编译器的行为。它们影响代码优化、警告、错误处理、安全性、可移植性等方面。下面是每个编译参数的详细解释：

### 通用参数 (`CFLAGS`, `CXXFLAGS`, `ASFLAGS`)
1. **`-O2`**: 启用优化级别 2，表示在不影响编译时间太多的情况下优化代码性能。常见的优化包括消除不必要的代码、循环展开、常量折叠等。

2. **`-MMD`**: 生成依赖文件（`.d` 文件），但不包括系统头文件。这用于自动追踪依赖项，帮助构建系统（如 `make`）知道在源文件改变时需要重新编译哪些文件。

3. **`-Wall`**: 启用几乎所有的常见警告，这可以帮助捕获潜在的错误或不好的编程习惯。

4. **`-Werror`**: 将所有的警告视为错误，如果有警告就终止编译。目的是保证代码严格遵循编译器的最佳实践。

5.  **`-fno-asynchronous-unwind-tables`**: 禁用异步展开表，通常用于减少生成的二进制文件的大小，因为展开表用于异常处理，而在某些环境下不需要。

6.  **`-fno-builtin`**: 禁止编译器使用内置函数。这可以防止编译器优化某些标准函数调用（如 `memcpy` 或 `printf`），强制它使用你自己实现的版本。

7.  **`-fno-stack-protector`**: 禁用堆栈保护机制。堆栈保护器用于检测和防止缓冲区溢出攻击。如果禁用它，则不会在堆栈上放置 "canary" 值。

8.  **`-Wno-main`**: 禁用关于 `main` 函数的警告。例如，如果你不使用标准的 `main` 函数声明，编译器可能会发出警告，而该选项会抑制这些警告。

9.  **`-U_FORTIFY_SOURCE`**: 取消定义 `FORTIFY_SOURCE` 宏。`FORTIFY_SOURCE` 是一个安全功能，用于检查某些函数调用（如 `memcpy` 和 `strcpy`）的安全性。禁用它可能是为了提高性能或在不需要的情况下关闭这些安全检查。

10. **`-fvisibility=hidden`**: 将符号的默认可见性设置为隐藏。通常用于限制库的导出符号，使得只有明确标记为导出的符号可用于动态链接。

### 特定编译语言的参数 (`CXXFLAGS`, `ASFLAGS`)
- **`CXXFLAGS += $(CFLAGS) -ffreestanding -fno-rtti -fno-exceptions`**:
    - `-ffreestanding`: 表示编译器不依赖标准库。适用于操作系统内核或嵌入式系统的开发。
    - `-fno-rtti`: 禁用运行时类型信息 (RTTI)，减少代码大小并提高性能。
    - `-fno-exceptions`: 禁用 C++ 异常处理，进一步减少代码大小和运行时开销。

- **`ASFLAGS  += -MMD $(INCFLAGS)`**:
    - `-MMD`: 同样生成依赖文件，这里用于汇编代码。
    - `$(INCFLAGS)`: 包含头文件搜索路径等相关标志。

### 链接器参数 (`LDFLAGS`)
1. **`-z noexecstack`**: 设置堆栈不可执行。这是一个安全措施，防止攻击者通过在堆栈上执行代码（如 shellcode）进行攻击。


### 链接规则

当我在 `klib.h` 中通过控制是否定义 `__NATIVE_USE_KLIB__`这个宏来实现条件编译，当我不编译某些函数的时候，它就会默认链接到 `glibc`；如果我编译这些函数，它被包含到 `native-klib.a` 中，然后链接，就会使用我自己实现的 `klib`。

## 五、总结
`$(AM_HOME)/Makefile` 的一些关键变量：
- `WORK_DIR`: 假设 `A` 包含`$(AM_HOME)/Makefile`，那么这个动作目录就是 `A` 所在的目录；
- `DST_DIR`: `A` 所在的目录下的`/build/$(ARCH)`；
- `IMAGE_REL`: 二进制镜像的相对目录；
- `IMAGE`: 绝对目录；
- `ARCHIVE`: 静态库，klib.a ,am.a
- `OBJS`: 源文件.o，目标文件
- `LIBS`: klib、am
- `LINKAGE`: 源文件.o、klib.a、am.a，链接文件，影响链接能否成功；
- `INC_PATH`: 符号搜索路径，影响编译是否成功。

以上就是对`make ARCH=native ALL=add run`的解析。


















































