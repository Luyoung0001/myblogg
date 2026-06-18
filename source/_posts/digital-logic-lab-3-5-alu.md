---
title: "数字电路实验（三点五）：ALU 设计与仿真"
date: 2024-07-16 16:57:40
tags:
  - "digital logic"
  - "ALU"
  - "simulation"
categories:
  - "Digital Design"
---

<!--more-->

## 设计 ALU

在CPU中，ALU的功能除了加减法运算之外，往往还包含逻辑运算、移位、乘除法、比较大小等等。我们这里按照RISC-V中基础指令集RV32I的ALU的设计要求来进行介绍。

RISC-V基础指令集RV32I只支持32位整型数值的操作。操作数可以是带符号补码整数或无符号数。ALU不需要完成乘除法，不需要进行溢出判断，相关操作由软件来完成。RV32I的ALU需要完成以下操作：

- 加减法操作 ：完成带符号补码数和无符号数的加减法操作，无需判断溢出和进位。此条件下，可以统一处理带符号数和无符号数。

- 逻辑运算 ：完成XOR、AND及OR操作，其他操作采用软件实现。例如，NOT可以使用与全一操作数进行XOR实现。

- 比较运算 ：完成带符号数与无符号数的全等和大小比较。此类运算均可利用减法实现。

- 全等可以用减法Zero输出判断。

- 带符号数的大小比较.

```verilog
module add(logic_tag,in1,in2,out,carray,zero,overflow);
    input [2:0]logic_tag;
    input [3:0] in1;
    input [3:0] in2;
    output reg [3:0] out;
    output reg carray;
    output reg zero;
    output reg overflow;

    reg [3:0] subs;
    reg [4:0] full_result;
    always@(logic_tag or in1 or in2) begin
        out = 4'b0000;
        subs = 4'b0000;
        case(logic_tag)
            3'b010:
                out = ~in1;
            3'b011:
                out = in1 & in2;
            3'b100:
                out = in1 | in2;
            3'b101:
                out = in1 ^ in2;
            3'b110: begin
                subs = ~in2;
                {carray,out} = in1 + subs+1'b1;
                overflow = (in1[3] == subs[3]) && (out[3] != in1[3]);
                if(overflow == 0)
                    out = (out[3] == 0? 4'b0000:4'b0001);
                else begin
                    // 如果 out 符号位为 0，说明 A 是负数，B 是正数
                    if(out[3] == 0) out = 4'b0001;
                    else out = 4'b0000;
                end
                    // 如果溢出，原始符号位一定不同，只需要比较符号位就行
                    out = (in1[3] == 0 && in2[3] == 1)?4'b0000 : 4'b0001;


            end
            3'b111:
                out = (in1 == in2) ? 4'b0001:4'b0000;

            default: begin
                subs = ({4{logic_tag[0]}}^in2);
                {carray,out} = in1 + subs +{1'b0,logic_tag};
                overflow = (in1[3] == subs[3]) && (out[3] != in1[3]);
                zero = ~(|out);
            end

        endcase

    end


endmodule
```

这里最麻烦的就是判断大小，思路是直接减，看符号位。但是减法可能会溢出，因此要在没有溢出的时候，看符号位。如果溢出了，说明输入的两个数符号位一定是不相同的，只需要看符号位就能判断出大小。

## 仿真

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/overflow.png)

经过测试关键数据，全部符合预期。