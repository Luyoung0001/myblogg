---
title: Rust 中的 traits
date: 2025-02-18 15:18:27
tags:
	- Rust
	- Java
categories:
    - 编程语言
    - Rust
---

#### 最近笔者在学 Rust，被 Rust 中精巧的设计深深吸引，尤其是 traits。它不仅能够应用到 Struct、Enum等，而且还能作为参数传入函数。

## 应用于结构体

先看这个例子：

```Rust
trait Shape {
    fn area(&self) -> f64; // 计算面积
    fn perimeter(&self) -> f64; // 计算周长
}

struct Circle {
    radius: f64,
}

struct Rectangle {
    width: f64,
    height: f64,
}

// 为 Circle 结构体实现 Shape 特征
impl Shape for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }

    fn perimeter(&self) -> f64 {
        2.0 * std::f64::consts::PI * self.radius
    }
}

// 为 Rectangle 结构体实现 Shape 特征
impl Shape for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }

    fn perimeter(&self) -> f64 {
        2.0 * (self.width + self.height)
    }
}

fn main() {
    let circle = Circle { radius: 5.0 };
    let rectangle = Rectangle { width: 4.0, height: 6.0 };

    println!("Circle Area: {}, Perimeter: {}", circle.area(), circle.perimeter());
    println!("Rectangle Area: {}, Perimeter: {}", rectangle.area(), rectangle.perimeter());
}
```

这个例子是一个很简单的例子，就是将 trait 应用在具体的 Struct 上。类似的还有Enum。如果这就是 traits 的全部，那么 traits 将不值一提。

## 作为函数参数

看这个例子：

```Rust
trait Shape {
    fn area(&self) -> f64;
    fn perimeter(&self) -> f64;
}

// ... (Circle 和 Rectangle 结构体以及 Shape 特征的实现，与前面的例子相同) ...

// 泛型函数，接受任何实现了 Shape 特征的类型 T
fn print_shape_info<T: Shape>(shape: &T) {
    println!("Area: {}, Perimeter: {}", shape.area(), shape.perimeter());
}

fn main() {
    let circle = Circle { radius: 5.0 };
    let rectangle = Rectangle { width: 4.0, height: 6.0 };

    println!("Circle:");
    print_shape_info(&circle); // 传入 Circle 实例

    println!("Rectangle:");
    print_shape_info(&rectangle); // 传入 Rectangle 实例
}
```

这里的函数 `print_shape_info` 接受一个泛型，但是函数签名对这个泛型做出了限制`<T: Shape>`：要求这个泛型实现了Shape traits。

这意味着，随便定义一个类型，只要给这个类型加一个特征，比如 Display，那么这个类型就能使用所有可以接受 Display 的方法。

比如：

```Rust
use std::fmt; // 需要引入 fmt 模块才能使用 Display trait

#[derive(Debug)] // 为了方便打印 Point 结构体
struct Point {
    x: i32,
    y: i32,
}

// 为 Point 结构体实现 Display 特征
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y) // 定义 Point 的 Display 格式
    }
}

fn main() {
    let p = Point { x: 10, y: 20 };

    // 因为 Point 实现了 Display 特征，所以可以使用 println! 宏的 "{}" 格式化
    println!("Point p: {}", p); // 输出: Point p: (10, 20)

    // 也可以将 Point 传递给任何接受 Display trait 的函数或宏
    let message = format!("The point is: {}", p);
    println!("{}", message); // 输出: The point is: The point is: (10, 20)
}
```

甚至，在这个例子中，实现了 `Display` 的类型 自动 实现了 `ToString` 特征，而 `to_string()` 方法是由 `ToString` 特征提供的。

`ToString` 的自动实现 (Blanket Implementation):  Rust 标准库为所有实现了 `Display` 特征的类型，自动提供了一个 `ToString` 特征的实现 (blanket implementation)。  这个 blanket implementation 大致是这样的 (简化版)：

```Rust
impl<T: fmt::Display> ToString for T {
    fn to_string(&self) -> String {
        format!("{}", self) // 内部使用 Display 特征进行格式化
    }
}
```
