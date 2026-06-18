---
title: "C 语言关键字解析：static 与 inline"
date: 2024-09-17 12:50:40
tags:
  - "C"
  - "static"
  - "inline"
categories:
  - "Programming Languages"
---

<!--more-->

## static、inline

### **1. `static` 关键字**

- **作用**：限制函数的作用域仅限于定义它的源文件（即内部链接）。这意味着每个包含该头文件的源文件都会生成该函数的一个独立实例，避免了多个定义导致的链接错误。
- **在头文件中使用**：适用于小型、频繁调用的函数，允许每个源文件拥有自己的函数副本，避免链接器的重复定义错误。

### **2. `inline` 关键字**

- **作用**：建议编译器将函数调用替换为函数体本身，以减少函数调用的开销，提高性能。
- **在头文件中使用**：结合 `static` 使用时，可以在多个源文件中安全地定义内联函数，而不会导致链接冲突。
- **链接行为**：
  - **C99 标准**：`inline` 函数具有外部链接，但需要在某个翻译单元中提供一个非 `inline` 的定义，供链接器使用。
  - **GNU 扩展**：GNU C 允许 `inline` 函数在头文件中被多次定义，只要它们是 `inline` 的，并且链接器能够处理这些定义。

### **组合使用 `static inline`**

- **消除重复定义**：在头文件中使用 `static inline`，每个包含该头文件的源文件都会有自己的函数副本，避免了链接器在合并多个目标文件时因重复定义同名函数而报错。
- **优化性能**：编译器可能将这些函数内联展开，减少函数调用的开销。



## 一个例子

test.h:
```C
#include <stdio.h>
static inline void func1() {
    printf("this is func1: static inline void func1()\n");
}

```

test2.c:
```C
#include "test.h"
int func2() {
    func1();
    printf("this is func2: int func2() \n");
    return 0;
}

```

test3.c:
```C
#include "test.h"

int func3() {
    func1();
    printf("this is func3: int func3() \n");
    return 0;
}

```

main.c:

```C
#include <stdio.h>
int main() {
    printf("this is main: int main()\n");
    func2();
    func3();
    return 0;
}
```

Makefile:

```Makefile
main: main.o test2.o test3.o
	gcc -o main main.o test2.o test3.o

clean:
	rm main.o test2.o test3.o main

```

`{static(Y) inline(Y)]、[static(Y) inline(N)}`这两种情况下都可以编译成功，这很正常：

```bash
$ make
cc    -c -o main.o main.c
main.c: In function ‘main’:
main.c:4:5: warning: implicit declaration of function ‘func2’ [-Wimplicit-function-declaration]
    4 |     func2();
      |     ^~~~~
main.c:5:5: warning: implicit declaration of function ‘func3’ [-Wimplicit-function-declaration]
    5 |     func3();
      |     ^~~~~
cc    -c -o test2.o test2.c
cc    -c -o test3.o test3.c
gcc -o main main.o test2.o test3.o

luyoung at luyoung-desktop in ~/Test/test13
$ nm test2.o | grep func1
0000000000000000 t func1

luyoung at luyoung-desktop in ~/Test/test13
$ nm test3.o | grep func1
0000000000000000 t func1

luyoung at luyoung-desktop in ~/Test/test13
$ nm main | grep func1
000000000000117b t func1
00000000000011b9 t func1
```

可以看到，虽然是 inline，但是编译器没有听从建议，全部用的是内部链接。

`{static(N) inline(Y)]`这两种情况下编译不成功了：

```bash
make
cc    -c -o main.o main.c
main.c: In function ‘main’:
main.c:4:5: warning: implicit declaration of function ‘func2’ [-Wimplicit-function-declaration]
    4 |     func2();
      |     ^~~~~
main.c:5:5: warning: implicit declaration of function ‘func3’ [-Wimplicit-function-declaration]
    5 |     func3();
      |     ^~~~~
cc    -c -o test2.o test2.c
cc    -c -o test3.o test3.c
gcc -o main main.o test2.o test3.o
/usr/bin/ld: test2.o: in function `func2':
test2.c:(.text+0xe): undefined reference to `func1'
/usr/bin/ld: test3.o: in function `func3':
test3.c:(.text+0xe): undefined reference to `func1'
collect2: error: ld returned 1 exit status
make: *** [Makefile:2: main] Error 1

luyoung at luyoung-desktop in ~/Test/test13
$ nm test2.o | grep func1
                 U func1

luyoung at luyoung-desktop in ~/Test/test13
$ nm test3.o | grep func1
                 U func1
```

