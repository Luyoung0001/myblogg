---
title: "数字电路实验（一）：基础门级电路实践"
date: 2024-07-13 16:57:40
tags:
  - "digital logic"
  - "ysyx"
  - "lab"
categories:
  - "Digital Design"
---

<!--more-->

## 实验过程

本实验采用行为建模，top.v 代码如下：
```verilog
module c41(X0,X1,X2,X3,Y,F);
input [1:0] Y;
input [1:0] X0;
input [1:0] X1;
input [1:0] X2;
input [1:0] X3;

output reg [1:0] F;
always @ (Y or X0 or X1 or X2 or X3)
    case(Y)
        0: F = X0;
        1: F = X1;
        2: F = X2;
        3: F = X3;
        default: F = 2'b00;
    endcase
endmodule

module top(
    input clk,
    input rst,
    input [4:0] btn,
    input [9:0] sw,
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

c41 my_41(
    .Y(sw[1:0]),
    .X0(sw[3:2]),
    .X1(sw[5:4]),
    .X2(sw[7:6]),
    .X3(sw[9:8]),
    .F(ledr[1:0])
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

这里需要注意的是，原来的 example 中的 top.v 代码的按钮只有 8 个，因此得修改一下。并且还得修改 绑定函数：
```cpp
void nvboard_bind_all_pins(Vtop* top) {
	...
	nvboard_bind_pin( &top->sw, 10, SW9, SW8, SW7, SW6, SW5, SW4, SW3, SW2, SW1, SW0);
	...
}

```

之后就可以愉快地按按钮了：
![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/buttons.png)

以上就是这个实验的全部内容了~

