---
title: debugger（五）：source level stepping
date: 2024-06-11 18:00:33
tags: c++ lldb debugger
categories: C++ debugger
---

<!--more-->

## 〇、前言
前面的源代码打印，利用了 DWARF 格式化的信息，现在我们更进一步，利用它分别进行 **stepi**、**step_over**、**step_in**、**step_out**。

## 一、stepi
这个最简单，我们只需要利用 ptrace 就行：

```cpp
void Debugger::single_step_instruction() {
    ptrace(PTRACE_SINGLESTEP, m_pid, nullptr, nullptr);
    wait_for_signal();
}

void Debugger::single_step_instruction_with_breakpoint_check() {
    if (m_breakpoints.count(get_pc())) {
        step_over_breakpoint();
    }
    else {
        single_step_instruction();
    }
}

void Debugger::step_over_breakpoint() {
  // 二次检查
  if (m_breakPoints.count(get_pc())) {
    auto &bp = m_breakPoints[get_pc()];
    if (bp.is_enabled()) {
      bp.disable();
      ptrace(PTRACE_SINGLESTEP, m_pid, nullptr, nullptr);
      wait_for_signal();
      bp.enable();
    }
  }
}
```

这里需要注意的是，如果改变了被跟踪程序的状态，必须要调用 `wait()`，这是因为我们必须要同步一些信息，比如 pc等寄存器，不然就会保错。

## 二、step_out
要完成这个函数，我们需要对栈帧所有了解。当函数执行到一个函数之中时，比如：

```cpp
void f() {
  int foo = 1;		<-----执行在此处
  int foo1 = 1;
  e();
  int foo2 = 1;
  int foo3 = 1;
}

int main() {
  int foo = 1;
  int foo1 = 1;
  int foo2 = 1;
  int foo3 = 1;
  f();
  int foo4 = 1;
}
```

当 `main()` 函数进入 `f()` 的时候，`f()` 首先会完成 **prologue**，新的栈帧是 `f()` 自己创建的。
### 函数调用过程

1. **Prologue**：函数调用开始时，调用者函数（例如 `main()`）会将当前函数的返回地址压入栈中（事实上，这是在 CALL 中隐式执行的），然后执行被调用函数（例如 `f()`）的 prologue 部分。Prologue 是被调用函数的一部分，用于准备函数的栈帧和执行环境。

2. **栈帧创建**：在 prologue 部分，被调用函数会为自己创建一个新的栈帧。栈帧包含了函数的局部变量、参数、返回地址等信息。这个栈帧通常位于栈上，所以在创建时需要适当地调整栈指针。

3. **保存寄存器状态**：在 prologue 部分，被调用函数还可能需要保存调用前的寄存器状态，以便在函数结束后恢复。

4. **函数执行**：被调用函数开始执行其主体部分，执行其中的语句和操作。

5. **Epilogue**：函数执行完毕后，执行 epilogue 部分。Epilogue 用于清理栈帧和恢复调用前的环境，包括恢复寄存器状态和返回地址。

### 栈帧布局

1. **局部变量分配**：在栈帧中，局部变量通常位于栈顶的一段区域。在调用函数的 prologue 部分，会为局部变量分配空间。

2. **参数传递**：函数参数可以通过栈传递，也可以通过寄存器传递（特别是在寄存器架构中）。参数通常存储在栈帧的特定位置，被调用函数在开始时会从这些位置读取参数。

3. **返回地址**：返回地址是在调用函数 prologue 部分压入栈中的，它指示了在函数执行完毕后应该返回到哪里继续执行。

4. **其他信息**：栈帧还可能包含其他的信息，如前一个函数的栈帧指针、异常处理相关信息等。

### 其它情况

- **递归调用**：每次函数调用都会创建一个新的栈帧，如果函数递归调用自身，会导致多个栈帧同时存在于栈上。

- **内联函数**：内联函数在调用时不会创建新的栈帧，而是将函数的内容嵌入到调用者函数中，从而减少了函数调用的开销。