可以看到它报的错误是 `undefined reference to func1`，并且 `test2.o`、`test3.o` 中有未定义的 `func1`，这是为什么？这是因为编译器期望存在一个外部定义，但是并没有外部定义，因此C99要求：inline 函数具有外部链接，但需要在某个翻译单元中提供一个非 inline 的声明，供链接器使用。比如：

```C
#include "test.h"
void func1();
int func3() {
    func1();
    printf("this is func3: int func3() \n");
    return 0;
}

```

就可以通过编译。

另外，如果强制内联，那么内联函数就自然不存在什么外部链接了，直接在预处理阶段就处理好了:

```C
#include <stdio.h>
inline __attribute__((always_inline)) void func1() {
    printf("this is func1: static inline void func1()\n");
}

```

或者，直接在变异的时候，最大概率内联（不一定会内联，但是概率较大，最好的方法就是强制内联）：

```Makefile
test2.o: test2.c
	gcc -O3 -finline-functions -c test2.c -o test2.o
test3.o: test3.c
	gcc -O3 -finline-functions -c test3.c -o test3.o
```

## 总结

### **1. 函数的链接属性**

在C语言中，函数有两种主要的链接属性：

- **内部链接（Internal Linkage）**：函数只能在定义它的源文件内部使用。使用 `static` 关键字可以实现这一点。

- **外部链接（External Linkage）**：函数可以被整个程序中的其他源文件访问。默认情况下，函数具有外部链接，除非使用 `static` 限制其作用域。

### **2. `inline` 关键字的作用**

`inline` 关键字是对编译器的一个建议，表示希望将函数调用替换为函数体本身，以减少函数调用的开销，提高执行效率。然而，`inline` 并不保证编译器一定会内联所有的函数调用。编译器会根据函数的复杂性、调用频率以及优化级别等因素决定是否内联。

### **3. 仅移除 `static`，保留 `inline` 的影响**

当您在头文件中定义一个函数并仅移除 `static`，保留 `inline`，如下：

```c
inline uint32_t inst_fetch(vaddr_t *pc, int len) {
    // 函数体
}
```

这会导致以下情况发生：

1. **外部链接**：函数 `inst_fetch` 现在具有外部链接，这意味着它在整个程序中都是一个全局符号，可以被其他源文件引用。

2. **多重定义的风险**：如果多个源文件包含这个头文件，每个源文件都会尝试定义一个名为 `inst_fetch` 的全局符号。这违反了C语言的“一定义规则”（One Definition Rule），即每个全局符号只能有一个定义。

3. **编译器的内联行为**：
   - **部分内联**：编译器可能会在某些调用点内联 `inst_fetch`，将函数体直接插入到调用处，从而不生成对该函数的外部调用。
   - **未内联的调用**：对于未被内联的调用，编译器会生成对 `inst_fetch` 的函数调用指令，期望在链接阶段找到该函数的定义。

4. **链接器的期望**：
   - 链接器在处理这些调用时，期望在某处找到 `inst_fetch` 的唯一定义。
   - 由于头文件中的 `inline` 定义被多个源文件包含，并且没有提供一个统一的外部定义，链接器无法找到一个确定的 `inst_fetch` 实现，这也是为什么 C99 要求必须提供一个外部实现或者定义。
   - 结果，链接器报告“未定义引用”（undefined reference）错误，因为它找不到函数的实际实现。

### **4. 为什么会看到未定义引用**

尽管 `inline` 关键字建议编译器内联展开函数，编译器并不保证所有的调用都会被内联。对于那些未被内联的调用，编译器生成了对 `inst_fetch` 的外部调用指令，但由于缺乏一个统一的外部定义，链接器无法解析这些调用，导致未定义引用错误。

### **5. `nm` 工具显示未定义符号的原因**

当使用 `nm` 工具查看对象文件时，看到类似以下的输出：

```
U func1
```

这里的 `U` 表示“未定义”（Undefined）符号，意味着这个对象文件引用了 `func1`，但没有在该文件中找到它的定义。具体原因如下：

- **未内联的调用**：某些调用点未被内联展开，编译器生成了对 `func1` 的调用，但并未在当前源文件中提供其实现。
- **缺乏统一的外部定义**：由于头文件中定义的 `inline` 函数被多个源文件包含，且没有一个源文件提供一个非 `inline` 的外部定义，导致链接器找不到 `func1` 的实际实现。

