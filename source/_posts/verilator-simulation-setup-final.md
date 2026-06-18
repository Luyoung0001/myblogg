---
title: "Verilator 仿真环境搭建：波形、自动化与收尾"
date: 2024-07-13 13:20:40
tags:
  - "Verilator"
  - "waveform"
  - "automation"
categories:
  - "Digital Design"
---

<!--more-->

## 一、前言
上一篇文章中的最后提到了应该写一个脚本来一键执行从转换、编译到运行的所有步骤，这正是这一篇文章的一个主题。

## 二、波形图
在安装了 gtkwave 之后，就可以利用--trace 参数来在运行的过程中，生成 wave.vcd，然后通过 gtkwave 查看：

```bash
gtkwave wave.vcd
```

如下图所示：
![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/wavegtk.png)

## 三、自动化
事实上，是要将很多命令放在一起，顺序执行就好了：
```bash
.PHONY: all clean

all: sim
	@echo "Write this Makefile by your self."

sim: ./csrc/main.cpp
	$(call git_commit, "sim RTL") # DO NOT REMOVE THIS LINE!!!
	verilator -Wall --cc --exe --build ./csrc/main.cpp ./vsrc/top.v --trace && \
	make -C obj_dir -f Vtop.mk Vtop && \
	./obj_dir/Vtop

clean:
	rm -rf obj_dir
	rm wave.vcd

include ../Makefile
```

然后再 Makefile 所在的目录，直接 make all 就可以了，当然 ysyx 还要求保留git追踪的命令，事实上，这个很好理解。因为 Makefile 的最后包含了上一级目录的 Makefile:
```bash
# DO NOT modify the following code!!!

TRACER = tracer-ysyx
GITFLAGS = -q --author='$(TRACER) <tracer@ysyx.org>' --no-verify --allow-empty

YSYX_HOME = $(NEMU_HOME)/..
WORK_BRANCH = $(shell git rev-parse --abbrev-ref HEAD)
WORK_INDEX = $(YSYX_HOME)/.git/index.$(WORK_BRANCH)
TRACER_BRANCH = $(TRACER)

LOCK_DIR = $(YSYX_HOME)/.git/

# prototype: git_soft_checkout(branch)
define git_soft_checkout
	git checkout --detach -q && git reset --soft $(1) -q -- && git checkout $(1) -q --
endef

# prototype: git_commit(msg)
define git_commit
	-@flock $(LOCK_DIR) $(MAKE) -C $(YSYX_HOME) .git_commit MSG='$(1)'
	-@sync $(LOCK_DIR)
endef

.git_commit:
	-@while (test -e .git/index.lock); do sleep 0.1; done;               `# wait for other git instances`
	-@git branch $(TRACER_BRANCH) -q 2>/dev/null || true                 `# create tracer branch if not existent`
	-@cp -a .git/index $(WORK_INDEX)                                     `# backup git index`
	-@$(call git_soft_checkout, $(TRACER_BRANCH))                        `# switch to tracer branch`
	-@git add . -A --ignore-errors                                       `# add files to commit`
	-@(echo "> $(MSG)" && echo $(STUID) $(STUNAME) && uname -a && uptime `# generate commit msg`) \
	                | git commit -F - $(GITFLAGS)                        `# commit changes in tracer branch`
	-@$(call git_soft_checkout, $(WORK_BRANCH))                          `# switch to work branch`
	-@mv $(WORK_INDEX) .git/index                                        `# restore git index`

.clean_index:
	rm -f $(WORK_INDEX)

_default:
	@echo "Please run 'make' under subprojects."

.PHONY: .git_commit .clean_index _default

