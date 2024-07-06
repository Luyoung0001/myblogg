---
title: C++调试：内存管理
date: 2024-05-16 22:44:48
tags: c++ lldb
categories: 经验方法 C++
---

<!--more-->

## 正文
本文将会通过一个例子，通过 `nm`、`lldb` 工具查看 **ELF** 文件的内存管理：

```cpp
#include <iostream>
using namespace std;
/*
说明：C++
中不再区分初始化和未初始化的全局变量、静态变量的存储区，如果非要区分下述程序标注在了括号中
*/
int g_var = 0; // g_var 在全局区（.data 段）
char *gp_var;  // gp_var 在全局区（.bss 段）

int main() {
    int var;     // var 在栈区
    char *p_var; // p_var 在栈区
    char arr[] =
        "abc"; // arr 为数组变量，存储在栈区；"abc"为字符串常量，存储在常量区
    const char *p_var1 =
        "123456"; // p_var1 在栈区；"123456"为字符串常量，存储在常量区
    static int s_var = 0; // s_var 为静态变量，存在静态存储区（.data 段）
    p_var = (char *)malloc(10); // 分配得来的 10 个字节的区域在堆区
    free(p_var);
    return 0;
}
```

编译，通过 `nm` 打印符号信息表（ `nm` 只能打印出全局符号，因为局部变量不会出现在全局符号表中。`nm` 主的用途是列出全局符号，比如全局变量、静态变量以及函数名等。局部变量的信息通常在编译时被分配到栈上，并不包含在全局符号表中）：

```bash
g++ -g mem.cxx -o main
nm ./main
0000000100008028 b __ZZ4mainE5s_var
0000000100008010 d __dyld_private
0000000100000000 T __mh_execute_header
                 U _free
0000000100008018 S _g_var
0000000100008020 S _gp_var
0000000100003f0c T _main
                 U _malloc
                 U dyld_stub_binder
```
可以看到 `_g_var`、`_gp_var` 都位于`.sbss` ，尽管一个初始化一个未初始化，但是C++中不再区分初始化和未初始化的全局变量、静态变量的存储区，因此它们显示都是 `S`。以下是更详细的解释：
在 `nm` 命令输出中，`S` 和 `B` 都指示变量未初始化，但它们并不相同，也不属于同一个类别。每个符号都独立表示不同的存储特性或区段。这里是更详细的区分：

1. **b/B (大写)**:
   - `B` 表示变量位于 `.bss` 段，这是专门用于存储未初始化的全局变量和静态变量的内存区段。这些变量在程序启动时自动被初始化为零。

2. **s/S (大写)**:
   - `S` 通常用于指示小型未初始化数据段，如 `.sbss`。`.sbss` 是 `.bss` 段的一个小段，用于存储较小的未初始化变量，以优化内存的使用。`S` 并不直接属于 `B`，但它表示的是一个与 `B` 类似的概念，专门针对小型数据。

简而言之，`S` 和 `B` 都涉及未初始化的数据，但它们代表的是不同的内存区段。`S` 不属于 `B`，但两者都用于未初始化数据的存储，只是大小和存储优化方面有所区别。因此，在处理大型和小型未初始化数据时，编译器和链接器可能会选择将它们分别放在 `.bss` 或 `.sbss` 段中。

接着用 `lldb` 进行调试，`lldb` 可以查看 `main`  中的局部符号：

```bash
...
Target 0: (main) stopped.
(lldb) frame variable -L var
0x000000016fdfeff8: (int) var = 802912
(lldb) frame variable -L p_var
0x000000016fdfeff0: (char *) p_var = 0x0000000100504080 ""
(lldb) frame variable -L arr
0x000000016fdfefec: (char [4]) arr = "abc"
(lldb) frame variable -L p_var1
0x000000016fdfefe0: (const char *) p_var1 = 0x0000000100003fb0 "123456"
(lldb) frame variable -L s_var
0x0000000100008028: (int) s_var = 0
...
```
这些信息足够查看以及推断：

局部符号     | 地址 | section 
-- |-|-
var  |0x000000016fdfeff8 |栈区
p_var  | 0x000000016fdfeff0|栈区
arr  | 0x000000016fdfefec|栈区
p_var1|0x000000016fdfefe0|栈区
s_var|0x0000000100008028|.bss
"123456"|0x0000000100003fb0|常量区

之所以推断 `0x0000000100003fb0` 位于常量区，依据是上面的信息`0000000100003f0c T _main`，`.text` 接下来就是 `.rodata`，地址很近。




