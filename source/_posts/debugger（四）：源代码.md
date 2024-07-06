---
title: debugger（四）：源代码
date: 2024-06-10 22:36:15
tags: c++ debugger
categories: C++ debugger
---

<!--more-->

## 〇、前言
终于来到令人激动的源代码 level 了，这里将会有一些很有意思的算法，来实现源代码级别的调试，这将会非常有趣。

## 一、使用 libelfin 库
我们不可能直接去读取整个 .debug info 段来进行设置，这是没有必要的，可以使用现成的库。首先初始化 debugger 对象：

```cpp
class debugger {
public:
    debugger (std::string prog_name, pid_t pid)
         : m_prog_name{std::move(prog_name)}, m_pid{pid} {
        auto fd = open(m_prog_name.c_str(), O_RDONLY);

        m_elf = elf::elf{elf::create_mmap_loader(fd)};
        m_dwarf = dwarf::dwarf{dwarf::elf::create_loader(m_elf)};
    }
    //...

private:
    //...
    dwarf::dwarf m_dwarf;
    elf::elf m_elf;
};
```

不必太过关注这里函数的细节，只需要关注它们做了什么。事实上，m_dwarf、m_elf 和 文件名 m_prog_name 关联起来了，然后就交给它们进行处理了。我们还需要知道 load_addr，这非常重要，因为debuf info 只会提供静态的信息，load_addr 取决于运行时，因此得想办法在 `/proc` 中获取：

```cpp
void Debugger::initialise_load_address() {
   if (m_elf.get_hdr().type == elf::et::dyn) {
      std::ifstream map("/proc/" + std::to_string(m_pid) + "/maps");

      //Read the first address from the file
      std::string addr;
      std::getline(map, addr, '-');
      m_load_address = std::stoi(addr, 0, 16);
   }
}
```

## 二、获取信息
通过一个 **pc** 怎么获取函数名呢？注意这个 **pc** 是一个 **offset addr**，传参的时候一定要转换。思路很简单，首先遍历所有的 **cu**，然后判断 **cu** 的 **low_pc** 和 **high_pc**，如果在这个 **cu** 符合，那么就通过 **cu** 拿到 **cu.root**。**cu.root** 是一个根 **die**，通过它可以遍历所有的 **die**。之后再判断 **die**的 **tag** 是不是一个函数，如果是且包含 **pc**，那么就是我们要找的函数。实现如下：

```cpp
dwarf::die Debugger::get_function_from_pc(std::intptr_t pc) {
  for (auto &cu : m_dwarf.compilation_units()) { // 循环遍历所有cu
    if (die_pc_range(cu.root()).contains(pc)) {
      for (const auto &die :
           cu.root()) { 
        if (die.tag ==
            dwarf::DW_TAG::subprogram) { 
          if (die_pc_range(die).contains(pc)) {
            return die;
          }
        }
      }
    }
  }
  throw std::out_of_range{"Cannot find function"};
}
```

接着通过 pc 来获取 line entry：

```cpp
dwarf::line_table::iterator Debugger::get_line_entry_from_pc(uint64_t pc) {
    for (auto &cu : m_dwarf.compilation_units()) {
        if (die_pc_range(cu.root()).contains(pc)) {
            auto &lt = cu.get_line_table();
            auto it = lt.find_address(pc);
            if (it == lt.end()) {
                throw std::out_of_range{"Cannot find line entry"};
            }
            else {
                return it;
            }
        }
    }

    throw std::out_of_range{"Cannot find line entry"};
}
```
接着我们打印源代码。思路是通过 debug info 中的源代码路径和 line table 来获取，好消息是，我们不必做更多的底层实现：

```cpp
void Debugger::print_source(const std::string& file_name, unsigned line, unsigned n_lines_context) {
    std::ifstream file {file_name};

    auto start_line = line <= n_lines_context ? 1 : line - n_lines_context;
    auto end_line = line + n_lines_context + (line < n_lines_context ? n_lines_context - line : 0) + 1;

    char c{};
    auto current_line = 1u;
    while (current_line != start_line && file.get(c)) {
        if (c == '\n') {
            ++current_line;
        }
    }
    std::cout << (current_line==line ? "> " : "  ");

    while (current_line <= end_line && file.get(c)) {
        std::cout << c;
        if (c == '\n') {
            ++current_line;
            std::cout << (current_line==line ? "> " : "  ");
        }
    }
    std::cout << std::endl;
}
```

## 三、测试

```bash
minidbg> break 0x555555555191
Set breakpoint at address 0x555555555191
minidbg> conti
Hit breakpoint at adsress 0x555555555191
  #include <iostream>
  int main() {
>   std::cerr << "hello,world0.\n";
    return 0;
  }
```

我们确实成功的打印出了源代码。上述基本的信息获取，基本思路就是对 DWARF 的理解，然后利用库函数接口获取我们想要的信息。

