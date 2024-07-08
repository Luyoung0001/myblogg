---
title: debugger（七）：栈帧（backtrace）
date: 2024-06-11 20:37:04
tags:
    - c++
    - debugger
categories:
    - 系统编程
    - C++
    - debugger
---

<!--more-->

## 〇、前言
在前面已经详细得介绍了栈帧，这里实现 **backtrace**。

## 一、backtrace
思路是遍历 **stack**，搜索 **stack pointer**，逐个打印栈帧信息，一直打印到 **main** 函数。

```cpp
void Debugger::print_backtrace() {
    auto output_frame = [frame_number = 0] (auto&& func) mutable {
        std::cout << "frame #" << frame_number++ << ": 0x" << dwarf::at_low_pc(func)
                  << ' ' << dwarf::at_name(func) << std::endl;
    };

    auto current_func = get_function_from_pc(get_pc());
    output_frame(current_func);

    auto frame_pointer = get_register_value(m_pid, reg::rbp);
    auto return_address = read_memory(frame_pointer+8);

    while (dwarf::at_name(current_func) != "main") {
        current_func = get_function_from_pc(return_address);
        output_frame(current_func);
        frame_pointer = read_memory(frame_pointer);
        return_address = read_memory(frame_pointer+8);
     }
}
```

这样就好了，测试一下：

```bash
> ./minidbg stack
Start debugging the progress: stack, pid = 147785:
unknown SIGTRAP code 0
minidbg> b a
Set breakpoint at address 0x555555555131
minidbg> continue
Hit breakpoint at adsress 0x555555555131
  void a() {
>     int foo = 1;
      int foo1 = 1;
      int foo2 = 1;
      int foo3 = 1;
  }

  void b() {
      int foo = 1;
      int foo1 = 1;
      a();

minidbg> backtrace
frame #0: 0x1129 a
frame #1: 0x1150 b
frame #2: 0x1180 c
frame #3: 0x11b0 d
frame #4: 0x11e0 e
frame #5: 0x1210 f
frame #6: 0x1240 main
```
**backtrace** 符合预期。

通过读取帧指针（frame pointer）和返回地址来遍历整个调用栈，直到达到main函数为止。每次循环都会输出当前函数的栈帧信息，并更新帧指针和返回地址以跳转到下一个栈帧。