```

这个 Makefile 通过.git_commit 完成了创建跟踪分支、切换分支、提交、在切换的过程，总之你可以看到一个 trace 分支，里面记录了每一次 make all 的记录~

我建议这里好好理解环境变量的意义、以及在哪里进行设置。说一点，大概就是假设你用 bash 进行 make，那么 make 就会使用 .bashrc 定义的环境变量。

## 四、Ubuntu 中的环境变量
详细点说，在 Ubuntu（以及其他基于 Linux 的操作系统）中，环境变量是用于存储系统范围或用户级配置和偏好的变量。它们通常用于存储如文件路径、程序配置、用户信息等数据，并可在命令行程序和脚本中被访问和使用。

### 设置环境变量

在 Ubuntu 中，可以通过多种方式设置环境变量：

1. **临时设置**：
   - 在终端会话中直接使用 `export` 命令设置变量。例如：
     ```bash
     export PATH=$PATH:/your/new/path
     ```
   - 这种设置只在当前终端会话中有效，关闭终端后失效。

2. **永久设置**：
   - 通过编辑 `~/.bashrc`、`~/.profile` 或全系统的 `/etc/profile` 和 `/etc/environment` 文件来设置环境变量。
   - `~/.bashrc` 通常用于设置特定用户的 shell 会话。
   - `~/.profile` 用于登录时加载的设置。
   - `/etc/profile` 和 `/etc/environment` 用于为所有用户设置全系统环境变量。

### `export PATH` 是环境变量

在 `.bashrc` 文件中使用 `export PATH=...` 确实是在设置环境变量。`PATH` 是一个特别重要的环境变量，它定义了 shell 搜索可执行文件的目录。通过修改 `PATH` 可以让系统知道在哪些额外的目录中查找命令和程序。

### 环境变量和 Makefile

当你在 bash 中运行 `make -j Makefile` 时，Makefile 中的变量可以访问在 `.bashrc` 中设置的环境变量。例如，如果你在 `.bashrc` 中设置了 `PATH`，那么在执行 Makefile 时，任何依赖于 `PATH` 来查找程序的命令都将使用这个设置。然而，Makefile 中定义的变量通常是局部的，仅在 Makefile 执行的上下文中有效，且不会影响 `.bashrc` 中的设置。

### 示例：在 Makefile 中使用环境变量

假设你在 `.bashrc` 中设置了一个环境变量，比如 `MY_LIB_PATH`，你可以在 Makefile 中这样使用它：

```makefile
compile:
    gcc -L${MY_LIB_PATH} -o my_program my_program.c
