---
title: "从 C 语言到二进制程序：预处理、编译、汇编与链接"
date: 2024-08-08 17:22:33
tags:
  - "C"
  - "compiler"
  - "ELF"
categories:
  - "Systems Programming"
---

<!--more-->

## 预处理

```c
#include <stdio.h>
#define MSG \
    "Hello \
World!\n"
int main() {
    printf(MSG /* "hi!\n" */);
#ifdef __riscv
    printf("Hello RISC-V!\n");
#endif
    return 0;
}

```

可以通过 gcc -E a.c 来查看预处理结果：
```bash
...
extern int __uflow (FILE *);
extern int __overflow (FILE *, int);
# 902 "/usr/include/stdio.h" 3 4

# 2 "a.c" 2




# 5 "a.c"
int main() {
    printf("Hello World!\n" );



    return 0;
}
```

这个预处理结果非常多，就是因为 include 了一个头文件，它展开之后就会非常大。因此可以说，预处理本质上就是文本替换，包括上面定义的宏 MSG。

## 头文件怎么查找

```bash
gcc -E a.c --verbose > /dev/null
Using built-in specs.
COLLECT_GCC=/usr/bin/gcc
OFFLOAD_TARGET_NAMES=nvptx-none:amdgcn-amdhsa
OFFLOAD_TARGET_DEFAULT=1
Target: x86_64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Ubuntu 11.4.0-1ubuntu1~22.04' --with-bugurl=file:///usr/share/doc/gcc-11/README.Bugs --enable-languages=c,ada,c++,go,brig,d,fortran,objc,obj-c++,m2 --prefix=/usr --with-gcc-major-version-only --program-suffix=-11 --program-prefix=x86_64-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --enable-bootstrap --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-vtable-verify --enable-plugin --enable-default-pie --with-system-zlib --enable-libphobos-checking=release --with-target-system-zlib=auto --enable-objc-gc=auto --enable-multiarch --disable-werror --enable-cet --with-arch-32=i686 --with-abi=m64 --with-multilib-list=m32,m64,mx32 --enable-multilib --with-tune=generic --enable-offload-targets=nvptx-none=/build/gcc-11-XeT9lY/gcc-11-11.4.0/debian/tmp-nvptx/usr,amdgcn-amdhsa=/build/gcc-11-XeT9lY/gcc-11-11.4.0/debian/tmp-gcn/usr --without-cuda-driver --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=x86_64-linux-gnu --with-build-config=bootstrap-lto-lean --enable-link-serialization=2
Thread model: posix
Supported LTO compression algorithms: zlib zstd
gcc version 11.4.0 (Ubuntu 11.4.0-1ubuntu1~22.04)
COLLECT_GCC_OPTIONS='-E' '-v' '-mtune=generic' '-march=x86-64'
 /usr/lib/gcc/x86_64-linux-gnu/11/cc1 -E -quiet -v -imultiarch x86_64-linux-gnu a.c -mtune=generic -march=x86-64 -fasynchronous-unwind-tables -fstack-protector-strong -Wformat -Wformat-security -fstack-clash-protection -fcf-protection -dumpbase a.c -dumpbase-ext .c
ignoring nonexistent directory "/usr/local/include/x86_64-linux-gnu"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/include-fixed"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/../../../../x86_64-linux-gnu/include"
#include "..." search starts here:
#include <...> search starts here:
 /usr/lib/gcc/x86_64-linux-gnu/11/include
 /usr/local/include
 /usr/include/x86_64-linux-gnu
 /usr/include
End of search list.
COMPILER_PATH=/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/:/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/
LIBRARY_PATH=/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../../lib/:/lib/x86_64-linux-gnu/:/lib/../lib/:/usr/lib/x86_64-linux-gnu/:/usr/lib/../lib/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../:/lib/:/usr/lib/
COLLECT_GCC_OPTIONS='-E' '-v' '-mtune=generic' '-march=x86-64'

```

这段信息非常详细，比如它优先搜索include "..." 路径，其次才是include <>。以下是详细解释：

1. **Using built-in specs, COLLECT_GCC, OFFLOAD_TARGET_NAMES, etc.:**
   - 这些信息描述了编译器的配置和环境。
   - `COLLECT_GCC=/usr/bin/gcc` 表示 gcc 的执行路径。
   - `OFFLOAD_TARGET_NAMES` 和 `OFFLOAD_TARGET_DEFAULT` 涉及到目标设备，用于代码卸载至特定的加速器（如 GPU）。
   - `Target: x86_64-linux-gnu` 指明目标平台是 x86_64 架构的 Linux 系统。

2. **Configured with:**
   - 这一行列出了编译 GCC 时的配置选项。它包括支持的语言（如 C, C++），目标架构，优化选项等。

