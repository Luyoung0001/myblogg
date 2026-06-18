---
title: "调试器实现（一）：断点机制与案例分析"
date: 2024-05-25 23:32:01
tags:
  - "debugger"
  - "breakpoint"
  - "ptrace"
categories:
  - "Systems Programming"
---

<!--more-->

## 〇、前言
最近在学习 **debugger** 的实现原理，并按照博客实现，是一个很不错的小项目，这是[地址](https://blog.tartanllama.xyz/writing-a-linux-debugger-breakpoints/)。由于 **macOS** 的问题，系统调用并不完全相同，因此实现了两个版本分支，一个是 **main** 版本分支（macOS M1 silicon），另一个是 **linux** 版本分支（Ubuntu 20.04 x86），这是[仓库地址](https://github.com/Luyoung0001/mydebugger)。以下以及后都用 **linux** 版本代码阐述其原理。

## 一、断点创建
这很简单，主要是由 `ptrace()` 实现（**debug**工具都依赖于 `ptrace()` ）：

```cpp
#ifndef BREAKPOINT_HPP_
#define BREAKPOINT_HPP_
#include <stdint.h>
#include <sys/ptrace.h>
#include <sys/wait.h>
class BreakPoint {
  pid_t m_pid;
  intptr_t m_addr;
  bool m_enabled;
  uint8_t m_saved_data; // 最低位的旧数据（1 字节）,之后需要恢复

public:
  BreakPoint() {}
  BreakPoint(pid_t pid, intptr_t addr)
      : m_pid(pid), m_addr(addr), m_enabled(false), m_saved_data{} {}
  auto is_enabled() const -> bool { return m_enabled; }
  auto get_address() const -> intptr_t { return m_addr; }

  void enable() {
    auto data = ptrace(PTRACE_PEEKDATA, m_pid, m_addr, nullptr);
    m_saved_data = static_cast<uint8_t>(data & 0xff); // save bottom byte

    uint64_t int3 = 0xcc;
    uint64_t data_with_int3 = ((data & ~0xff) | int3); // set bottom byte to
                                                       // 0xcc
    ptrace(PTRACE_POKEDATA, m_pid, m_addr, data_with_int3);

    m_enabled = true;
  }
  void disable() {
    auto data = ptrace(PTRACE_PEEKDATA, m_pid, m_addr, nullptr);
    auto restored_data = ((data & ~0xff) | m_saved_data);
    ptrace(PTRACE_POKEDATA, m_pid, m_addr, restored_data);

    m_enabled = false;
  }
};
#endif

```

以上是 **BreakPoint** 类的定义。重点是关注 `enable()` 和 `disable()` 两个方法，在这两个方法中，这段代码及其关键：

```cpp
	auto data = ptrace(PTRACE_PEEKDATA, m_pid, m_addr, nullptr);
    m_saved_data = static_cast<uint8_t>(data & 0xff); // save bottom byte

    uint64_t int3 = 0xcc;
    uint64_t data_with_int3 = ((data & ~0xff) | int3); // set bottom byte to
                                                       // 0xcc
    ptrace(PTRACE_POKEDATA, m_pid, m_addr, data_with_int3);
```

这里先说明一下，`int3` 是 `x86` 中的一个中断指令，只要我们把某个指令修改为 `int3`，那么它运行到这里就会停下来。另外，我们只是打个断点，又不想真正得越过这个指令（这个指令被越过不执行，谁都不知道会发生什么），所以后面得恢复这个执行，并重新执行它，这就是 `disable()`，我们先讨论 `enable()`。

因为 **int3** 指令的代码为 `0xcc`，这很明显是一个 1 字节指令，只要我们在我们想打断的指令处，将操作码改为 `0xcc`，这个指令就会停下来（这里牵扯到字节序，因为指令第一个字节是低地址，因为我们需要将 **int3** 放在一个指令的最低处）。然后再将这个被篡改的指令放回到原处，就成功的打了一个断点。

至于 `disable()`，其实做的也是这样的事情，将原来的被替换的一个字节再恢复放回去：

```cpp
	auto data = ptrace(PTRACE_PEEKDATA, m_pid, m_addr, nullptr);
    auto restored_data = ((data & ~0xff) | m_saved_data);
    ptrace(PTRACE_POKEDATA, m_pid, m_addr, restored_data);

    m_enabled = false;
```

以上都是很简单的东西，我们现在就可以检验这个事情了，对了以下是 debugger 类的定义：

```cpp
#ifndef DEBUGGER_HPP_
#define DEBUGGER_HPP_

#include "../ext/linenoise/linenoise.h"
#include "breakpoint.hpp"
#include "helpers.hpp"
#include <cstddef>
#include <iostream>
#include <string>
#include <unordered_map>

class debugger {
  std::string m_prog_name;
  pid_t m_pid;
  std::unordered_map<std::intptr_t, BreakPoint> m_breakPoints; // 存储断点

public:
  // 这里不应该给默认参数，断言：传了正确的 prog_name，pid
  debugger(std::string prog_name, pid_t pid)
      : m_prog_name(prog_name), m_pid(pid) {}
  void run() {
    int wait_status;
    auto options = 0;
    waitpid(m_pid, &wait_status, options);
    char *line = nullptr;
    while ((line = linenoise("minidbg> ")) != nullptr) {
      handl_command(line);
      linenoiseHistoryAdd(line);
      linenoiseFree(line);
    }
  }
  // handlers
  void handl_command(const std::string &line) {
    auto args = split(line, ' ');
    auto command = args[0];
    if (is_prefix(command, "continue")) {
      continue_execution();

    } else if (is_prefix(command, "break")) { // break 地址
      std::string addr{args[1], 2};
      set_breakPoint(std::stol(addr, 0, 16));

    } else {
      std::cerr << "Unkown command\n";
    }
  }
  void continue_execution() {
    ptrace(PTRACE_CONT, m_pid, nullptr, nullptr);

    int wait_status;
    auto options = 0;
    waitpid(m_pid, &wait_status, options);
  }

  void set_breakPoint(std::intptr_t addr) {
    std::cout << "Set breakpoint at address 0x" << std::hex << addr
              << std::endl;
    BreakPoint bp{m_pid, addr};
    bp.enable();
    m_breakPoints[addr] = bp;
  }

  ~debugger() {}
};

#endif
```


## 二、检测
`main()` 就是 **debugger** 的 `main()` 了：

```cpp
#include "../include/debugger.hpp"
#include <cstddef>
#include <iostream>
#include <unistd.h>
#include <sys/personality.h>
int main(int argc, char *argv[]) {
  if (argc < 2) {
    std::cerr << "Program paras are not right.";
    return -1;
  }
  auto proj = argv[1];
  auto pid = fork();
  if (pid == 0) {
    personality(ADDR_NO_RANDOMIZE); // 取消随机内存
    // child progress
    // debugged progress
    ptrace(PTRACE_TRACEME, 0, nullptr, nullptr);
    execl(proj, proj, nullptr);
  } else if (pid >= 1) {
    // parent progress
    // debugger progress

    std::cout << "Start debugging the progress: " << proj << ", pid = " << pid
              << ":\n";
    debugger dbg(proj, pid);
    dbg.run();
  }

  return 0;
}
```
`被 debug` 的进程放在子进程中，然后由父进程，也就是我们的 `debugger process`，由它进行调试。

我们先写一个被 **debug** 的程序，这个程序输出 `hello,world.`：

```cpp
#include <iostream>
int main() {
  std::cerr << "hello,world.\n";
  return 0;
}
```
编译后，我们要打断点进行测试，可以看到目前只能传入一个地址，这个地址还是 `0x` 开头的 16 进制地址，我们对于这个地址丝毫没有头绪，因为我们不知道  `std::cerr << "hello,world.\n";`这个语句对应的汇编代码的指令地址是什么。这个程序首先有一个程序结构，对这个不清楚的话，可以看看我之前写的文章，是关于 `elf` 的，可以参考 c++ 内存模型或者 c++内存管理的那几篇博客。

现在看看这个被调试的程序的文件结构：

```bash
objdump -d hw

hw:     file format elf64-x86-64


Disassembly of section .init:

0000000000001000 <_init>:
    1000:       f3 0f 1e fa             endbr64
    1004:       48 83 ec 08             sub    $0x8,%rsp
    1008:       48 8b 05 d9 2f 00 00    mov    0x2fd9(%rip),%rax        # 3fe8 <__gmon_start__>
    100f:       48 85 c0                test   %rax,%rax
    1012:       74 02                   je     1016 <_init+0x16>
    1014:       ff d0                   callq  *%rax
    1016:       48 83 c4 08             add    $0x8,%rsp
    101a:       c3                      retq

Disassembly of section .plt:

0000000000001020 <.plt>:
    1020:       ff 35 82 2f 00 00       pushq  0x2f82(%rip)        # 3fa8 <_GLOBAL_OFFSET_TABLE_+0x8>
    1026:       f2 ff 25 83 2f 00 00    bnd jmpq *0x2f83(%rip)        # 3fb0 <_GLOBAL_OFFSET_TABLE_+0x10>
    102d:       0f 1f 00                nopl   (%rax)
    1030:       f3 0f 1e fa             endbr64
    1034:       68 00 00 00 00          pushq  $0x0
    1039:       f2 e9 e1 ff ff ff       bnd jmpq 1020 <.plt>
    103f:       90                      nop
    1040:       f3 0f 1e fa             endbr64
    1044:       68 01 00 00 00          pushq  $0x1
    1049:       f2 e9 d1 ff ff ff       bnd jmpq 1020 <.plt>
    104f:       90                      nop
    1050:       f3 0f 1e fa             endbr64
    1054:       68 02 00 00 00          pushq  $0x2
    1059:       f2 e9 c1 ff ff ff       bnd jmpq 1020 <.plt>
    105f:       90                      nop

Disassembly of section .plt.got:

0000000000001060 <__cxa_finalize@plt>:
    1060:       f3 0f 1e fa             endbr64
    1064:       f2 ff 25 65 2f 00 00    bnd jmpq *0x2f65(%rip)        # 3fd0 <__cxa_finalize@GLIBC_2.2.5>
    106b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)

Disassembly of section .plt.sec:

0000000000001070 <__cxa_atexit@plt>:
    1070:       f3 0f 1e fa             endbr64
    1074:       f2 ff 25 3d 2f 00 00    bnd jmpq *0x2f3d(%rip)        # 3fb8 <__cxa_atexit@GLIBC_2.2.5>
    107b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)

0000000000001080 <_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@plt>:
    1080:       f3 0f 1e fa             endbr64
    1084:       f2 ff 25 35 2f 00 00    bnd jmpq *0x2f35(%rip)        # 3fc0 <_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@GLIBCXX_3.4>
    108b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)

0000000000001090 <_ZNSt8ios_base4InitC1Ev@plt>:
    1090:       f3 0f 1e fa             endbr64
    1094:       f2 ff 25 2d 2f 00 00    bnd jmpq *0x2f2d(%rip)        # 3fc8 <_ZNSt8ios_base4InitC1Ev@GLIBCXX_3.4>
    109b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)

Disassembly of section .text:

00000000000010a0 <_start>:
    10a0:       f3 0f 1e fa             endbr64
    10a4:       31 ed                   xor    %ebp,%ebp
    10a6:       49 89 d1                mov    %rdx,%r9
    10a9:       5e                      pop    %rsi
    10aa:       48 89 e2                mov    %rsp,%rdx
    10ad:       48 83 e4 f0             and    $0xfffffffffffffff0,%rsp
    10b1:       50                      push   %rax
    10b2:       54                      push   %rsp
    10b3:       4c 8d 05 d6 01 00 00    lea    0x1d6(%rip),%r8        # 1290 <__libc_csu_fini>
    10ba:       48 8d 0d 5f 01 00 00    lea    0x15f(%rip),%rcx        # 1220 <__libc_csu_init>
    10c1:       48 8d 3d c1 00 00 00    lea    0xc1(%rip),%rdi        # 1189 <main>
    10c8:       ff 15 12 2f 00 00       callq  *0x2f12(%rip)        # 3fe0 <__libc_start_main@GLIBC_2.2.5>
    10ce:       f4                      hlt
    10cf:       90                      nop

00000000000010d0 <deregister_tm_clones>:
    10d0:       48 8d 3d 39 2f 00 00    lea    0x2f39(%rip),%rdi        # 4010 <__TMC_END__>
    10d7:       48 8d 05 32 2f 00 00    lea    0x2f32(%rip),%rax        # 4010 <__TMC_END__>
    10de:       48 39 f8                cmp    %rdi,%rax
    10e1:       74 15                   je     10f8 <deregister_tm_clones+0x28>
    10e3:       48 8b 05 ee 2e 00 00    mov    0x2eee(%rip),%rax        # 3fd8 <_ITM_deregisterTMCloneTable>
    10ea:       48 85 c0                test   %rax,%rax
    10ed:       74 09                   je     10f8 <deregister_tm_clones+0x28>
    10ef:       ff e0                   jmpq   *%rax
    10f1:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)
    10f8:       c3                      retq
    10f9:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)

0000000000001100 <register_tm_clones>:
    1100:       48 8d 3d 09 2f 00 00    lea    0x2f09(%rip),%rdi        # 4010 <__TMC_END__>
    1107:       48 8d 35 02 2f 00 00    lea    0x2f02(%rip),%rsi        # 4010 <__TMC_END__>
    110e:       48 29 fe                sub    %rdi,%rsi
    1111:       48 89 f0                mov    %rsi,%rax
    1114:       48 c1 ee 3f             shr    $0x3f,%rsi
    1118:       48 c1 f8 03             sar    $0x3,%rax
    111c:       48 01 c6                add    %rax,%rsi
    111f:       48 d1 fe                sar    %rsi
    1122:       74 14                   je     1138 <register_tm_clones+0x38>
    1124:       48 8b 05 c5 2e 00 00    mov    0x2ec5(%rip),%rax        # 3ff0 <_ITM_registerTMCloneTable>
    112b:       48 85 c0                test   %rax,%rax
    112e:       74 08                   je     1138 <register_tm_clones+0x38>
    1130:       ff e0                   jmpq   *%rax
    1132:       66 0f 1f 44 00 00       nopw   0x0(%rax,%rax,1)
    1138:       c3                      retq
    1139:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)

0000000000001140 <__do_global_dtors_aux>:
    1140:       f3 0f 1e fa             endbr64
    1144:       80 3d e5 2f 00 00 00    cmpb   $0x0,0x2fe5(%rip)        # 4130 <completed.0>
    114b:       75 2b                   jne    1178 <__do_global_dtors_aux+0x38>
    114d:       55                      push   %rbp
    114e:       48 83 3d 7a 2e 00 00    cmpq   $0x0,0x2e7a(%rip)        # 3fd0 <__cxa_finalize@GLIBC_2.2.5>
    1155:       00
    1156:       48 89 e5                mov    %rsp,%rbp
    1159:       74 0c                   je     1167 <__do_global_dtors_aux+0x27>
    115b:       48 8b 3d a6 2e 00 00    mov    0x2ea6(%rip),%rdi        # 4008 <__dso_handle>
    1162:       e8 f9 fe ff ff          callq  1060 <__cxa_finalize@plt>
    1167:       e8 64 ff ff ff          callq  10d0 <deregister_tm_clones>
    116c:       c6 05 bd 2f 00 00 01    movb   $0x1,0x2fbd(%rip)        # 4130 <completed.0>
    1173:       5d                      pop    %rbp
    1174:       c3                      retq
    1175:       0f 1f 00                nopl   (%rax)
    1178:       c3                      retq
    1179:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)

0000000000001180 <frame_dummy>:
    1180:       f3 0f 1e fa             endbr64
    1184:       e9 77 ff ff ff          jmpq   1100 <register_tm_clones>

0000000000001189 <main>:
    1189:       f3 0f 1e fa             endbr64
    118d:       55                      push   %rbp
    118e:       48 89 e5                mov    %rsp,%rbp
    1191:       48 8d 35 6d 0e 00 00    lea    0xe6d(%rip),%rsi        # 2005 <_ZStL19piecewise_construct+0x1>
    1198:       48 8d 3d 81 2e 00 00    lea    0x2e81(%rip),%rdi        # 4020 <_ZSt4cerr@@GLIBCXX_3.4>
    119f:       e8 dc fe ff ff          callq  1080 <_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@plt>
    11a4:       b8 00 00 00 00          mov    $0x0,%eax
    11a9:       5d                      pop    %rbp
    11aa:       c3                      retq

00000000000011ab <_Z41__static_initialization_and_destruction_0ii>:
    11ab:       f3 0f 1e fa             endbr64
    11af:       55                      push   %rbp
    11b0:       48 89 e5                mov    %rsp,%rbp
    11b3:       48 83 ec 10             sub    $0x10,%rsp
    11b7:       89 7d fc                mov    %edi,-0x4(%rbp)
    11ba:       89 75 f8                mov    %esi,-0x8(%rbp)
    11bd:       83 7d fc 01             cmpl   $0x1,-0x4(%rbp)
    11c1:       75 32                   jne    11f5 <_Z41__static_initialization_and_destruction_0ii+0x4a>
    11c3:       81 7d f8 ff ff 00 00    cmpl   $0xffff,-0x8(%rbp)
    11ca:       75 29                   jne    11f5 <_Z41__static_initialization_and_destruction_0ii+0x4a>
    11cc:       48 8d 3d 5e 2f 00 00    lea    0x2f5e(%rip),%rdi        # 4131 <_ZStL8__ioinit>
    11d3:       e8 b8 fe ff ff          callq  1090 <_ZNSt8ios_base4InitC1Ev@plt>
    11d8:       48 8d 15 29 2e 00 00    lea    0x2e29(%rip),%rdx        # 4008 <__dso_handle>
    11df:       48 8d 35 4b 2f 00 00    lea    0x2f4b(%rip),%rsi        # 4131 <_ZStL8__ioinit>
    11e6:       48 8b 05 0b 2e 00 00    mov    0x2e0b(%rip),%rax        # 3ff8 <_ZNSt8ios_base4InitD1Ev@GLIBCXX_3.4>
    11ed:       48 89 c7                mov    %rax,%rdi
    11f0:       e8 7b fe ff ff          callq  1070 <__cxa_atexit@plt>
    11f5:       90                      nop
    11f6:       c9                      leaveq
    11f7:       c3                      retq

00000000000011f8 <_GLOBAL__sub_I_main>:
    11f8:       f3 0f 1e fa             endbr64
    11fc:       55                      push   %rbp
    11fd:       48 89 e5                mov    %rsp,%rbp
    1200:       be ff ff 00 00          mov    $0xffff,%esi
    1205:       bf 01 00 00 00          mov    $0x1,%edi
    120a:       e8 9c ff ff ff          callq  11ab <_Z41__static_initialization_and_destruction_0ii>
    120f:       5d                      pop    %rbp
    1210:       c3                      retq
    1211:       66 2e 0f 1f 84 00 00    nopw   %cs:0x0(%rax,%rax,1)
    1218:       00 00 00
    121b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)

0000000000001220 <__libc_csu_init>:
    1220:       f3 0f 1e fa             endbr64
    1224:       41 57                   push   %r15
    1226:       4c 8d 3d 5b 2b 00 00    lea    0x2b5b(%rip),%r15        # 3d88 <__frame_dummy_init_array_entry>
    122d:       41 56                   push   %r14
    122f:       49 89 d6                mov    %rdx,%r14
    1232:       41 55                   push   %r13
    1234:       49 89 f5                mov    %rsi,%r13
    1237:       41 54                   push   %r12
    1239:       41 89 fc                mov    %edi,%r12d
    123c:       55                      push   %rbp
    123d:       48 8d 2d 54 2b 00 00    lea    0x2b54(%rip),%rbp        # 3d98 <__do_global_dtors_aux_fini_array_entry>
    1244:       53                      push   %rbx
    1245:       4c 29 fd                sub    %r15,%rbp
    1248:       48 83 ec 08             sub    $0x8,%rsp
    124c:       e8 af fd ff ff          callq  1000 <_init>
    1251:       48 c1 fd 03             sar    $0x3,%rbp
    1255:       74 1f                   je     1276 <__libc_csu_init+0x56>
    1257:       31 db                   xor    %ebx,%ebx
    1259:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)
    1260:       4c 89 f2                mov    %r14,%rdx
    1263:       4c 89 ee                mov    %r13,%rsi
    1266:       44 89 e7                mov    %r12d,%edi
    1269:       41 ff 14 df             callq  *(%r15,%rbx,8)
    126d:       48 83 c3 01             add    $0x1,%rbx
    1271:       48 39 dd                cmp    %rbx,%rbp
    1274:       75 ea                   jne    1260 <__libc_csu_init+0x40>
    1276:       48 83 c4 08             add    $0x8,%rsp
    127a:       5b                      pop    %rbx
    127b:       5d                      pop    %rbp
    127c:       41 5c                   pop    %r12
    127e:       41 5d                   pop    %r13
    1280:       41 5e                   pop    %r14
    1282:       41 5f                   pop    %r15
    1284:       c3                      retq
    1285:       66 66 2e 0f 1f 84 00    data16 nopw %cs:0x0(%rax,%rax,1)
    128c:       00 00 00 00

0000000000001290 <__libc_csu_fini>:
    1290:       f3 0f 1e fa             endbr64
    1294:       c3                      retq

Disassembly of section .fini:

0000000000001298 <_fini>:
    1298:       f3 0f 1e fa             endbr64
    129c:       48 83 ec 08             sub    $0x8,%rsp
    12a0:       48 83 c4 08             add    $0x8,%rsp
    12a4:       c3                      retq
```

可以看到，这个程序虽然只是输出 `hello,world.`，但依然很复杂，因为它要包含其它很多的基础资源或者子程序，我们只需要重点关注 `main`：

```bash
0000000000001189 <main>:
    1189:       f3 0f 1e fa             endbr64
    118d:       55                      push   %rbp
    118e:       48 89 e5                mov    %rsp,%rbp
    1191:       48 8d 35 6d 0e 00 00    lea    0xe6d(%rip),%rsi        # 2005 <_ZStL19piecewise_construct+0x1>
    1198:       48 8d 3d 81 2e 00 00    lea    0x2e81(%rip),%rdi        # 4020 <_ZSt4cerr@@GLIBCXX_3.4>
    119f:       e8 dc fe ff ff          callq  1080 <_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@plt>
    11a4:       b8 00 00 00 00          mov    $0x0,%eax
    11a9:       5d                      pop    %rbp
    11aa:       c3                      retq
```

可以看到，这个段是从`0000000000001189`开始的，需要关注的输出语句为：

```cpp
    119f:       e8 dc fe ff ff          callq  1080 <_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@plt>
```

这个地址 `119f` 就是我们需要打断点的地方，被我们找出来了，这个地址是定死的，它在运行的时候，需要加载到内存中。问题是，加载到哪里？

我们并不知道加载到哪里，**换句话说，我们不知道段地址是什么，它不固定**，这主要是为了程序数据安全考虑，采用了内存分布随机化，**我们可以关掉内存分布随机化**：

```cpp
if (pid == 0) {
    personality(ADDR_NO_RANDOMIZE); // 取消随机内存
    // child progress
    // debugged progress
    ptrace(PTRACE_TRACEME, 0, nullptr, nullptr);
    execl(proj, proj, nullptr);
    ...
```
这样它就固定了，我们可以这样查看它在运行的时候的 `map`，首先用我们程序进行调试：

```cpp
./main hw
Start debugging the progress: hw, pid = 260915:
minidbg>
```
可以看到，`pid` 为 `260915`，另开一个 **zsh**，直接查看：

```bash
cat /proc/260915/maps
555555554000-555555555000 r--p 00000000 fc:01 698165                     /root/mydebugger/src/hw
555555555000-555555556000 r-xp 00001000 fc:01 698165                     /root/mydebugger/src/hw
555555556000-555555557000 r--p 00002000 fc:01 698165                     /root/mydebugger/src/hw
555555557000-555555559000 rw-p 00002000 fc:01 698165                     /root/mydebugger/src/hw
7ffff7fcb000-7ffff7fce000 r--p 00000000 00:00 0                          [vvar]
7ffff7fce000-7ffff7fcf000 r-xp 00000000 00:00 0                          [vdso]
7ffff7fcf000-7ffff7fd0000 r--p 00000000 fc:01 1185709                    /usr/lib/x86_64-linux-gnu/ld-2.31.so
7ffff7fd0000-7ffff7ff3000 r-xp 00001000 fc:01 1185709                    /usr/lib/x86_64-linux-gnu/ld-2.31.so
7ffff7ff3000-7ffff7ffb000 r--p 00024000 fc:01 1185709                    /usr/lib/x86_64-linux-gnu/ld-2.31.so
7ffff7ffc000-7ffff7ffe000 rw-p 0002c000 fc:01 1185709                    /usr/lib/x86_64-linux-gnu/ld-2.31.so
7ffff7ffe000-7ffff7fff000 rw-p 00000000 00:00 0
7ffffffde000-7ffffffff000 rw-p 00000000 00:00 0                          [stack]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0
```

可以看到我们的可执行代码也就是 `main` 段在这里：

```bash
555555555000-555555556000 r-xp 00001000 fc:01 698165                     /root/mydebugger/src/hw
```

这以后都是固定的，虽然不安全，但是仅仅是为了演示，就没关系了。段的偏移地址为`555555554000`，因为我们需要打断点的地址为`119f`，因此：

```bash
基址 + 指令相对地址
= 555555554000 + 119f
= 55555555519f
```
可以预见的是，如果 `break 0x55555555519f`，之后执行，并不会打印出 `hello,world`，但是我们如果打到了下一条地址：`0x5555555551a4`，运行之后，就会理解打印出 `hello,world`。以下进行检测：

```bash
./main hw
Start debugging the progress: hw, pid = 261169:
minidbg> break 0x55555555519f
Set breakpoint at address 0x55555555519f
minidbg> continue
minidbg>
```
我们换一个地址：

```bash
./main hw
Start debugging the progress: hw, pid = 261407:
minidbg> break 0x0x5555555551a4
Set breakpoint at address 0x5555555551a4
minidbg> continue
hello,world.
minidbg>
```
可以看到，以下就打印出了`hello,world`。以上符合我们的预期，因此实验是成功的，另外不需要担心 `pid` 不一样，由于我们关闭了地址空间布局随机化（**ASLR**, Address Space Layout Randomization），段地址不会变，因此地址也是固定的。














