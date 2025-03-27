---
title: C语言内联汇编
date: 2025-2-6 17:57:27
tags: C语言
categories: 汇编
---
<!--more-->

## [](#一个示例 "一个示例")一个示例

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><code class="hljs c"><span class="hljs-keyword">asm</span> <span class="hljs-title function_">volatile</span><span class="hljs-params">(</span><br><span class="hljs-params">       <span class="hljs-string">"jalr x0, 0(%0)"</span></span><br><span class="hljs-params">       :</span><br><span class="hljs-params">       : <span class="hljs-string">"r"</span>(jump_address)</span><br><span class="hljs-params">       :</span><br><span class="hljs-params">   )</span>;<br></code></pre></td></tr></tbody></table>

**代码片段的格式和含义分解：**

这段代码使用了 GCC 扩展内联汇编的通用格式，其基本结构如下：

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><code class="hljs c"><span class="hljs-keyword">asm</span> [<span class="hljs-keyword">volatile</span>] (<br>  汇编指令模板 :  <span class="hljs-comment">// 字符串，包含汇编指令和占位符</span><br>  输出操作数列表 : <span class="hljs-comment">// 可选，指定汇编指令的输出操作数</span><br>  输入操作数列表 : <span class="hljs-comment">// 可选，指定汇编指令的输入操作数</span><br>  破坏列表        <span class="hljs-comment">// 可选，指定汇编指令可能修改的寄存器或内存</span><br>);<br></code></pre></td></tr></tbody></table>

1.  **`asm volatile(`**

    +   **`asm`**: 关键字，表示开始内联汇编代码块。
    +   **`volatile`**: 可选关键字。使用 `volatile` 的目的是告诉编译器，这段内联汇编代码具有**副作用**（例如，修改了内存或影响了程序状态），**不要对这段代码进行任何优化**，每次调用都必须严格按照代码编写的顺序执行。在跳转指令这种场景下，通常需要使用 `volatile`，以确保跳转行为不被编译器优化掉。
2.  **`"jalr x0, 0(%0)"`**

    +   这是 **汇编指令模板**，是一个**字符串**，包含了实际的 RISC-V 汇编指令。
    +   **`jalr x0, 0(%0)`**: 这是 RISC-V 的 **跳转指令**。
        +   **`jalr`**: 指令名，表示 “Jump and Link Register”。 它会执行跳转，并将返回地址（跳转指令的下一条指令的地址）保存在指定的寄存器中，但在这里，由于目标寄存器是 `x0` (zero 寄存器)，所以实际上忽略了返回地址的保存，它只执行跳转功能。
        +   **`x0`**: 目标寄存器，用于存放返回地址。 在这里被指定为 `x0`，表示**不保存返回地址，只进行跳转**。
        +   **`0(%0)`**: 跳转目标地址的计算方式。
            +   **`%0`**: 这是一个\*\*占位符 (Placeholder)\*\*，用于在汇编指令模板中引用 C 语言的变量。 `%0` 表示 **第一个操作数** (这里的第一个操作数指的是后面 `输入操作数列表` 中的第一个操作数)。
            +   **`(%0)`**: 表示 **间接寻址**。 它会将 `%0` 所代表的寄存器中的值，作为**内存地址**，并从该内存地址读取数据。
            +   **`0(%0)` 完整含义**: 表示跳转目标地址是：**寄存器 `%0` 中存储的地址值 + 偏移量 0**。 实际上就是直接使用寄存器 `%0` 中的值作为跳转目标地址。
3.  **`: // 输出操作数: 无`**

    +   **输出操作数列表**: 位于第一个冒号 `:` 之后，用于指定汇编指令的**输出操作数**。
    +   **`:` 之后为空**: 表示这段汇编代码 **没有输出操作数**，即汇编指令执行后，不会将结果值写回到 C 语言的变量中。
4.  **`: "r"(jump_address) // 输入操作数: jump_address (C变量) 映射到寄存器`**

    +   **输入操作数列表**: 位于第二个冒号 `:` 之后，用于指定汇编指令的**输入操作数**。
    +   **`"r"(jump_address)`**: 指定了一个输入操作数。
        +   **`"r"`**: **约束 (Constraint) 字符串**。 `"r"` 约束告诉编译器，将 `jump_address` 这个 C 变量 **分配到一个通用寄存器** (register)。 编译器会选择一个合适的通用寄存器，并将 `jump_address` 的值加载到这个寄存器中，然后在汇编指令模板中使用 `%0` 占位符来引用这个寄存器。
        +   **`(jump_address)`**: **C 语言表达式**。 `jump_address` 是一个 C 变量，它提供了汇编指令所需的输入值，即跳转的目标地址。
5.  **`: "memory" // 破坏性操作: 无`**

    +   **破坏列表 (Clobber List)**: 位于第三个冒号 `:` 之后，用于告诉编译器，这段内联汇编代码可能会**修改某些寄存器或内存位置**，从而阻止编译器进行某些可能导致错误的优化。
    +   **`"memory"`**: **破坏描述符 (Clobber Descriptor)**。 `"memory"` 告知编译器，这段内联汇编代码可能会**修改内存中的数据**。 即使代码中没有显式地写内存操作指令，但如果汇编代码的执行可能导致内存状态改变（例如，跳转到未知的代码区域，该区域的代码可能会修改内存），也需要使用 `"memory"` 来保守地告诉编译器，防止编译器做出错误的假设。 在本例中，虽然 `jalr` 指令本身不直接修改内存，但由于它是一个跳转指令，可能会跳转到任意地址的代码执行，为了安全起见，使用 `"memory"` 是一个好的实践。
    +   **`: "memory"` 完整含义**: 表示这段内联汇编代码可能会修改内存。

**代码功能：**

这段内嵌汇编代码的功能是： \*\*执行一个间接跳转 (register jump)\*\*。 跳转的目标地址 **由 C 变量 `jump_address` 的值来指定**。 代码会将 `jump_address` 的值加载到一个寄存器中，然后使用 `jalr x0, 0(register)` 指令，以寄存器中的值作为目标地址进行跳转，但不保存返回地址。 由于使用了 `volatile` 和 `"memory"` clobber，编译器会保证这段代码被严格执行，并且不会因为优化而产生意外的行为。 这种代码通常用于需要直接控制程序流程，例如跳转到特定的地址执行某些初始化代码、处理异常、或者实现某些特殊的控制流逻辑等场景。

* * *

## [](#基本语法格式 "基本语法格式")基本语法格式

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br></pre></td><td class="code"><pre><code class="hljs c"><span class="hljs-keyword">asm</span> [<span class="hljs-keyword">volatile</span>] (<br>  <span class="hljs-string">"assembly code template"</span><br>  : [output operands]<br>  : [input operands]<br>  : [clobber <span class="hljs-built_in">list</span>]<br>);<br></code></pre></td></tr></tbody></table>

+   **`asm`**: 关键字，开始内联汇编块。
+   **`[volatile]`**: 可选，`volatile` 关键字告知编译器不要优化这段汇编代码。
+   **`"assembly code template"`**: 汇编指令字符串，可以包含多条指令，指令之间用分号 `;` 或换行符 `\n` 分隔。 可以使用占位符 `%0`, `%1`, `%2`… 引用操作数。
+   **`: [output operands]`**: 可选，输出操作数列表。 每个输出操作数形如 `"[约束](C 变量)"`，多个操作数用逗号 `,` 分隔。
+   **`: [input operands]`**: 可选，输入操作数列表。 每个输入操作数形如 `"[约束](C 表达式)"`，多个操作数用逗号 `,` 分隔。
+   **`: [clobber list]`**: 可选，破坏列表。 用双引号 `" "` 括起来的寄存器名或特殊符号（如 `"memory"`），多个破坏描述符用逗号 `,` 分隔。

## [](#操作数和约束-Operands-and-Constraints "操作数和约束 (Operands and Constraints)")操作数和约束 (Operands and Constraints)

操作数用于在 C 代码和汇编代码之间传递数据。 约束字符串用于指定操作数的类型和位置 (寄存器、内存等)。

**常用约束 (RISC-V GCC 常用约束，更完整的约束列表请查阅 GCC RISC-V 扩展文档):**

+   **`r`**: 通用寄存器 (General-purpose register)。 编译器会为操作数分配一个合适的通用寄存器 (例如 `x1` - `x31`)。
+   **`i`**: 立即数 (immediate value)。 操作数是一个编译时可知的立即数 (整数常量)。
+   **`f`**: 浮点寄存器 (Floating-point register) (用于浮点操作)。 RISC-V 32 位架构中，通常使用扩展指令集 ‘F’ 或 ‘D’ 支持浮点运算。
+   **`m`**: 内存操作数 (memory operand)。 操作数是一个内存地址。
+   **`I`**: 0-31 范围内的立即数 (适用于移位指令的移位量)。

**操作数占位符:**

+   在汇编指令模板中，使用 `%0`, `%1`, `%2`… 来引用操作数。
+   `%0` 对应第一个操作数，`%1` 对应第二个操作数，以此类推。
+   输出操作数从 `%0` 开始编号，输入操作数从输出操作数之后继续编号。 例如，如果有一个输出操作数，那么第一个输入操作数就是 `%1`。
+   在 RISC-V 汇编中，寄存器通常用 `x0`, `x1`, `x2`… 表示，而在内联汇编的操作数中，我们使用占位符 `%0`, `%1`… ，编译器会负责将这些占位符替换为实际分配的寄存器或立即数。

## [](#破坏列表-Clobber-List "破坏列表 (Clobber List)")破坏列表 (Clobber List)

+   破坏列表用于告知编译器，内联汇编代码可能会修改某些资源，从而影响编译器的优化决策。
+   **常用破坏描述符:**
    +   **寄存器名 (例如 `"x1"`, `"x5"`, `"x10"`)**: 表示汇编代码可能会修改指定的寄存器。 如果汇编代码显式地修改了某个寄存器，或者依赖于某个寄存器的值，就应该将其添加到破坏列表中。
    +   **`"memory"`**: 表示汇编代码可能会修改内存。 当汇编代码访问了内存（例如，通过 `lw` 或 `sw` 指令，或者像 `jalr` 这样的跳转指令可能执行未知的内存操作），就应该使用 `"memory"`。
    +   **`"cc"`**: 表示汇编代码可能会修改条件码寄存器 (Condition Code register, RISC-V 中条件码隐含在比较指令和分支指令中，通常不需要显式 clobber “cc”)。

## [](#简单示例教程-RISC-V-32 "简单示例教程 (RISC-V 32)")简单示例教程 (RISC-V 32)

**示例 1: 将立即数加载到寄存器**

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br></pre></td><td class="code"><pre><code class="hljs c"><span class="hljs-meta">#<span class="hljs-keyword">include</span> <span class="hljs-string">&lt;stdio.h&gt;</span></span><p><span class="hljs-type">int</span> <span class="hljs-title function_">main</span><span class="hljs-params">()</span> {<br>  <span class="hljs-type">int</span> value;<br>  <span class="hljs-keyword">asm</span> (<br>    <span class="hljs-string">"li %0, 100"</span>   <span class="hljs-comment">// RISC-V: li (load immediate) 指令，将 100 加载到寄存器</span><br>    : <span class="hljs-string">"=r"</span>(value)  <span class="hljs-comment">// 输出操作数: value (C变量) 映射到寄存器，使用 "=r" 表示输出，且分配寄存器</span><br>    :              <span class="hljs-comment">// 输入操作数: 无</span><br>    :              <span class="hljs-comment">// 破坏列表: 无</span><br>  );<br>  <span class="hljs-built_in">printf</span>(<span class="hljs-string">"Value: %d\n"</span>, value); <span class="hljs-comment">// 输出 value 的值，应该是 100</span><br>  <span class="hljs-keyword">return</span> <span class="hljs-number">0</span>;<br>}</p></code></pre></td></tr></tbody></table>

+   **`"li %0, 100"`**: 汇编指令模板，`li` (load immediate) 指令将立即数 100 加载到寄存器 `%0`。
+   **`"=r"(value)`**: 输出操作数。
    +   **`=r`**: 约束字符串。 `=` 表示输出操作数 (write-only)，`r` 表示分配通用寄存器。
    +   **(value)**: C 变量 `value`，汇编指令的执行结果将存储到这个变量中。
+   **输出**: 程序会输出 “Value: 100”。

**示例 2: 寄存器加法**

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br></pre></td><td class="code"><pre><code class="hljs c"><span class="hljs-meta">#<span class="hljs-keyword">include</span> <span class="hljs-string">&lt;stdio.h&gt;</span></span><p><span class="hljs-type">int</span> <span class="hljs-title function_">main</span><span class="hljs-params">()</span> {<br>  <span class="hljs-type">int</span> a = <span class="hljs-number">5</span>, b = <span class="hljs-number">10</span>, sum;<br>  <span class="hljs-keyword">asm</span> (<br>    <span class="hljs-string">"add %0, %1, %2"</span>  <span class="hljs-comment">// RISC-V: add 指令，将 %1 和 %2 相加，结果存入 %0</span><br>    : <span class="hljs-string">"=r"</span>(sum)       <span class="hljs-comment">// 输出操作数: sum (C变量)，分配寄存器，结果写入 sum</span><br>    : <span class="hljs-string">"r"</span>(a), <span class="hljs-string">"r"</span>(b)  <span class="hljs-comment">// 输入操作数: a 和 b (C变量)，分配寄存器，作为加法指令的输入</span><br>    :                 <span class="hljs-comment">// 破坏列表: 无</span><br>  );<br>  <span class="hljs-built_in">printf</span>(<span class="hljs-string">"Sum: %d\n"</span>, sum); <span class="hljs-comment">// 输出 sum 的值，应该是 15</span><br>  <span class="hljs-keyword">return</span> <span class="hljs-number">0</span>;<br>}</p></code></pre></td></tr></tbody></table>

+   **`"add %0, %1, %2"`**: RISC-V `add` 指令，将寄存器 `%1` 和 `%2` 的值相加，结果存入寄存器 `%0`。
+   **`"=r"(sum)`**: 输出操作数，结果写入 C 变量 `sum`。
+   **`"r"(a), "r"(b)`**: 输入操作数，C 变量 `a` 和 `b` 的值作为加法指令的输入。
+   **输出**: 程序会输出 “Sum: 15”。

**示例 3: 内存加载 (Load Word) 和破坏列表**

<table><tbody><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br><span class="line">3</span><br><span class="line">4</span><br><span class="line">5</span><br><span class="line">6</span><br><span class="line">7</span><br><span class="line">8</span><br><span class="line">9</span><br><span class="line">10</span><br><span class="line">11</span><br><span class="line">12</span><br><span class="line">13</span><br><span class="line">14</span><br><span class="line">15</span><br><span class="line">16</span><br><span class="line">17</span><br></pre></td><td class="code"><pre><code class="hljs c"><span class="hljs-meta">#<span class="hljs-keyword">include</span> <span class="hljs-string">&lt;stdio.h&gt;</span></span><p><span class="hljs-type">int</span> <span class="hljs-title function_">main</span><span class="hljs-params">()</span> {<br>  <span class="hljs-type">int</span> data_in_memory = <span class="hljs-number">0x12345678</span>;<br>  <span class="hljs-type">int</span> loaded_value;<br>  <span class="hljs-type">int</span> *addr_ptr = &amp;data_in_memory;</p><p>  <span class="hljs-keyword">asm</span> <span class="hljs-title function_">volatile</span> <span class="hljs-params">(</span><br><span class="hljs-params">    <span class="hljs-string">"lw %0, (%1)"</span>     <span class="hljs-comment">// RISC-V: lw (load word) 指令，从地址 (%1) 加载一个字到 %0</span></span><br><span class="hljs-params">    : <span class="hljs-string">"=r"</span>(loaded_value) <span class="hljs-comment">// 输出操作数: loaded_value (C变量)</span></span><br><span class="hljs-params">    : <span class="hljs-string">"r"</span>(addr_ptr)     <span class="hljs-comment">// 输入操作数: addr_ptr (C变量，指向内存地址)</span></span><br><span class="hljs-params">    : <span class="hljs-string">"memory"</span>         <span class="hljs-comment">// 破坏列表: "memory"，表示汇编代码可能会访问内存</span></span><br><span class="hljs-params">  )</span>;</p><p>  <span class="hljs-built_in">printf</span>(<span class="hljs-string">"Loaded value: 0x%x\n"</span>, loaded_value); <span class="hljs-comment">// 输出从内存加载的值，应该是 0x12345678</span><br>  <span class="hljs-keyword">return</span> <span class="hljs-number">0</span>;<br>}</p></code></pre></td></tr></tbody></table>

+   **`"lw %0, (%1)"`**: RISC-V `lw` (load word) 指令，从内存地址 `(%1)` 加载一个 32 位字到寄存器 `%0`。
+   **`"=r"(loaded_value)`**: 输出操作数，加载的值存入 `loaded_value`。
+   **`"r"(addr_ptr)`**: 输入操作数，C 指针变量 `addr_ptr` (包含内存地址) 作为加载指令的地址来源。
+   **`"memory"`**: 破坏列表，因为 `lw` 指令会从内存中读取数据，因此使用 `"memory"` 是必要的，尽管本例中并没有 *修改* 内存，但 `lw` 指令的内存访问也可能影响编译器的优化假设。
+   **输出**: 程序会输出 “Loaded value: 0x12345678”。

C 语言内联汇编提供了一种在 C 代码中直接编写汇编指令的方式，可以实现一些用纯 C 代码难以或无法完成的底层操作和优化。 掌握内联汇编的关键在于理解其语法格式、操作数约束、破坏列表，并结合具体的处理器架构（如 RISC-V）的汇编指令集进行实践。