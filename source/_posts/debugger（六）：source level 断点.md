---
title: debugger（六）：source level 断点
date: 2024-06-11 18:37:27
tags: c++ lldb debugger
categories: C++ debugger
---

<!--more-->

## 〇、前言
前面一直使用 break 0xADDR 来打断点，这种打断点的方式及其不方便，需要手动获取某一行的断点，这一节的目标是采用以下方式打断点：

```bash
// 函数名
break main 
// 源代码
break main.cpp:10
// 地址
break 0xADDR
```

## 一、break 函数名
这个可以在所有的编译单元中搜索。对于每一个 **cu**，只需要判断：`(die.has(dwarf::DW_AT::name) && at_name(die) == name)`，如果条件成立，那么就可以获取 **line entry** 了，但是这个 **entry** 并不好用，这一行可能是函数名，我们需要跳过这个 **prologue** 或者没用的行。之后就可以在这一行对应的地址打断点了：

```cpp
void Debugger::set_breakpoint_at_function(const std::string& name) {
    for (const auto& cu : m_dwarf.compilation_units()) {
        for (const auto& die : cu.root()) {
            if (die.has(dwarf::DW_AT::name) && at_name(die) == name) {
                auto low_pc = at_low_pc(die);
                auto entry = get_line_entry_from_pc(low_pc);
                ++entry; //skip useless line
                set_breakPoint(offset_dwarf_address(entry->address));
            }
        }
    }
}
```

**line table** 大概是这个样子：

```bash
.debug_line: line number info for a single cu
Source lines (from CU-DIE at .debug_info offset 0x0000000b):

            NS new statement, BB new basic block, ET end of text sequence
            PE prologue end, EB epilogue begin
            IS=val ISA number, DI=val discriminator value
<pc>        [lno,col] NS BB ET PE EB IS= DI= uri: "filepath"
0x00001189  [   2,12] NS uri: "/home/luyoung/mydebugger/examples/hello.cpp"
0x00001191  [   3,18] NS
0x000011aa  [   4,12] NS
0x000011af  [   5, 1] NS
0x000011b1  [   5, 1] NS
0x000011c3  [   5, 1] NS
0x000011c9  [   5, 1] DI=0x1
0x000011d2  [  74,25] NS uri: "/usr/include/c++/11/iostream"
0x00001204  [   5, 1] NS uri: "/home/luyoung/mydebugger/examples/hello.cpp"
0x00001207  [   5, 1] NS
0x0000120f  [   5, 1] NS
0x00001220  [   5, 1] NS ET
```
而 `get_line_entry_from_pc(low_pc);`得到的行号是 `2`，对应于这个源代码中的是：

```cpp
#1 #include <iostream>
#2 int main() {
#3     std::cerr << "hello,world0.\n";
#4     return 0;
#5 }
```
因此，把断点打在函数名并不是一个很好的主意，我们应该打在 `#3` 行，也就是 `0x00001191` 地址处。


## 二、break source line
首先同样遍历 **cu**，每一个 **cu** 都对应一个源文件。接着就判断条件 `is_suffix(file, at_name(cu.root()))`，之后拿到 **line table,** 找到传入的行数，打断点就行了：

```cpp
void Debugger::set_breakpoint_at_source_line(const std::string& file, unsigned line) {
    for (const auto& cu : m_dwarf.compilation_units()) {
        if (is_suffix(file, at_name(cu.root()))) {
            const auto& lt = cu.get_line_table();

            for (const auto& entry : lt) {
                if (entry.is_stmt && entry.line == line) {
                    set_breakPoint(offset_dwarf_address(entry.address));
                    return;
                }
            }
        }
    }
}
```




