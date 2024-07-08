---
title: debugger（二）：读、写内存以及寄存器
date: 2024-06-10 20:40:27
tags:
    - c++
    - debugger
categories:
    - C++
    - debugger
---

<!--more-->

## 〇、前言
上一节，可以通过 `break 0xADDR` 的方式打断点，这种打断点的原理很简单，就是修改指令的第一个字节为 `int3`（`0xc`c）。本文将会读写内存单元、读写寄存器。

##  一、读写内存
启动 mini debugger，并打一个断点：

```bash
> ./minidbg hello
Start debugging the progress: hello, pid = 128959:
unknown SIGTRAP code 0
minidbg> break 0x5555555551d5
Set breakpoint at address 0x5555555551d5
minidbg> continue
minidbg> memory read 0x5555555551d5
4800000e60058dcc
minidbg>
```

一下为进程的maps 部分信息：
```bash
00000000000011c9 <main>:
    11c9:       f3 0f 1e fa             endbr64
    11cd:       55                      push   %rbp
    11ce:       48 89 e5                mov    %rsp,%rbp
    11d1:       48 83 ec 10             sub    $0x10,%rsp
    11d5:       48 8d 05 60 0e 00 00    lea    0xe60(%rip),%rax        # 203c <_IO_stdin_used+0x3c>
    11dc:       48 89 c6                mov    %rax,%rsi
    11df:       48 8d 05 3a 2e 00 00    lea    0x2e3a(%rip),%rax        # 4020 <_ZSt4cerr@GLIBCXX_3.4>
    11e6:       48 89 c7                mov    %rax,%rdi
    11e9:       e8 b2 fe ff ff          call   10a0 <_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@plt>
    11ee:       48 8d 05 56 0e 00 00
    ...
```

可以看到，打了一个断点之后，`0x5555555551d5` 的数据被改写了，其中 `0x48` 被修改为了 `0xcc`。我们要做的就是读写内存，并且利用写数据的方法打一个断点来检验成果。

事实上，读写内存很简单，直接可以用 `ptrace()` 系统调用来实现：

```cpp
uint64_t Debugger::read_memory(std::intptr_t address) {
  return ptrace(PTRACE_PEEKDATA, m_pid, address, nullptr);
}

void Debugger::write_memory(std::intptr_t address, uint64_t value) {
  ptrace(PTRACE_POKEDATA, m_pid, address, value);
}
```

接着尝试使用 memory write 的方式进行打断点：

```bash
./minidbg hello
Start debugging the progress: hello, pid = 129662:
unknown SIGTRAP code 0
minidbg> memo write 0x5555555551d5 0x4800000e60058dcc
minidbg> memo read 0x5555555551d5
4800000e60058dcc
minidbg> conti
Hit breakpoint at adsress 0x5555555551d5
  inline void print1() { std::cerr << "helloworld1.\n"; }
  inline void print2() { std::cerr << "helloworld2.\n"; }
  inline void print3() { std::cerr << "helloworld3.\n"; }
  inline void print4() { std::cerr << "helloworld4.\n"; }
  int main() {
>   std::cerr << "hello,world0.\n";
    std::cerr << "hello,world1.\n";
    std::cerr << "hello,world2.\n";
    std::cerr << "hello,world3.\n";
    std::cerr << "hello,world4.\n";
    for (int i = 0; i < 5; i++) {
      std::cerr << "hello,world." << i << std::endl;

minidbg>
```

可以看到非常成功，利用 `memory write` 成功打了一个“断点”。事实上，这个断点只是暂时的，它并没有被记录在 `debug` 信息系统中，不过这不重要，这只是在验证 `memory write` 的功能。

## 二、读写寄存器
这个比 memory 读写能稍微复杂一点，因为要做一些辅助性工作。首先要创建一些枚举、结构体以及一些辅助函数。

这里会用到一个库 `<sys/user.h>`，这个库中提供了` ptrace()`进行修改读写的数据结构，比如 `struct user_regs_struct`：

```cpp
struct user_regs_struct
{
  long int ebx;
  long int ecx;
  long int edx;
  long int esi;
  long int edi;
  long int ebp;
  long int eax;
  long int xds;
  long int xes;
  long int xfs;
  long int xgs;
  long int orig_eax;
  long int eip;
  long int xcs;
  long int eflags;
  long int esp;
  long int xss;
};
```

我们可以类似于这样：

```cpp
  user_regs_struct regs;
  ptrace(PTRACE_GETREGS, pid, nullptr, &regs);
```
对被 `debug` 的进程进行访问。要对寄存器进行有效快速地访问，首先得定义一些函数，这些函数分别是：从寄存器本身、寄存器编号、寄存器名称进行访问。函数原型如下：

