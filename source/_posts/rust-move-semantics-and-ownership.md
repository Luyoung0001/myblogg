---
title: "Rust 所有权笔记：移动语义与资源转移"
date: 2025-04-07 15:18:27
tags:
  - "Rust"
  - "ownership"
  - "move semantics"
categories:
  - "Programming Languages"
---

## 移动特性

C++11 会通过移动语义来消除内存拷贝成本，而 rust 将这种特性发挥到极致，推出了所有权机制。

看一段代码：

```Rust
let s1 = String::from("hello");
let s2 = s1;
```

当 s2 绑定 s1 的资源的时候，就会将 s1 的资源转移到 s2 中，这是因为一份资源只能有一个拥有者。

换句话说，s1 资源转移到 s2 中，不仅仅发生了浅拷贝，而且 s1 变量也无效了。这个和 C++ 还不一样：

```C++
#include <iostream>
#include <utility> // std::move

class ResourceHolder {
public:
    int* data;

    ResourceHolder(int value) : data(new int(value)) {}

    // 移动构造函数
    ResourceHolder(ResourceHolder&& other) noexcept : data(other.data) {
        other.data = nullptr; // 将原对象的指针置空
    }

    ~ResourceHolder() {
        delete data; // 安全：delete nullptr 是允许的
    }
};

int main() {
    ResourceHolder obj1(42);
    ResourceHolder obj2 = std::move(obj1); // 移动构造

    if (obj1.data == nullptr) {
        std::cout << "obj1.data 已被置空" << std::endl; // 输出此句
    }

    // 危险操作：访问已被移动的 obj1.data
    // std::cout << *obj1.data << std::endl; // 未定义行为（可能崩溃）

    return 0;
}
```

C++ 的对象被移动之后，原有对象并没有因此失效，只是资源不可访问。而 Rust 对象是彻底的实效。

## 克隆（深拷贝）

Rust 永远没有所谓的深拷贝，只能通过 clone  来实现资源在堆上的复制：

```Rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("s1 = {}, s2 = {}", s1, s2);
```

## 浅拷贝

前面说的资源的转移，就附带了浅拷贝，浅拷贝发生在栈上。带有 Copy 特征的对象都可以在栈上浅拷贝：
- 所有整数类型，比如 u32
- 布尔类型，bool，它的值是 true 和 false
- 所有浮点数类型，比如 f64
- 字符类型，char
- 元组，当且仅当其包含的类型也都是 Copy 的时候。比如，(i32, i32) 是 Copy 的，但 (i32, String) 就不是
- 不可变引用 &T ，例如转移所有权中的最后一个例子，但是注意：可变引用 &mut T 是不可以 Copy的

## 函数

函数是一个表达式，它有返回值：

```Rust
fn main() {
    let s = String::from("hello");  // s 进入作用域

    takes_ownership(s);             // s 的值移动到函数里 ...
                                    // ... 所以到这里不再有效

    let x = 5;                      // x 进入作用域

    makes_copy(x);                  // x 应该移动函数里，
                                    // 但 i32 是 Copy 的，所以在后面可继续使用 x

} // 这里, x 先移出了作用域，然后是 s。但因为 s 的值已被移走，
  // 所以不会有特殊操作

fn takes_ownership(some_string: String) { // some_string 进入作用域
    println!("{}", some_string);
} // 这里，some_string 移出作用域并调用 `drop` 方法。占用的内存被释放

fn makes_copy(some_integer: i32) { // some_integer 进入作用域
    println!("{}", some_integer);
} // 这里，some_integer 移出作用域。不会有特殊操作
```

同样，函数的返回值也有所有权：

```Rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership 将返回值
                                        // 移给 s1

    let s2 = String::from("hello");     // s2 进入作用域

    let s3 = takes_and_gives_back(s2);  // s2 被移动到
                                        // takes_and_gives_back 中,
                                        // 它也将返回值移给 s3
} // 这里, s3 移出作用域并被丢弃。s2 也移出作用域，但已被移走，
  // 所以什么也不会发生。s1 移出作用域并被丢弃

fn gives_ownership() -> String {             // gives_ownership 将返回值移动给
                                             // 调用它的函数

    let some_string = String::from("hello"); // some_string 进入作用域.

    some_string                              // 返回 some_string 并移出给调用的函数
}

// takes_and_gives_back 将传入字符串并返回该值
fn takes_and_gives_back(a_string: String) -> String { // a_string 进入作用域

    a_string  // 返回 a_string 并移出给调用的函数
}
```

## 悬垂引用(Dangling References)

考虑一个问题，如果引用 s 的资源没了，怎么办？先来看看这个简单的 C 语言例子：

```C
int* foo() {
    int a;          // 变量a的作用域开始
    a = 100;
    char *c = "xyz";   // 变量c的作用域开始
    return &a;
}                   // 变量a和c的作用域结束

```

这个函数返回后，a 被释放了，因此返回的值带来不确定性。从根本上讲，虽然不会报错，但是这是典型的非法例子。

再考虑类似的情景：

```Rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String {
    let s = String::from("hello");

    &s
}
```

当 s 离开作用域之后资源被释放了，那么 s 的引用自然就没有意义了。Rust 很严格，编译直接报错，不像 C 那样危险。