3. **Thread model, Supported LTO compression algorithms, gcc version:**
   - `Thread model: posix` 表明 GCC 使用 POSIX 线程模型。
   - `Supported LTO compression algorithms: zlib zstd` 列出了链接时优化（LTO）支持的压缩算法。
   - `gcc version 11.4.0` 显示 GCC 的版本。

4. **COLLECT_GCC_OPTIONS:**
   - 列出了传递给 gcc 的选项，如 `-E`（只进行预处理），`-v`（详细输出），以及与目标机器有关的优化设置。

5. **/usr/lib/gcc/x86_64-linux-gnu/11/cc1:**
   - 这是实际执行预处理的程序路径。`cc1` 是 GCC 的一个内部组件，用于处理 C 语言文件。

6. **ignoring nonexistent directory:**
   - 这些行显示编译器在查找头文件时跳过了不存在的目录。

7. **#include "<...>" search starts here:**
   - 列出了编译器在处理包含文件时搜索的目录。这对于解决包含文件的依赖关系非常重要。

8. **COMPILER_PATH, LIBRARY_PATH:**
   - `COMPILER_PATH` 显示了编译器相关工具链的位置。
   - `LIBRARY_PATH` 列出了链接器搜索库文件的路径。


怎么验证呢？重新写一下代码：

```c
#include "stdio.h"
#define MSG \
    "Hello \
World!\n"
int main() {
    printf(MSG /* "hi!\n" */);
#ifdef __riscv
    printf("Hello RISC-V!\n");
#endif
    return 0;
}
```

之后，在文件下创建一个 aaa 目录并创建一个 stdin.h，然后预编译并查看预处理信息：
```bash
gcc -E a.c -Iaaa -v > /dev/null
Using built-in specs.
COLLECT_GCC=/usr/bin/gcc
OFFLOAD_TARGET_NAMES=nvptx-none:amdgcn-amdhsa
OFFLOAD_TARGET_DEFAULT=1
Target: x86_64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Ubuntu 11.4.0-1ubuntu1~22.04' --with-bugurl=file:///usr/share/doc/gcc-11/README.Bugs --enable-languages=c,ada,c++,go,brig,d,fortran,objc,obj-c++,m2 --prefix=/usr --with-gcc-major-version-only --program-suffix=-11 --program-prefix=x86_64-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --enable-bootstrap --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-vtable-verify --enable-plugin --enable-default-pie --with-system-zlib --enable-libphobos-checking=release --with-target-system-zlib=auto --enable-objc-gc=auto --enable-multiarch --disable-werror --enable-cet --with-arch-32=i686 --with-abi=m64 --with-multilib-list=m32,m64,mx32 --enable-multilib --with-tune=generic --enable-offload-targets=nvptx-none=/build/gcc-11-XeT9lY/gcc-11-11.4.0/debian/tmp-nvptx/usr,amdgcn-amdhsa=/build/gcc-11-XeT9lY/gcc-11-11.4.0/debian/tmp-gcn/usr --without-cuda-driver --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=x86_64-linux-gnu --with-build-config=bootstrap-lto-lean --enable-link-serialization=2
Thread model: posix
Supported LTO compression algorithms: zlib zstd
gcc version 11.4.0 (Ubuntu 11.4.0-1ubuntu1~22.04)
COLLECT_GCC_OPTIONS='-E' '-I' 'aaa' '-v' '-mtune=generic' '-march=x86-64'
 /usr/lib/gcc/x86_64-linux-gnu/11/cc1 -E -quiet -v -I aaa -imultiarch x86_64-linux-gnu a.c -mtune=generic -march=x86-64 -fasynchronous-unwind-tables -fstack-protector-strong -Wformat -Wformat-security -fstack-clash-protection -fcf-protection -dumpbase a.c -dumpbase-ext .c
ignoring nonexistent directory "/usr/local/include/x86_64-linux-gnu"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/include-fixed"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/../../../../x86_64-linux-gnu/include"
#include "..." search starts here:
#include <...> search starts here:
 aaa
 /usr/lib/gcc/x86_64-linux-gnu/11/include
 /usr/local/include
 /usr/include/x86_64-linux-gnu
 /usr/include
End of search list.
In file included from a.c:1:
aaa/stdio.h:1:10: error: #include expects "FILENAME" or <FILENAME>
    1 | #include printf(...)
```

可以看到它确实在本目录下搜索，并且报错。

预处理的其他工作：
- 去掉注释;
- 连接因断行符(行尾的\)而拆分的字符串;
- 处理条件编译 #ifdef/#else/#endif;
- 字符串化 #
- 标识符连接 ##

如果用`riscv64-linux-gnu-gcc -E a.c` 来处理：

