---
title: "Verilator 实践：编译多个 C++ 文件"
date: 2024-09-25 18:20:40
tags:
  - "Verilator"
  - "C++"
  - "simulation"
categories:
  - "Digital Design"
---

<!--more-->


## 目录结构

```dir
verilator_helpers/
│   └── include/
│       └── helpers.h
│   └── src/
│       └── helpers.cpp
│
verilatorTest/
├── Makefile
├── top.v
└── main.cpp
```




## 实验

在这个实践中，用 verilator 编译了多个文件：`helpers.cpp` `main.cpp`。

这是一个设想，没想到它是可以的，只要加一些简单的参数就好了：

main.cpp:
```cpp
// main.cpp
#include <Vtop.h>           // 生成的顶层模块头文件
#include <verilated.h>
#include <helpers.h>

void helpers(){
    help1();
    help2();
}
int main(int argc, char **argv) {
    helpers();
    Verilated::commandArgs(argc, argv);
    Vtop *top = new Vtop;

    // 初始化信号
    top->clk = 0;
    top->rst = 1;

    // 仿真循环
    for (int i = 0; i < 20; i++) {
        top->clk = !top->clk;
        if (i == 2)
            top->rst = 0;
        top->eval();
        // 输出结果
        printf("Cycle %d: counter = %d\n", i, top->counter);
    }

    delete top;
    return 0;
}

```

helpers.cpp:

```cpp
#include <helpers.h>
#include <iostream>
void help1() {
    std::cout << "help1()" << std::endl;
}
void help2() {
    std::cout << "help2()" << std::endl;
}

```

helpers.h:

```cpp
#ifndef _HELPERS_H_
#define _HELPERS_H_
void help1();
void help2();
#endif
```

Makefile:

```Makefile
hdl:
	verilator --cc top.v --exe main.cpp /home/luyoung/Test/verilator_helpers/src/helpers.cpp --CFLAGS "-I/home/luyoung/Test/verilator_helpers/include"

build:
	bear -- make -C obj_dir -f Vtop.mk

run:
	./obj_dir/Vtop

clean:
	rm -rf ./obj_dir

```

从这个 `Makefile` 中可以看到，我们只需要在 `verilator` 后面加上其它源文件的路径，然后通过 `--CFLAGS` 再加上头文件的搜索路径，就可以一起生成 `Vtop.mk`:

```bash
$ cat Vtop.mk
# Verilated -*- Makefile -*-
# DESCRIPTION: Verilator output: Makefile for building Verilated archive or executable
#
# Execute this makefile from the object directory:
#    make -f Vtop.mk

default: Vtop

### Constants...
# Perl executable (from $PERL)
PERL = perl
# Path to Verilator kit (from $VERILATOR_ROOT)
VERILATOR_ROOT = /usr/local/share/verilator
# SystemC include directory with systemc.h (from $SYSTEMC_INCLUDE)
SYSTEMC_INCLUDE ?=
# SystemC library directory with libsystemc.a (from $SYSTEMC_LIBDIR)
SYSTEMC_LIBDIR ?=

### Switches...
# C++ code coverage  0/1 (from --prof-c)
VM_PROFC = 0
# SystemC output mode?  0/1 (from --sc)
VM_SC = 0
# Legacy or SystemC output mode?  0/1 (from --sc)
VM_SP_OR_SC = $(VM_SC)
# Deprecated
VM_PCLI = 1
# Deprecated: SystemC architecture to find link library path (from $SYSTEMC_ARCH)
VM_SC_TARGET_ARCH = linux

### Vars...
# Design prefix (from --prefix)
VM_PREFIX = Vtop
# Module prefix (from --prefix)
VM_MODPREFIX = Vtop
# User CFLAGS (from -CFLAGS on Verilator command line)
VM_USER_CFLAGS = \
        -I/home/luyoung/Test/verilator_helpers/include \

# User LDLIBS (from -LDFLAGS on Verilator command line)
VM_USER_LDLIBS = \

# User .cpp files (from .cpp's on Verilator command line)
VM_USER_CLASSES = \
        helpers \
        main \

# User .cpp directories (from .cpp's on Verilator command line)
VM_USER_DIR = \
        . \
        /home/luyoung/Test/verilator_helpers/src \


### Default rules...
# Include list of all generated classes
include Vtop_classes.mk
# Include global rules
include $(VERILATOR_ROOT)/include/verilated.mk

### Executable rules... (from --exe)
VPATH += $(VM_USER_DIR)

helpers.o: /home/luyoung/Test/verilator_helpers/src/helpers.cpp
        $(OBJCACHE) $(CXX) $(CXXFLAGS) $(CPPFLAGS) $(OPT_FAST) -c -o $@ $<
main.o: main.cpp
        $(OBJCACHE) $(CXX) $(CXXFLAGS) $(CPPFLAGS) $(OPT_FAST) -c -o $@ $<

### Link rules... (from --exe)
Vtop: $(VK_USER_OBJS) $(VK_GLOBAL_OBJS) $(VM_PREFIX)__ALL.a $(VM_HIER_LIBS)
        $(LINK) $(LDFLAGS) $^ $(LOADLIBES) $(LDLIBS) $(LIBS) $(SC_LIBS) -o $@


# Verilated -*- Makefile -*-
```

可以看到，确实如此。







