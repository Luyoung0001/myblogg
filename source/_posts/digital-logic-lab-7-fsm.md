---
title: "数字电路实验（七）：状态机设计与仿真"
date: 2024-07-19 16:57:40
tags:
  - "digital logic"
  - "FSM"
  - "simulation"
categories:
  - "Digital Design"
---

<!--more-->

## 一、状态机
| 状态名 | y3 | y2 | y1 | y0 |
| ------ | -- | -- | -- | -- |
| A      | 0  | 0  | 0  | 0  |
| B      | 0  | 0  | 0  | 1  |
| C      | 0  | 0  | 1  | 0  |
| D      | 0  | 0  | 1  | 1  |
| E      | 0  | 1  | 0  | 0  |
| F      | 0  | 1  | 0  | 1  |
| G      | 0  | 1  | 1  | 0  |
| H      | 0  | 1  | 1  | 1  |
| I      | 1  | 0  | 0  | 0  |

这是一个状态编码，建立一个Verilog文件，用SW0作为FSM低电平有效同步复位端，用SW1作为输入w，用KEY0作为手动的时钟输入，用LEDR0作为输出z，用LEDR4-LEDR7显示4个触发器的状态，完成该状态机的具体设计。

思路就是，w 作为输出，状态寄存器不断右移，然后检查状态，更新输出：

```verilog
module fsm(e,w,clk,z,state);
input wire e;
input wire w;
input wire clk;
output reg z;
output reg[3:0] state;

always @(clk) begin
     if(e) begin
        state <= {w,state[3:1]};
        case (state)
            4'b0000:z = 1'b1;
            4'b1111:z = 1'b1;
            default:z = 1'b0;
        endcase
     end else begin
            z = 1'b0;
            state <= 4'b0000;
     end
end

endmodule
```

## 二、实验
- 七段数码管低两位显示当前按键的键码，中间两位显示对应的ASCII码（转换可以考虑自行设计一个ROM并初始化）。只需完成字符和数字键的输入，不需要实现组合键和小键盘。

- 当按键松开时，七段数码管的低四位全灭。

- 七段数码管的高两位显示按键的总次数。按住不放只算一次按键。只考虑顺序按下和放开的情况，不考虑同时按多个键的情况。