- **优化技术**：编译器会对函数栈帧进行优化，例如将局部变量寄存器化、使用栈帧重用等，以提高程序的性能和效率。

比如这段代码：

```bash
#1  0x000055555555526d in main () at /home/luyoung/mydebugger/examples/stack.cpp:53
(gdb) disassemble 
Dump of assembler code for function _Z1fv:
   0x0000555555555210 <+0>:     endbr64 
   0x0000555555555214 <+4>:     push   %rbp
   0x0000555555555215 <+5>:     mov    %rsp,%rbp
   0x0000555555555218 <+8>:     sub    $0x10,%rsp
=> 0x000055555555521c <+12>:    movl   $0x1,-0x10(%rbp)
   0x0000555555555223 <+19>:    movl   $0x1,-0xc(%rbp)
   0x000055555555522a <+26>:    call   0x5555555551e0 <_Z1ev>
   0x000055555555522f <+31>:    movl   $0x1,-0x8(%rbp)
   0x0000555555555236 <+38>:    movl   $0x1,-0x4(%rbp)
   0x000055555555523d <+45>:    nop
   0x000055555555523e <+46>:    leave  
   0x000055555555523f <+47>:    ret    
End of assembler dump.
```

对应于：

```cpp
void f() {
  int foo = 1;		<-----执行在此处
  int foo1 = 1;
  e();
  int foo2 = 1;
  int foo3 = 1;
}
```

这就是 `main() ` 进入 `f()` 后所做的工作：

- `endbr64`：这是一个指令，用于指示处理器启用 64 位模式下的 endbr 指令。
- `push %rbp`：将当前栈帧main() 的基址指针（Frame Pointer, RBP）压入栈中，以便后续函数使用。

- `mov %rsp,%rbp`：将栈顶指针（Stack Pointer, RSP）的值赋给基址指针（RBP），建立当前函数的栈帧。

- `sub $0x10,%rsp`：在栈上分配 16 字节的空间，用于存储局部变量或临时数据，更新 rsp。

- `movl $0x1,-0x10(%rbp)`：将立即数 1 存储到相对于基址指针（RBP）偏移量为 -0x10 的内存位置。

- `movl $0x1,-0xc(%rbp)`：将立即数 1 存储到相对于基址指针（RBP）偏移量为 -0xc 的内存位置。

- `call 0x5555555551b0 <_Z1dv>`：调用地址为 `0x5555555551b0` 的函数 `_Z1dv`，这个函数可能接受参数并返回结果。

- `movl $0x1,-0x8(%rbp)`：将立即数 1 存储到相对于基址指针（RBP）偏移量为 -0x8 的内存位置。

- `movl $0x1,-0x4(%rbp)`：将立即数 1 存储到相对于基址指针（RBP）偏移量为 -0x4 的内存位置。

- `nop`：空操作指令，不执行任何操作。

- `leave`：恢复栈帧，将栈帧移出栈。

- `ret`：返回指令，从当前函数返回到调用者函数。



因此我们想要跳出函数 `f()`，得知晓这个函数的返回地址，它放在哪里呢？

返回地址实际上是在调用函数之前的步骤中由调用指令（如 `call`）隐式处理的，它会将下一条指令的地址（即函数 `f()` 的地址）**压入栈**中，作为返回时应该跳转的地址。这里就很清晰了，因为在这之后，紧接着入栈的是 **rbp**，接着是把 **rsp** 放入 **rbp**，当前 **rbp** 中值就是那时候的 **rsp**，也是新的栈帧。我们只要把 **rbp+8**，就能得到它了。

```cpp
void Debugger::step_out() {
    auto frame_pointer = get_register_value(m_pid, reg::rbp);
    auto return_address = read_memory(frame_pointer+8);

    bool should_remove_breakpoint = false;
    if (!m_breakpoints.count(return_address)) {
        set_breakpoint_at_address(return_address);
        should_remove_breakpoint = true;
    }

    continue_execution();

    if (should_remove_breakpoint) {
        remove_breakpoint(return_address);
 
```

