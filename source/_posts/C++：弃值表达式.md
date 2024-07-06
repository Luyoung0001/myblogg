---
title: C++：弃值表达式
date: 2024-04-11 20:41:12
tags: c++
categories: C++
---

<!--more-->

## 正文
有时候需要利用某些表达式的副作用来实现某些目的：

```cpp
#include <iostream>
template <typename... Args>
void print(const Args &...args) {
    Arr{0,(std::cout << args<< ' ',0)...};
}
int main() {
    print("hello", 1, 2, 3, 4, 5, 6);
    print();
    return 0;
}
```
但这会产生警告：

```bash
multi2.cxx:7:5: warning: expression result unused [-Wunused-value]
    Arr{0,(std::cout << args<< ' ',0)...};
    ^  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
multi2.cxx:10:5: note: in instantiation of function template specialization 'print<char [6], int, int, int, int, int, int>' requested here
    print("hello", 1, 2, 3, 4, 5, 6);
    ^
multi2.cxx:7:5: warning: expression result unused [-Wunused-value]
    Arr{0,(std::cout << args<< ' ',0)...};
    ^  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
multi2.cxx:11:5: note: in instantiation of function template specialization 'print<>' requested here
    print();
    ^
2 warnings generated.
```
为了消除警告，可以用关键字 `void`明确表示这是一个弃值表达式，即：

```cpp
#include <iostream>
template <typename... Args>
void print(const Args &...args) {
    using Arr = int[];
    (void)Arr{0,(std::cout << args<< ' ',0)...};
}
int main() {
    print("hello", 1, 2, 3, 4, 5, 6);
    print();
    return 0;
}
```

关于 void 表明弃值表达式的使用，实际上并非 void 本身表示弃值表达式，而是在某些上下文中使用 void 来表明该表达式的返回值被丢弃。例如，在函数声明中，**如果函数的返回类型是 void，那么调用该函数就意味着对返回值的忽略，从而可以被视为一种弃值操作。**

比如：

```cpp
#include <iostream>
int getValue() {
    std::cout << "Getting value..." << std::endl;
    return 42;
}

int main() {

    (void)getValue();
    // 使用弃值表达式,利用副作用
    int x = 10;
    (void)((x > 5) ? (std::cout << "hello1\n", 0)
                   : (std::cout << "hello2\n", 0));

    return 0;
}
```
上面的 `getValue()`确实有一个返回值，但是我们调用它的时候并不需要这个返回值，因此可以用关键字 `void` 表明我们放弃了这个值，并告诉编译器不要产生警告。另外，根据条件 `x > 5` 的结果，选择输出 "hello1" 还是 "hello2"。无论选择哪个分支，最终都返回 0，这个返回值在这里被忽略，只关注分支执行时的副作用（输出文本）。