功能 2 的实现思路是，设置一个 longbuffer，每来一个键码，先右移，然后将键码装填在 longbuffer 最高位。当按键松开的时候，longbuffer 的最低位一定是 0xF0。以下是代码实现的逻辑，为了方便，这个程序没有实现键码到 ascii 码的转换：
```verilog
module ps2_keyboard(clk,resetn,ps2_clk,ps2_data,reg0,reg1,reg2,reg3,reg4,reg5);
    input clk,resetn,ps2_clk,ps2_data;

    // 次数
    output reg [7:0] reg0;
    reg [7:0] decimal;
    reg [7:0] ge;
    reg [7:0] shi;
    output reg [7:0] reg1;

    // ascii 码
    output reg [7:0] reg2;
    output reg [7:0] reg3;
    // 键码
    reg [7:0] codes;
    output reg [7:0] reg4;
    output reg [7:0] reg5;

    reg [9:0] buffer;        // ps2_data bits
    reg [3:0] count;  // count ps2_data bits
    reg [2:0] ps2_clk_sync;
    reg [7:0] ch_counter;
    reg [15:0] longbuffer;

    always @(posedge clk) begin
        ps2_clk_sync <=  {ps2_clk_sync[1:0],ps2_clk};
    end

    wire sampling = ps2_clk_sync[2] & ~ps2_clk_sync[1];

    always @(posedge clk) begin
        if (resetn == 0) begin // reset
            count <= 0;
            ch_counter <= 0;
        end
        else begin
            if (sampling) begin
                if (count == 4'd10) begin
                    if ((buffer[0] == 0) &&
                            (ps2_data)       &&
                            (^buffer[9:1])) begin // 奇校验
                        longbuffer = {buffer[8:1],longbuffer[15:8]};
                        if(buffer[8:1] == 8'b11110000) begin
                            ch_counter  <= ch_counter + 8'd1;
                        end
                        else begin

                            codes = buffer[8:1];
                            // 显示信息
                            // 次数
                            decimal = ch_counter;
                            shi = (decimal / 8'b00001010);
                            ge = decimal % 8'b00001010;
                            case (shi[3:0])
                                4'd0:
                                    reg0 = ~8'b11111101; // 0
                                4'd1:
                                    reg0 = ~8'b01100000; // 1
                                4'd2:
                                    reg0 = ~8'b11011010; // 2
                                4'd3:
                                    reg0 = ~8'b11110010; // 3
                                4'd4:
                                    reg0 = ~8'b01100110; // 4
                                4'd5:
                                    reg0 = ~8'b10110110; // 5
                                4'd6:
                                    reg0 = ~8'b10111110; // 6
                                4'd7:
                                    reg0 = ~8'b11100000; // 7
                                4'd8:
                                    reg0 = ~8'b11111110; // 8
                                4'd9:
                                    reg0 = ~8'b11100110; // 9
                                default:
                                    reg0 = ~8'b11111101;
                            endcase
                            case (ge[3:0])
                                4'd0:
                                    reg1 = ~8'b11111101; // 0
                                4'd1:
                                    reg1 = ~8'b01100000; // 1
                                4'd2:
                                    reg1 = ~8'b11011010; // 2
                                4'd3:
                                    reg1 = ~8'b11110010; // 3
                                4'd4:
                                    reg1 = ~8'b01100110; // 4
                                4'd5:
                                    reg1 = ~8'b10110110; // 5
                                4'd6:
                                    reg1 = ~8'b10111110; // 6
                                4'd7:
                                    reg1 = ~8'b11100000; // 7
                                4'd8:
                                    reg1 = ~8'b11111110; // 8
                                4'd9:
                                    reg1 = ~8'b11100110; // 9
                                default:
                                    reg1 = ~8'b11111101;
                            endcase

                            // ascii 码
                            case(codes[3:0])
                                4'b0000:
                                    reg2 = ~8'b11111101; // 0
                                4'b0001:
                                    reg2 = ~8'b01100000; // 1
                                4'b0010:
                                    reg2 = ~8'b11011010; // 2
                                4'b0011:
                                    reg2 = ~8'b11110010; // 3
                                4'b0100:
                                    reg2 = ~8'b01100110; // 4
                                4'b0101:
                                    reg2 = ~8'b10110110; // 5
                                4'b0110:
                                    reg2 = ~8'b10111110; // 6
                                4'b0111:
                                    reg2 = ~8'b11100000; // 7
                                4'b1000:
                                    reg2 = ~8'b11111110; // 8
                                4'b1001:
                                    reg2 = ~8'b11100110; // 9
                                4'b1010:
                                    reg2 = ~8'b11101110; // A
                                4'b1011:
                                    reg2 = ~8'b00111110; // b
                                4'b1100:
                                    reg2 = ~8'b10011100; // C
                                4'b1101:
                                    reg2 = ~8'b01111010; // d
                                4'b1110:
                                    reg2 = ~8'b10011110; // e
                                4'b1111:
                                    reg2 = ~8'b10001110; // f
                                default:
                                    reg2 = ~8'b10001110; // f
                            endcase
                            case(codes[7:4])
                                4'b0000:
                                    reg3 = ~8'b11111101; // 0
                                4'b0001:
                                    reg3 = ~8'b01100000; // 1
                                4'b0010:
                                    reg3 = ~8'b11011010; // 2
                                4'b0011:
                                    reg3 = ~8'b11110010; // 3
                                4'b0100:
                                    reg3 = ~8'b01100110; // 4
                                4'b0101:
                                    reg3 = ~8'b10110110; // 5
                                4'b0110:
                                    reg3 = ~8'b10111110; // 6
                                4'b0111:
                                    reg3 = ~8'b11100000; // 7
                                4'b1000:
                                    reg3 = ~8'b11111110; // 8
                                4'b1001:
                                    reg3 = ~8'b11100110; // 9
                                4'b1010:
                                    reg3 = ~8'b11101110; // A
                                4'b1011:
                                    reg3 = ~8'b00111110; // b
                                4'b1100:
                                    reg3 = ~8'b10011100; // C
                                4'b1101:
                                    reg3 = ~8'b01111010; // d
                                4'b1110:
                                    reg3 = ~8'b10011110; // e
                                4'b1111:
                                    reg3 = ~8'b10001110; // f
                                default:
                                    reg3 = ~8'b10001110; // f
                            endcase
                            // 通码
                            case(codes[3:0])
                                4'b0000:
                                    reg5 = ~8'b11111101; // 0
                                4'b0001:
                                    reg5 = ~8'b01100000; // 1
                                4'b0010:
                                    reg5 = ~8'b11011010; // 2
                                4'b0011:
                                    reg5 = ~8'b11110010; // 3
                                4'b0100:
                                    reg5 = ~8'b01100110; // 4
                                4'b0101:
                                    reg5 = ~8'b10110110; // 5
                                4'b0110:
                                    reg5 = ~8'b10111110; // 6
                                4'b0111:
                                    reg5 = ~8'b11100000; // 7
                                4'b1000:
                                    reg5 = ~8'b11111110; // 8
                                4'b1001:
                                    reg5 = ~8'b11100110; // 9
                                4'b1010:
                                    reg5 = ~8'b11101110; // A
                                4'b1011:
                                    reg5 = ~8'b00111110; // b
                                4'b1100:
                                    reg5 = ~8'b10011100; // C
                                4'b1101:
                                    reg5 = ~8'b01111010; // d
                                4'b1110:
                                    reg5 = ~8'b10011110; // e
                                4'b1111:
                                    reg5 = ~8'b10001110; // f
                                default:
                                    reg5 = ~8'b10001110; // f
                            endcase
                            case(codes[7:4])
                                4'b0000:
                                    reg4 = ~8'b11111101; // 0
                                4'b0001:
                                    reg4 = ~8'b01100000; // 1
                                4'b0010:
                                    reg4 = ~8'b11011010; // 2
                                4'b0011:
                                    reg4 = ~8'b11110010; // 3
                                4'b0100:
                                    reg4 = ~8'b01100110; // 4
                                4'b0101:
                                    reg4 = ~8'b10110110; // 5
                                4'b0110:
                                    reg4 = ~8'b10111110; // 6
                                4'b0111:
                                    reg4 = ~8'b11100000; // 7
                                4'b1000:
                                    reg4 = ~8'b11111110; // 8
                                4'b1001:
                                    reg4 = ~8'b11100110; // 9
                                4'b1010:
                                    reg4 = ~8'b11101110; // A
                                4'b1011:
                                    reg4 = ~8'b00111110; // b
                                4'b1100:
                                    reg4 = ~8'b10011100; // C
                                4'b1101:
                                    reg4 = ~8'b01111010; // d
                                4'b1110:
                                    reg4 = ~8'b10011110; // e
                                4'b1111:
                                    reg4 = ~8'b10001110; // f
                                default:
                                    reg4 = ~8'b10001110; // f
                            endcase
                            if(longbuffer[7:0] == 8'b11110000) begin
                                reg2 = 8'b11111111;
                                reg3 = 8'b11111111;
                                reg4 = 8'b11111111;
                                reg5 = 8'b11111111;

                            end
                        end
                    end
                    count <= 0;     // for next
                end
                else begin
                    buffer[count] <= ps2_data;  // store ps2_data
                    count <= count + 3'b1;
                end
            end
        end
    end
endmodule
```

### 仿真结果
![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/keyboard.png)