```bash
...
extern int __uflow (FILE *);
extern int __overflow (FILE *, int);
# 902 "/usr/riscv64-linux-gnu/include/stdio.h" 3

# 2 "a.c" 2




# 5 "a.c"
int main() {
    printf("Hello World!\n" );

    printf("Hello RISC-V!\n");

    return 0;
}
```

## 编译

### 词法分析

对于示例代码：
```c
#include <stdio.h>
int main() {  // compute 1 + 2
    int x = 1, y = 2;
    int z = x + y;
    printf("z = %d\n", z);
    return 0;
}

```
`clang -fsyntax-only -Xclang -dump-tokens b.c`:
```bash
...
equal '='        [LeadingSpace] Loc=<b.c:3:18>
numeric_constant '2'     [LeadingSpace] Loc=<b.c:3:20>
semi ';'                Loc=<b.c:3:21>
int 'int'        [StartOfLine] [LeadingSpace]   Loc=<b.c:4:5>
identifier 'z'   [LeadingSpace] Loc=<b.c:4:9>
equal '='        [LeadingSpace] Loc=<b.c:4:11>
identifier 'x'   [LeadingSpace] Loc=<b.c:4:13>
plus '+'         [LeadingSpace] Loc=<b.c:4:15>
identifier 'y'   [LeadingSpace] Loc=<b.c:4:17>
semi ';'                Loc=<b.c:4:18>
identifier 'printf'      [StartOfLine] [LeadingSpace]   Loc=<b.c:5:5>
l_paren '('             Loc=<b.c:5:11>
string_literal '"z = %d\n"'             Loc=<b.c:5:12>
comma ','               Loc=<b.c:5:22>
identifier 'z'   [LeadingSpace] Loc=<b.c:5:24>
r_paren ')'             Loc=<b.c:5:25>
semi ';'                Loc=<b.c:5:26>
return 'return'  [StartOfLine] [LeadingSpace]   Loc=<b.c:6:5>
numeric_constant '0'     [LeadingSpace] Loc=<b.c:6:12>
semi ';'                Loc=<b.c:6:13>
r_brace '}'      [StartOfLine]  Loc=<b.c:7:1>
eof ''          Loc=<b.c:7:2>
```
可以看到，词法分析本质上就是识别并记录源文件中的每一个token：

- 标识符, 关键字, 常数, 字符串, 运算符, 大括号, 分号…；
- 还记录了token的位置(文件名:行号:列号)。

词法分析器本质上是一个字符串匹配程序。

### 语法分析

`clang -fsyntax-only -Xclang -ast-dump a.c`:

```bash
...
| |-ParmVarDecl 0x11a0378 <col:24, col:29> col:30 'FILE *'
| `-ParmVarDecl 0x11a03f8 <col:32> col:35 'int'
`-FunctionDecl 0x11a0610 <b.c:2:1, line:7:1> line:2:5 main 'int ()'
  `-CompoundStmt 0x11a0ae8 <col:12, line:7:1>
    |-DeclStmt 0x11a0808 <line:3:5, col:21>
    | |-VarDecl 0x11a06c8 <col:5, col:13> col:9 used x 'int' cinit
    | | `-IntegerLiteral 0x11a0730 <col:13> 'int' 1
    | `-VarDecl 0x11a0768 <col:5, col:20> col:16 used y 'int' cinit
    |   `-IntegerLiteral 0x11a07d0 <col:20> 'int' 2
    |-DeclStmt 0x11a0930 <line:4:5, col:18>
    | `-VarDecl 0x11a0838 <col:5, col:17> col:9 used z 'int' cinit
    |   `-BinaryOperator 0x11a0910 <col:13, col:17> 'int' '+'
    |     |-ImplicitCastExpr 0x11a08e0 <col:13> 'int' <LValueToRValue>
    |     | `-DeclRefExpr 0x11a08a0 <col:13> 'int' lvalue Var 0x11a06c8 'x' 'int'
    |     `-ImplicitCastExpr 0x11a08f8 <col:17> 'int' <LValueToRValue>
    |       `-DeclRefExpr 0x11a08c0 <col:17> 'int' lvalue Var 0x11a0768 'y' 'int'
    |-CallExpr 0x11a0a40 <line:5:5, col:25> 'int'
    | |-ImplicitCastExpr 0x11a0a28 <col:5> 'int (*)(const char *, ...)' <FunctionToPointerDecay>
    | | `-DeclRefExpr 0x11a0948 <col:5> 'int (const char *, ...)' Function 0x1185648 'printf' 'int (const char *, ...)'
    | |-ImplicitCastExpr 0x11a0a88 <col:12> 'const char *' <NoOp>
    | | `-ImplicitCastExpr 0x11a0a70 <col:12> 'char *' <ArrayToPointerDecay>
    | |   `-StringLiteral 0x11a09a8 <col:12> 'char[8]' lvalue "z = %d\n"
    | `-ImplicitCastExpr 0x11a0aa0 <col:24> 'int' <LValueToRValue>
    |   `-DeclRefExpr 0x11a09c8 <col:24> 'int' lvalue Var 0x11a0838 'z' 'int'
    `-ReturnStmt 0x11a0ad8 <line:6:5, col:12>
      `-IntegerLiteral 0x11a0ab8 <col:12> 'int' 0

```