```

这里 `${MY_LIB_PATH}` 将被替换为在 `.bashrc` 中定义的值。

理解这些很关键。

## 五、接入 NVBoard
NVBoard 是一个用于 FPGA 仿真的开源可视化工具，它允许开发者在没有物理 FPGA 硬件的情况下可视化和测试他们的 Verilog 或 VHDL 项目。这个工具主要是为了辅助教学和研究，使得学生和研究者可以更直观地理解数字电路的设计和行为。

### 核心特点

1. **实时可视化**：
   - NVBoard 允许用户通过一个图形界面实时观察到信号的变化，这种可视化方式有助于理解和调试复杂的硬件设计。

2. **无需物理硬件**：
   - 使用 NVBoard，用户不需要购买或设置实际的 FPGA 板。这降低了学习和开发的成本，使得硬件设计和教学更加可访问。

3. **简化的测试流程**：
   - NVBoard 提供了一个平台，允许用户通过简单的配置就可以模拟各种硬件设备（如 LED、七段显示器、开关等），从而简化了测试和验证流程。

4. **开源**：
   - 作为一个开源项目，NVBoard 鼓励社区参与，用户可以自由地使用、修改和分发这个工具。开源属性也意味着它能够得到持续的改进和扩展。

### 使用场景

- **教育**：在教学数字电路和 FPGA 编程时，NVBoard 为学生提供了一个无风险、易于访问的环境，学生可以在其中实验和学习。
- **原型设计**：设计者可以在没有实际硬件的情况下，快速原型设计和测试他们的数字逻辑。
- **硬件设计验证**：在实际部署前，设计者可以使用 NVBoard 进行广泛的测试，以确保设计符合预期的功能。

### 研究 example

首先注意到 example 中的 Makefile 有这变量 NVBOARD_HOME，这意味着我们必须将这个记录在写入.bash或者.zshrc中。

接着还有 README 文件，这个文件是一个很好的参考。如果我想写一个 Verilog 代码并且 Verilator 仿真，如果像前面那样，只是利用 main.cpp 打印一些值，或者输出一个 wave.vcd看看波形，那么这显然不如直接利用硬件来仿真。example 给了一个这样的例子。

首先要构建 NVBoard ，这些构建过程在 nvboard/scripts/nvboard.mk 中，此 Makefile 依然被 example 中的 Makefile 包含。另外还有一个约束文件，这个约束文件会被脚本解析、生成一个 绑定函数（nvboard_bind_all_pins()）。具体而言：

`constr/top.nxdc` 文件被用作约束文件，这通常包含了特定的硬件设计约束信息。在硬件仿真和实际硬件设计中，这样的约束文件是至关重要的，因为它们定义了硬件信号与物理引脚之间的映射关系，以及其他可能的设计规格。

#### 1. 约束文件的角色

`top.nxdc` 文件具体用于定义：
- 模块顶层名称。
- 各个信号与具体物理引脚之间的绑定关系。
- 复杂的信号组（例如多个引脚的信号向量）的定义。

这些信息对于仿真工具（如Verilator）和硬件板卡接口工具（如NVBoard）至关重要，因为它们需要这些数据来正确地模拟硬件的行为和与实际物理设备的交互。

#### 2. Python 脚本功能

提供的Python脚本 `auto_pin_bind.py` 执行以下几个关键任务：
- **解析约束文件** (`NxdcParser`)：分析 `.nxdc` 文件来提取顶层模块名称和信号到引脚的映射。这包括处理简单的信号到单个引脚的映射和复杂的信号到引脚组的映射。
- **检查引脚有效性** (`BoardDescParser`)：确保指定的引脚在当前的硬件板卡描述中是有效的，这个描述可能存储在一个特定的文件中（如示例中的`board/N4`）。
- **生成自动绑定代码** (`AutoBindWriter`)：根据解析的数据生成C++代码，这些代码使用NVBoard的API绑定Verilog顶层信号到NVBoard模拟环境中的具体引脚。

#### 3. 自动生成的代码如何工作

生成的代码通常会包括以下几部分：
- **包含必要的头文件**：如 `nvboard.h` 和针对特定顶层模块的Verilog生成的头文件。
- **引脚绑定函数**：一个具体的函数（如 `nvboard_bind_all_pins`），在这个函数中，每个信号都会通过调用 `nvboard_bind_pin` 被绑定到一个或多个物理引脚。例如：
  ```cpp
  nvboard_bind_pin(&top->VGA_VSYNC, 1, VGA_VSYNC);
  ```
  这里，`VGA_VSYNC` 信号被绑定到名为 `VGA_VSYNC` 的物理引脚。

这种自动化的脚本处理简化了从Verilog设计到实际硬件部署的过程，特别是在进行硬件仿真和验证时，能够快速适应设计的变化和迭代。

之后，Verilator 将 Verilog 代码、NVBoard等相关的代码一起处理、编译、链接，生成一个可执行二进制文件。这样就从 Verolog 接入到了 NVBoard。

### 理解 RTL 仿真的行为

在使用Verilator进行Verilog代码的仿真时，了解其如何将Verilog转换为C++代码，并模拟时序逻辑电路的行为非常重要。以下是一步步指导来理解这一过程：

### 1. **从Verilog到C++代码的转换**

Verilator 是一个将Verilog代码转换为可编译的C++/SystemC代码的工具，它主要用于硬件仿真。Verilator优化了速度和效率，特别擅长处理大规模设计。其主要步骤包括：

- **解析**：解析Verilog代码，理解模块间的连接、信号、参数等。
- **Elaboration（展开）**：在这一阶段，Verilator会展开所有的实例，解析生成的层次结构。
- **优化**：优化步骤包括删除未使用的信号、简化逻辑表达式等，以提高生成代码的效率。
- **代码生成**：最后，Verilator将生成的内部表示转换为C++代码。这些代码将模拟Verilog代码的逻辑行为。

### 2. **时序逻辑的仿真**

Verilog 中的时序逻辑通常依赖于寄存器和时钟信号。在Verilator中，这些时序逻辑被转换为C++中的类和函数，来模拟硬件的行为。关键点如下：

- **时钟信号**：Verilator会识别Verilog代码中的时钟信号，并在C++代码中为每个时钟边沿生成相应的逻辑。例如，对于每一个`posedge`和`negedge`，Verilator会生成不同的代码块来处理这些事件。
- **寄存器更新**：在Verilog中，寄存器的值通常在时钟边缘更新。Verilator会在C++代码中重现这一行为，通常是在模拟时钟的变化时，通过函数调用来更新状态。
- **组合逻辑与时序逻辑的分离**：Verilator尽量将组合逻辑和时序逻辑分开处理。组合逻辑在每次仿真步骤中根据输入信号即时计算，而时序逻辑则在适当的时钟事件发生时更新。

### 3. **仿真流程**

在通过Verilator生成的C++仿真环境中，仿真流程通常如下：

- **初始化**：设置初始状态，包括所有信号和寄存器的初始值。
- **仿真循环**：进入一个循环，每次循环模拟一个或多个时钟周期。在每个时钟周期中，Verilator生成的代码会先计算所有的组合逻辑，然后在时钟边缘更新所有寄存器。
- **观察和调试**：仿真过程中可以观察变量的值，用于调试或验证逻辑是否按预期工作。

### 4. **示例理解**

假设你有以下Verilog代码：

```verilog
module example (
    input clk,
    input reset,
    input [7:0] input_data,
    output reg [7:0] output_data
);