### **6. 为什么同时去掉 `static` 和 `inline` 会导致多重定义错误**

当同时去掉 `static` 和 `inline`，函数定义变为一个普通的全局函数，具有外部链接：

```c
uint32_t inst_fetch(vaddr_t *pc, int len) {
    // 函数体
}
```

这种情况下：

1. **外部链接**：函数 `inst_fetch` 具有外部链接，可以被整个程序中的其他源文件访问。
2. **多重定义**：多个源文件包含这个头文件，每个源文件都会生成一个名为 `inst_fetch` 的全局符号。
3. **链接器冲突**：链接器在合并这些目标文件时，发现多个同名的全局符号，导致多重定义错误。

### **7. 为什么仅移除 `static` 会导致未定义引用，而去掉 `static` 和 `inline` 会导致多重定义**

- **仅移除 `static`，保留 `inline`**：
  - **部分内联**：编译器可能内联一些调用，但未内联的调用需要一个外部定义。
  - **缺少外部定义**：由于没有提供一个统一的外部定义，链接器无法找到 `inst_fetch`，导致未定义引用错误。

- **同时去掉 `static` 和 `inline`**：
  - **全部外部定义**：每个包含头文件的源文件都定义了一个全局的 `inst_fetch`，导致链接器发现多个定义，报出多重定义错误。



## 声明和定义：static


**第一问：**

在 `nemu/include/common.h` 中添加一行 `volatile static int dummy;` 后重新编译 NEMU。由于 `common.h` 被多个源文件包含，而 `static` 关键字在文件作用域内使变量具有内部链接，每个源文件都会有自己独立的 `dummy` 变量。

**因此，NEMU 中的 `dummy` 变量实体数量等于包含了 `common.h` 的源文件数量。**

---

**第二问：**

在 `nemu/include/debug.h` 中也添加一行 `volatile static int dummy;`，然后重新编译 NEMU。

**比较和解释：**

- **实体数量：** NEMU 中的 `dummy` 变量实体数量仍然等于源文件的数量，因为每个源文件中，`dummy` 变量只会被定义一次。

- **原因：** 即使某个源文件同时包含了 `common.h` 和 `debug.h`，由于两个头文件中对 `dummy` 的声明都是 `volatile static int dummy;`，这在 C 语言中属于多次声明同一个具有内部链接的变量，是被允许的。

- **结论：** 与第一问相比，`dummy` 变量的实体数量不变，仍然等于源文件的数量。

---

**第三问：**

将两处 `dummy` 变量修改为初始化形式：`volatile static int dummy = 0;`，然后重新编译 NEMU。

**发现的问题：**

- 编译器会在包含了 **同时包含 `common.h` 和 `debug.h`** 的源文件中报错，提示重复定义了 `dummy` 变量。

**原因分析：**

- **定义与声明的区别：** 在 C 语言中，未初始化的文件作用域的 `static` 变量（如 `static int dummy;`）被视为**声明**，可多次声明。但带有初始化的 `static` 变量（如 `static int dummy = 0;`）被视为**定义**，在同一作用域内只能定义一次。

- **重复定义错误：** 当一个源文件同时包含了 `common.h` 和 `debug.h`，并且这两个头文件中都对 `dummy` 进行了初始化定义，编译器会检测到在同一作用域内对具有内部链接的变量进行了多次定义，从而报错。

**为什么之前没有出现这样的问题？**

- **未初始化时的行为：** 未初始化的 `static` 变量在同一作用域内可以多次声明，编译器会将其视为对同一变量的多次声明，不会报错。

- **初始化后的行为：** 一旦对 `static` 变量进行了初始化，就变成了定义。在同一作用域内多次定义同名的具有内部链接的变量是非法的，编译器会报重复定义错误。

---

**总结：**

- 在头文件中声明未初始化的 `static` 变量是安全的，即使多个头文件中有相同的声明，编译器也不会报错。

- 但在头文件中定义（初始化）具有内部链接的 `static` 变量时，需要确保这些头文件不会被同一源文件多次包含，否则会导致重复定义错误。

- **建议：** 为避免此类问题，最好避免在头文件中对 `static` 变量进行初始化。如果需要初始化，应该在源文件中进行，或者采取其他设计方式。