可以看到，语法分析就是将 token 组织成树形结构，AST(Abstract Syntax Tree), 可以反映出源程序的层次结构，还报告语法错误, 例如漏了分号。

### 语义分析

按照C语言的语义确定AST中每个表达式的类型：
- 相容的类型将根据C语言标准规范进行类型转换
  - 算术类型转换
- 报告语义错误
  - 未定义的引用
  - 运算符的操作数类型不匹配(如struct + int)
  - 函数调用参数的类型和数量不匹配
  - …


### 静态程序分析

本质上就是分析 AST 的信息，比如语法错误、代码风格、规范、性能、漏洞、逻辑错误等等等等。

### 中间代码生成
`clang -S -emit-llvm a.c`:
```bash
; Function Attrs: noinline nounwind optnone uwtable
define dso_local i32 @main() #0 {
  %1 = alloca i32, align 4
  %2 = alloca i32, align 4
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  store i32 1, i32* %2, align 4
  store i32 2, i32* %3, align 4
  %5 = load i32, i32* %2, align 4
  %6 = load i32, i32* %3, align 4
  %7 = add nsw i32 %5, %6
  store i32 %7, i32* %4, align 4
  %8 = load i32, i32* %4, align 4
  %9 = call i32 (i8*, ...) @printf(i8* noundef getelementptr inbounds ([8 x i8], [8 x i8]* @.str, i64 0, i64 0), i32 noundef %8)
  ret i32 0
}
```
中间代码很关键，它是编译器定义的，它可以面向不同的平台、不同的指令集等等。它作为代码抽象层，可以支持多种源语言和目标语言(硬件指令集)。

```bash
 gcc -c b.c
$ objdump -d b.o

b.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:   f3 0f 1e fa             endbr64
   4:   55                      push   %rbp
   5:   48 89 e5                mov    %rsp,%rbp
   8:   48 83 ec 10             sub    $0x10,%rsp
   c:   c7 45 f4 01 00 00 00    movl   $0x1,-0xc(%rbp)
  13:   c7 45 f8 02 00 00 00    movl   $0x2,-0x8(%rbp)
  1a:   8b 55 f4                mov    -0xc(%rbp),%edx
  1d:   8b 45 f8                mov    -0x8(%rbp),%eax
  20:   01 d0                   add    %edx,%eax
  22:   89 45 fc                mov    %eax,-0x4(%rbp)
  25:   8b 45 fc                mov    -0x4(%rbp),%eax
  28:   89 c6                   mov    %eax,%esi
  2a:   48 8d 05 00 00 00 00    lea    0x0(%rip),%rax        # 31 <main+0x31>
  31:   48 89 c7                mov    %rax,%rdi
  34:   b8 00 00 00 00          mov    $0x0,%eax
  39:   e8 00 00 00 00          call   3e <main+0x3e>
  3e:   b8 00 00 00 00          mov    $0x0,%eax
  43:   c9                      leave
  44:   c3                      ret

```

### 未指定行为、未定义行为、ABI

C语言标准并没有明确定义类型的长度, 而是全部交给具体实现来定义，不过定义了类型的最小范围：

```c
// 假设1字节 = 8比特
sizeof(  signed char     ) >= 1
sizeof(unsigned char     ) >= 1
sizeof(  signed short    ) >= 2
sizeof(unsigned short    ) >= 2
sizeof(         int      ) >= 2
sizeof(unsigned int      ) >= 2
sizeof(         long     ) >= 4
sizeof(unsigned long     ) >= 4
sizeof(         long long) >= 8
sizeof(unsigned long long) >= 8
```
可以看到，事实上C 标准连 1 个字节是多少都没有定义，历史上 1~48bit 都用过。

- 3.6 byte
  - NOTE 2   A byte is composed of a contiguous sequence of bits, the number of which is
implementation-defined

函数调用时参数求值顺序是unspecified B。

实现定义行为(Implementation-defined Behavior)：一类特殊的未指定行为, 具体实现需要将选择写到文档里。

未定义行为：不确定的、错误的行为。比如有符号数溢出，比如 uint8_t a = 128 经常被错误的理解为 -1，事实上，这是 UB，没有这样定义，这也是为什么 PA1中为什么要用无符号数，不产生超范围的 UB。

ABI: 在真实的系统中, 这相当于编译器+(处理器+操作系统+运行库)。