这里有一个细节，就是得判断返回地址是不是一个断点，如果是，那就不用管，继续执行之后它自动会停在返回处；如果不是，那就要手动打断点，单步执行之后它就回停在那里，接着将断点取消，就可以了。

## 三、step_in()
这个比较简单，假设在函数直行到：

```cpp
void f() {
  int foo = 1;		<-----执行在此处
  int foo1 = 1;
  e();
  int foo2 = 1;
  int foo3 = 1;
}
```
那么 **step_in** 会继续执行，直行到 `e()` 的时候，会跳进去，我们只需要通过当前 **pc**来获取行号，然后单步执行，一直到行号变化为止。

```cpp
void Debugger::step_in() {
   auto line = get_line_entry_from_pc(get_offset_pc())->line;

   while (get_line_entry_from_pc(get_offset_pc())->line == line) {
      single_step_instruction_with_breakpoint_check();
   }

   auto line_entry = get_line_entry_from_pc(get_offset_pc());
   print_source(line_entry->file->path, line_entry->line);
}

uint64_t Debugger::get_offset_pc() {
   return offset_load_address(get_pc());
}

```
当行号不一样的时候，有可能进入到了一个函数，也有可能进入了本函数的下一行（本行不是函数）。之所以要用 `while()` 是因为 `prace()` 只能按照汇编指令一行一行执行，源代码一行可能对应着多行汇编指令。


## 四、step_over
step_over意味着如果下一行是一个函数，会直接运行下一行结束，而不是进入函数，并且会停在下下一行。

> A couple of horrible options are to keep stepping until we’re at a new line in the current function, or to set a breakpoint at every line in the current function. The former would be ridiculously inefficient if we’re stepping over a function call, as we’d need to single step through every single instruction in that call graph, so I’ll go for the second solution.

这里就需要用第二个方法来解决这个问题，即在当前函数中给所有的行（除了本行）打上断点，然后 **continue**，那么将会停在下一个断点，也就是下下一行。这个方法很妙，当然第一种方法也不赖，它的思路是一直指令级别的 **step**，然后停在行数变化的那一行，这种方法的缺点是效率低。这里用第二种方法，打断点的方法。

当然了，打完断点还得取消断点，这里得设计一个容器，将断点装起来，然后再取消掉。还有一个细节问题，就是如果直行到函数的最后一行，这时候就需要运行完停留在 `main()` 的 `f()` 的下一行了，因此这里还必须将 `f()` 的返回地址也打一个断点：

```cpp
void f() {
  int foo = 1;		
  int foo1 = 1;
  e();
  int foo2 = 1;
  int foo3 = 1;		<-----执行在此处
}

int main() {
  int foo = 1;
  int foo1 = 1;
  int foo2 = 1;
  int foo3 = 1;
  f();
  int foo4 = 1;
}
```


```cpp
void Debugger::step_over() {
    auto func = get_function_from_pc(get_offset_pc());
    auto func_entry = at_low_pc(func);
    auto func_end = at_high_pc(func);

    auto line = get_line_entry_from_pc(func_entry);
    auto start_line = get_line_entry_from_pc(get_offset_pc());

    std::vector<std::intptr_t> to_delete{};

    while (line->address < func_end) {
        auto load_address = offset_dwarf_address(line->address);
        if (line->address != start_line->address && !m_breakpoints.count(load_address)) {
            set_breakpoint_at_address(load_address);
            to_delete.push_back(load_address);
        }
        ++line;
    }
    // 获取本函数的返回地址防止这是本函数的最后一行
    auto frame_pointer = get_register_value(m_pid, reg::rbp);
    auto return_address = read_memory(frame_pointer+8);
    if (!m_breakpoints.count(return_address)) {
        set_breakpoint_at_address(return_address);
        to_delete.push_back(return_address);
    }

    continue_execution();

    for (auto addr : to_delete) {
        remove_breakpoint(addr);
    }
}
```