```cpp
uint64_t get_register_value(pid_t pid, reg r);
uint64_t get_register_value_from_dwarf_register(pid_t pid, unsigned regnum);
reg get_register_from_name(const std::string &name);
```
要实现这些算法，首先得定义一些必要的数据结构：

```cpp
enum class reg {
  rax,rbx,rcx,rdx,
  rdi,rsi,rbp,rsp,
  r8,r9,r10,r11,
  r12,r13,r14,r15,
  rip,rflags,cs,orig_rax,
  fs_base,gs_base,
  fs,gs,ss,ds,es
};
constexpr std::size_t n_registers = 27;

struct reg_descriptor {
  reg r;
  int dwarf_r;
  std::string name;
};
```
以上包括一个寄存器描述符，这是进行以上访问的基本结构。接着初始化一个描述符数组，这个数组的类型为寄存器描述符，大小为 `27`：

```cpp
const std::array<reg_descriptor, n_registers> g_register_descriptors{{
    {reg::r15, 15, "r15"},
    {reg::r14, 14, "r14"},
    {reg::r13, 13, "r13"},
    {reg::r12, 12, "r12"},
    {reg::rbp, 6, "rbp"},
    {reg::rbx, 3, "rbx"},
    {reg::r11, 11, "r11"},
    {reg::r10, 10, "r10"},
    {reg::r9, 9, "r9"},
    {reg::r8, 8, "r8"},
    {reg::rax, 0, "rax"},
    {reg::rcx, 2, "rcx"},
    {reg::rdx, 1, "rdx"},
    {reg::rsi, 4, "rsi"},
    {reg::rdi, 5, "rdi"},
    {reg::orig_rax, -1, "orig_rax"},
    {reg::rip, -1, "rip"},
    {reg::cs, 51, "cs"},
    {reg::rflags, 49, "eflags"},
    {reg::rsp, 7, "rsp"},
    {reg::ss, 52, "ss"},
    {reg::fs_base, 58, "fs_base"},
    {reg::gs_base, 59, "gs_base"},
    {reg::ds, 53, "ds"},
    {reg::es, 50, "es"},
    {reg::fs, 54, "fs"},
    {reg::gs, 55, "gs"},
}};
```

然后就是访问的基本实现了：

```cpp
uint64_t get_register_value(pid_t pid, reg r) {
  user_regs_struct regs;
  ptrace(PTRACE_GETREGS, pid, nullptr, &regs);
  auto it =
      std::find_if(begin(g_register_descriptors), end(g_register_descriptors),
                   [r](auto &&rd) { return rd.r == r; });
  return *(reinterpret_cast<uint64_t *>(&regs) +
           (it - begin(g_register_descriptors)));
}

void set_register_value(pid_t pid, reg r, uint64_t value) {
  user_regs_struct regs;
  ptrace(PTRACE_GETREGS, pid, nullptr, &regs);
  auto it =
      std::find_if(begin(g_register_descriptors), end(g_register_descriptors),
                   [r](auto &&rd) { return rd.r == r; });
  *(reinterpret_cast<int64_t *>(&regs) + (it - begin(g_register_descriptors))) =
      value;
  ptrace(PTRACE_SETREGS, pid, nullptr, &regs);
}

uint64_t get_register_value_from_dwarf_register(pid_t pid, unsigned regnum) {
  auto it =
      std::find_if(begin(g_register_descriptors), end(g_register_descriptors),
                   [regnum](auto &&rd) { return rd.dwarf_r == regnum; });
  if (it == end(g_register_descriptors)) {
    throw std::out_of_range{"Unknown dwarf register"};
  }
  return get_register_value(pid, it->r);
}

std::string get_register_name(reg r) {
  auto it =
      std::find_if(begin(g_register_descriptors), end(g_register_descriptors),
                   [r](auto &&rd) { return rd.r == r; });
  return it->name;
}
reg get_register_from_name(const std::string &name) {
  auto it =
      std::find_if(begin(g_register_descriptors), end(g_register_descriptors),
                   [name](auto &&rd) { return rd.name == name; });
  return it->r;
}
```

这样就可以验证了：

```bash
./minidbg hello
Start debugging the progress: hello, pid = 130539:
unknown SIGTRAP code 0
minidbg> regi dump
...
r8 0x0000000000000000
rax 0x0000000000000000
rcx 0x0000000000000000
rdx 0x0000000000000000
rsi 0x0000000000000000
....
minidbg> regis write rax 0x1
minidbg> regi dump
...
r8 0x0000000000000000
rax 0x0000000000000001
rcx 0x0000000000000000
rdx 0x0000000000000000
...
minidbg>
```
上述，首先打印出了所有的寄存器，然后给 `rax` 中写了 `0x1`，接着 `dump`，可以看到 `rax` 成功地被修改为了 `1`。