always @(posedge clk or posedge reset) begin
    if (reset)
        output_data <= 8'b0;
    else
        output_data <= input_data;
end

endmodule
```

在这个例子中，`output_data` 寄存器的值依赖于时钟`clk`的上升沿和复位信号`reset`。在Verilator生成的C++代码中，你会看到对应于这段代码的类和方法，这些方法在模拟`clk`的上升沿时被调用，根据`input_data`和`reset`的值更新`output_data`。

对于 example 中的 C++ 代码：
```cpp
#include <nvboard.h>
#include "../build/obj_dir/Vtop.h"

static Vtop dut;

void nvboard_bind_all_pins(Vtop* top);

static void single_cycle() {
  dut.clk = 0; dut.eval();
  dut.clk = 1; dut.eval();
}

static void reset(int n) {
  dut.rst = 1;
  while (n -- > 0) single_cycle();
  dut.rst = 0;
}

int main() {
  nvboard_bind_all_pins(&dut);
  nvboard_init();

  reset(10);

  while(1) {
    nvboard_update();
    single_cycle();
  }
}

```

这个代码的关键是:
```cpp
static void single_cycle() {
  dut.clk = 0; dut.eval();
  dut.clk = 1; dut.eval();
}
```

eval() 调用了Vtop::eval_step()，而Vtop::eval_step() 调用了Vtop___024root___eval(&(vlSymsp->TOP))，Vtop___024root___eval(&(vlSymsp->TOP)) 函数则调用了 Verilog 生成的 C++代码中的关于 Verilog 的逻辑处理的过程。

## 六、流水灯
理解了上面的大致过程，那么流水灯就很简单了。

首先写流水灯的逻辑：

```verilog
module led(
  input clk,
  input rst,
  input [4:0] btn,
  input [7:0] sw,
  output [15:0] ledr
);
  reg [31:0] count;
  reg [15:0] led;
  always @(posedge clk) begin
    if (rst) begin led <= 1; count <= 0; end
    else begin
      if (count == 0) led <= {led[14:0], led[15]};
      count <= (count >= 5000000 ? 32'b0 : count + 1);
    end
  end

  assign ledr = led;
endmodule

```
```verilog
module top(
    input clk,
    input rst,
    input [4:0] btn,
    input [7:0] sw,
    input ps2_clk,
    input ps2_data,
    input uart_rx,
    output uart_tx,
    output [15:0] ledr,
    output VGA_CLK,
    output VGA_HSYNC,
    output VGA_VSYNC,
    output VGA_BLANK_N,
    output [7:0] VGA_R,
    output [7:0] VGA_G,
    output [7:0] VGA_B,
    output [7:0] seg0,
    output [7:0] seg1,
    output [7:0] seg2,
    output [7:0] seg3,
    output [7:0] seg4,
    output [7:0] seg5,
    output [7:0] seg6,
    output [7:0] seg7
);

led my_led(
    .clk(clk),
    .rst(rst),
    .btn(btn),
    .sw(sw),
    .ledr(ledr)
);

assign VGA_CLK = clk;

wire [9:0] h_addr;
wire [9:0] v_addr;
wire [23:0] vga_data;



vmem my_vmem(
    .h_addr(h_addr),
    .v_addr(v_addr[8:0]),
    .vga_data(vga_data)
);



endmodule

module vmem(
    input [9:0] h_addr,
    input [8:0] v_addr,
    output [23:0] vga_data
);

reg [23:0] vga_mem [524287:0];

initial begin
    $readmemh("resource/picture.hex", vga_mem);
end

assign vga_data = vga_mem[{h_addr, v_addr}];

endmodule

```

然后将约束文件、Makefile 移植过去，就可以跑了。







