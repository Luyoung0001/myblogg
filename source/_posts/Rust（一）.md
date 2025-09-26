---
title: Rust:struct、enum、string、module
date: 2025-09-26 13:03:13
tags: Rust 基础
categories: 编程语言
---

## 前言

这是第二次学习 Rust，上一次学习到一半就去忙别的事情了，半年过去了，感觉忘得差不多了（不用就会忘）。为了防止再次忘记，这次一边学习一边记录。主要基于 rustlings 来学习。


## struct

struct 具体包括：

- Classic Structs（经典结构体）
- Tuple Structs（元组结构体）
- Unit-like Structs（类似单元的结构体）


### Classic Structs (经典结构体)

经典结构体是 Rust 中最常用的一种数据结构，通常由一个或多个**具名字段** 组成。它的定义方式和使用方式类似于传统的面向对象编程中的类，但它是专门用于组织数据。

经典结构：
```rust
struct ColorClassicStruct {
    red: u8,
    green: u8,
    blue: u8,
}
```
创建和实例化经典结构体：

```rust
let green = ColorClassicStruct {
    red: 0,
    green: 255,
    blue: 0,
};
```

### Tuple Structs (元组结构体)

元组结构体是 Rust 中的一种特殊结构体，它没有命名的字段，而是仅仅使用位置来组织数据。这类似于元组（Tuple），但元组结构体定义了自己的类型。

定义元组结构体：
```rust
struct ColorTupleStruct(u8, u8, u8);
```

创建和实例化元组结构体：
```rust
let green = ColorTupleStruct(0, 255, 0);
```


访问元组结构体的字段时，你使用数字索引来进行访问：
```rust
assert_eq!(green.0, 0);
assert_eq!(green.1, 255);
assert_eq!(green.2, 0);
```

### Unit-like Structs (类似单元的结构体)

单元结构体是一种没有任何字段的结构体。它通常用于标记某种类型、传递某些上下文信息或在特定情况下作为占位符。它在 Rust 中的作用类似于“单位”类型（()，即空元组）。

定义单元结构体：
```rust
struct UnitLikeStruct;
```

这里 UnitLikeStruct 没有任何字段，它只是一个占位符类型。它通常在某些情况下用作标记类型，例如用来表示状态、行为等。

创建和实例化单元结构体：
```rust
let unit_like_struct = UnitLikeStruct;
```

使用单元结构体：

由于它没有字段，单元结构体通常用于匹配和类型标识。例如，在测试中，你可以通过 format! 宏将其转为字符串并打印出来。

```rust
let message = format!("{:?}s are fun!", unit_like_struct);
assert_eq!(message, "UnitLikeStructs are fun!");
```

### 例子_1:

```rust
// structs1.rs

struct ColorClassicStruct {
    // TODO: Something goes here
    red: u8,
    green: u8,
    blue: u8,
}

struct ColorTupleStruct(u8,u8,u8);

#[derive(Debug)]
struct UnitLikeStruct;

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn classic_c_structs() {
        // TODO: Instantiate a classic c struct!
        let green = ColorClassicStruct{
            red:0,
            green:255,
            blue:0,
        };

        assert_eq!(green.red, 0);
        assert_eq!(green.green, 255);
        assert_eq!(green.blue, 0);
    }

    #[test]
    fn tuple_structs() {
        // TODO: Instantiate a tuple struct!
        let green = ColorTupleStruct(0,255,0);

        assert_eq!(green.0, 0);
        assert_eq!(green.1, 255);
        assert_eq!(green.2, 0);
    }

    #[test]
    fn unit_structs() {
        // TODO: Instantiate a unit-like struct!
        let unit_like_struct = UnitLikeStruct;
        let message = format!("{:?}s are fun!", unit_like_struct);

        assert_eq!(message, "UnitLikeStructs are fun!");
    }
}

```

### 例子_2:

`#[derive(Debug)]` 是一个 Rust 属性宏（attribute macro），它自动为结构体生成一个实现了 Debug trait 的版本。Debug trait 允许结构体以调试格式输出。这是一个常用的技巧，帮助我们在开发和调试时，能够轻松打印结构体的内容。

更新语法中的 `..order_template` 是一个重要特性。它会把 `order_template` 中未明确指定的字段值“复制”到新结构体中，从而避免重复输入相同的字段值。

```rust
// structs2.rs

#[derive(Debug)]
struct Order {
    name: String,
    year: u32,
    made_by_phone: bool,
    made_by_mobile: bool,
    made_by_email: bool,
    item_number: u32,
    count: u32,
}

fn create_order_template() -> Order {
    Order {
        name: String::from("Bob"),
        year: 2019,
        made_by_phone: false,
        made_by_mobile: false,
        made_by_email: true,
        item_number: 123,
        count: 0,
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn your_order() {
        let order_template = create_order_template();
        // TODO: Create your own order using the update syntax and template above!
        let your_order = Order {
            name: String::from("Hacker in Rust"), // 更新 name 字段
            count: 1,                             // 更新 count 字段
            ..order_template                      // update syntax and template above
        };

        assert_eq!(your_order.name, "Hacker in Rust");
        assert_eq!(your_order.year, order_template.year);
        assert_eq!(your_order.made_by_phone, order_template.made_by_phone);
        assert_eq!(your_order.made_by_mobile, order_template.made_by_mobile);
        assert_eq!(your_order.made_by_email, order_template.made_by_email);
        assert_eq!(your_order.item_number, order_template.item_number);
        assert_eq!(your_order.count, 1);
    }
}

```

## enum

在 Rust 中，enum（枚举）是一种自定义类型，允许我们定义一组变体。每个变体可以具有不同的数据类型或者没有数据（像一个标志）。枚举在 Rust 中被广泛应用于表示可能的多种状态或事件。

比如：

```rust
#[derive(Debug)]
enum Message {
    Move {x:u8,y:u8},
    Echo(String),
    ChangeColor(u8, u8, u8),
    Quit,
}

impl Message {
    fn call(&self) {
        println!("{:?}", self);
    }
}

fn main() {
    let messages = [
        Message::Move { x: 10, y: 30 },
        Message::Echo(String::from("hello world")),
        Message::ChangeColor(200, 255, 255),
        Message::Quit,
    ];

    for message in &messages {
        message.call();
    }
}

```

使用的时候，需要带上 `Message::` 指明作用域。

再比如这个更复杂的例子：

```rust
// enums3.rs
enum Message {
    // TODO: implement the message variant types based on their usage below
    ChangeColor(u8, u8, u8),
    Echo(String),
    Move(Point),
    Quit,
}

struct Point {
    x: u8,
    y: u8,
}

struct State {
    color: (u8, u8, u8),
    position: Point,
    quit: bool,
    message: String,
}

impl State {
    fn change_color(&mut self, color: (u8, u8, u8)) {
        self.color = color;
    }

    fn quit(&mut self) {
        self.quit = true;
    }

    fn echo(&mut self, s: String) {
        self.message = s
    }

    fn move_position(&mut self, p: Point) {
        self.position = p;
    }

    fn process(&mut self, message: Message) {
        // TODO: create a match expression to process the different message
        // variants
        // Remember: When passing a tuple as a function argument, you'll need
        // extra parentheses: fn function((t, u, p, l, e))
        match message {
            Message::ChangeColor(r, g, b) => self.change_color((r, g, b)),
            Message::Echo(s) => self.echo(s),
            Message::Move(o) => self.move_position(o),
            Message::Quit => self.quit(),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_match_message_call() {
        let mut state = State {
            quit: false,
            position: Point { x: 0, y: 0 },
            color: (0, 0, 0),
            message: "hello world".to_string(),
        };
        state.process(Message::ChangeColor(255, 0, 255));
        state.process(Message::Echo(String::from("hello world")));
        state.process(Message::Move(Point { x: 10, y: 15 }));
        state.process(Message::Quit);

        assert_eq!(state.color, (255, 0, 255));
        assert_eq!(state.position.x, 10);
        assert_eq!(state.position.y, 15);
        assert_eq!(state.quit, true);
        assert_eq!(state.message, "hello world");
    }
}

```

## string

在 Rust 中，**字符串切片**（`&str`）和 **字符串**（`String`）是两种常用的字符串类型，它们在内存存储、生命周期管理和可变性等方面有所不同。下面是它们的主要区别：

### 1. **类型定义**

* **字符串切片（`&str`）**：是对字符串的一部分的引用，通常是不可变的，指向某个已有的字符串数据。它通常以字面量的形式出现，如 `"hello"`，并且它是不可变的。

  ```rust
  let s: &str = "hello";  // 字符串切片类型
  ```

* **字符串（`String`）**：是一个可变的、堆分配的字符串类型，通常用于在运行时构建和修改字符串。它的内容存储在堆上，可以在程序中动态变化。

  ```rust
  let mut s: String = String::from("hello");  // 字符串类型
  ```

### 2. **内存分配**

* **字符串切片（`&str`）**：通常是指向程序中某个常量字符串的引用。它的内存存储在程序的**只读数据段**（比如编译时已知的字符串字面量）。`&str` 是一个引用类型，它指向某个已经存在的字符串数据。

  * 字符串切片不需要分配堆内存，因此它通常比 `String` 类型更轻量。
  * 你不能改变字符串切片的内容，因为它是不可变的。

* **字符串（`String`）**：`String` 类型是一个动态分配的堆数据结构。它可以在程序运行时改变内容，增加、删除字符等操作。

  * 当你创建一个 `String` 类型时，它会在堆上分配内存，存储实际的字符串数据。
  * `String` 类型是可变的，可以在程序运行时修改它的内容。

### 3. **可变性**

* **字符串切片（`&str`）**：是不可变的，不能直接修改它的内容。如果你需要修改内容，就需要使用 `String` 类型。

  ```rust
  let s: &str = "hello";
  // s.push_str(" world");  // 错误：不能修改 &str 的内容
  ```

* **字符串（`String`）**：是可变的，你可以修改它的内容，比如向其中追加字符、删除字符等。

  ```rust
  let mut s: String = String::from("hello");
  s.push_str(" world");  // 可以修改 String 的内容
  println!("{}", s);  // 输出 "hello world"
  ```

### 4. **生命周期**

* **字符串切片（`&str`）**：是对某个字符串的引用，因此它有一个生命周期，生命周期由它引用的字符串决定。如果字符串的生命周期结束，字符串切片也无法再使用。

  ```rust
  let s: &str = "hello";  // 's' 是对 "hello" 的引用，生命周期受限于该字符串的存在
  ```

* **字符串（`String`）**：`String` 是拥有所有权的类型，它控制字符串数据的生命周期。当 `String` 被销毁时，它所包含的字符串数据也会被清理。

  ```rust
  let s: String = String::from("hello");  // 's' 拥有 "hello" 的所有权，生命周期结束时会自动清理
  ```

### 5. **转换**

* **从 `&str` 转换为 `String`**：可以通过 `to_string()` 或 `String::from()` 方法将字符串切片转换为 `String` 类型。

  ```rust
  let s: &str = "hello";
  let string_version: String = s.to_string();  // 将 &str 转换为 String
  ```

* **从 `String` 转换为 `&str`**：`String` 可以通过 `as_str()` 方法转换为 `&str`，这种转换是无需复制的，因为 `&str` 只是对字符串的引用。

  ```rust
  let s: String = String::from("hello");
  let slice_version: &str = s.as_str();  // 将 String 转换为 &str
  ```

### 6. **性能**

* **字符串切片（`&str`）**：由于它只是对已有字符串的引用，所以它非常高效，尤其是当你不需要修改字符串内容时。它不会占用额外的内存，也不会进行动态分配。

* **字符串（`String`）**：由于它是堆分配的，修改字符串时会涉及到内存重新分配和管理，因此 `String` 相对 `&str` 来说可能在某些场景下开销较大。

### 7. **使用场景**

* **字符串切片（`&str`）**：适用于你需要引用一个已经存在的字符串，并且不打算修改它。比如，函数参数传递时经常使用 `&str`，因为它不需要拷贝数据，且能直接引用已有的字符串。

* **字符串（`String`）**：适用于你需要动态创建、修改或存储字符串的场景。例如，当你在程序中动态地构建字符串或需要多次修改字符串内容时，`String` 是最合适的选择。

### 总结：

| 特性   | `&str`                         | `String`                  |
| ---- | ------------------------------ | ------------------------- |
| 类型   | 字符串切片（不可变引用）                   | 堆分配的可变字符串                 |
| 内存分配 | 引用已有的字符串数据，通常在栈上               | 堆分配，拥有自己的内存               |
| 可变性  | 不可变                            | 可变                        |
| 转换   | 使用 `.to_string()` 转换为 `String` | 使用 `.as_str()` 转换为 `&str` |
| 性能   | 较高效（无内存分配）                     | 需要堆分配和管理，可能有性能开销          |
| 使用场景 | 传递和引用已有的字符串（无需修改）              | 动态构建和修改字符串                |




当从 `&str` 转换为 `String`，或者从 `String` 转换为 `&str` 时，内存的管理和分配方式会有所不同。

### 1. **从 `&str` 转换为 `String`**

```rust
let s: &str = "hello";
let string_version: String = s.to_string();  // 将 &str 转换为 String
```

#### 内存变化：

* **`&str`** 是一个字符串切片，它是对已存在的字符串的引用，通常存储在程序的只读数据段中，或者是栈上。
* **`String`** 是一个堆分配的类型，它在堆上分配内存，并拥有它所包含的数据。转换为 `String` 后，数据会被从 `&str` 所引用的位置复制到新的堆内存中。

具体来说：

1. 当调用 `s.to_string()` 时，Rust 会为新的 `String` 分配堆内存。
2. 然后，它将 `&str` 中的字符串数据（即 `"hello"`）复制到 `String` 的堆内存中。
3. `String` 会管理它自己分配的堆内存，并且负责在不再需要时释放这部分内存。

因此，在这种转换中，会发生内存 **复制**，即 `&str` 中的数据被复制到新的堆内存中，`String` 拥有这块内存，并负责其生命周期。

#### 内存示意：

* **`&str`**：对 `"hello"` 字符串的引用，数据存储在程序的只读数据段。
* **`String`**：在堆上为 `"hello"` 分配新内存，并将其内容复制过去。

### 2. **从 `String` 转换为 `&str`**

```rust
let s: String = String::from("hello");
let slice_version: &str = s.as_str();  // 将 String 转换为 &str
```

#### 内存变化：

* **`String`** 是一个拥有字符串数据的堆分配类型，它在堆上分配内存来存储字符串数据。
* **`&str`** 是一个对已有字符串数据的引用，它并不拥有数据，而是借用它。

具体来说：

1. 调用 `s.as_str()` 时，Rust 返回一个对 `String` 中数据的引用（`&str`），而不会发生内存复制。
2. `&str` 只是借用 `String` 内部的内存，`String` 依然拥有数据的所有权，`&str` 只是对这些数据的借用。

因此，在这种转换中，并没有发生内存复制，`&str` 只是对 `String` 内部数据的引用。`String` 不会重新分配内存，`&str` 也不会拥有数据的所有权。

#### 内存示意：

* **`String`**：在堆上分配内存存储 `"hello"`。
* **`&str`**：借用 `String` 内部的内存，指向同样的 `"hello"` 数据，不会发生内存分配或复制。

### string or &str 有常用的三个方法

### 1. **`trim()` 方法**

```rust
input.trim()
```

* **`trim()`** 返回的是一个新的字符串切片（`&str`），**并不是一个新的字符串类型 `String`**。它去掉的是 `input` 字符串两端的空白字符，并返回一个引用，指向原始数据的切片。
* **为什么是引用**：`trim()` 返回的是对原始字符串数据的切片引用，这个引用本身并不拥有数据，而只是对原来数据的一个借用。因此，`trim()` 返回的是一个 **新的字符串切片**（`&str`），它指向原始字符串数据。
* **结论**：`trim()` 的返回值是一个 `&str`，它仍然借用原始数据，因此没有发生堆内存分配。

### 2. **`to_string()` 与 `+` 操作符**

```rust
input.to_string() + " world!"
```

* **`to_string()`** 会将字符串切片（`&str`）转换为一个 **新的 `String` 类型**，它会在堆上分配内存，并将 `input` 的内容复制到新的 `String` 中。
* **`+` 操作符**：`+` 被重载为一种字符串连接操作。它会将左侧的 `String` 和右侧的字符串（如 `" world!"`）拼接成一个新的 `String` 类型。这个操作会涉及到分配新的堆内存，并将两个字符串的内容复制到这个新的 `String` 中。

**总结**：

* `to_string()` 会分配新的堆内存并返回 `String`。
* `+` 操作符会在堆上分配新的内存，返回一个新的 `String`，它是由原始字符串和 `" world!"` 拼接而成。

### 3. **`replace()` 方法**

```rust
input.replace("cars", "balloons")
```

* **`replace()`** 是一个类似的方法，它会返回一个新的 `String`，并且会在新分配的内存中保存替换后的结果。`replace` 方法会遍历原始字符串，查找匹配的子字符串，然后构造一个新的字符串，替换其中的内容，并将结果存储在新分配的堆内存中。
* `replace()` 需要重新分配内存，因为它要生成一个新的字符串。

### 例子

```rust
// strings3.rs

fn trim_me(input: &str) -> String {
    // TODO: Remove whitespace from both ends of a string!
    input.trim().to_string()  // 去除两端空白字符，并转换为 String 类型
}

fn compose_me(input: &str) -> String {
    // TODO: Add " world!" to the string! There's multiple ways to do this!
    input.to_string() + " world!"
}

fn replace_me(input: &str) -> String {
    // TODO: Replace "cars" in the string with "balloons"!
    input.replace("cars", "balloons")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn trim_a_string() {
        assert_eq!(trim_me("Hello!     "), "Hello!");
        assert_eq!(trim_me("  What's up!"), "What's up!");
        assert_eq!(trim_me("   Hola!  "), "Hola!");
    }

    #[test]
    fn compose_a_string() {
        assert_eq!(compose_me("Hello"), "Hello world!");
        assert_eq!(compose_me("Goodbye"), "Goodbye world!");
    }

    #[test]
    fn replace_a_string() {
        assert_eq!(replace_me("I think cars are cool"), "I think balloons are cool");
        assert_eq!(replace_me("I love to look at cars"), "I love to look at balloons");
    }
}

```

## mod

在 Rust 中，**模块（`mod`）** 用来组织代码。模块可以包含函数、结构体、枚举、常量等。它有助于将代码分成更小、更有组织的部分。

* `mod sausage_factory` 定义了一个名为 `sausage_factory` 的模块。
* 模块中的两个函数：

  * **`get_secret_recipe()`**：这个函数返回一个字符串 `"Ginger"`，这是香肠的秘密配方。
  * **`make_sausage()`**：这个函数调用 `get_secret_recipe()` 获取秘密配方，并打印 `"sausage!"`。

### 2. **隐私和访问控制**

* **私有函数**：模块中的函数默认是 **私有的**（`private`），也就是说它们只能在该模块内部被访问。`get_secret_recipe()` 和 `make_sausage()` 函数没有加任何访问修饰符，因此它们在 `sausage_factory` 模块内是可见的，但在模块外不可访问。
* `get_secret_recipe()` 是一个私有函数，它无法直接在模块外被调用。
* `make_sausage()` 函数在 `sausage_factory` 模块内调用了 `get_secret_recipe()`，但是 `main` 函数无法直接访问 `get_secret_recipe()`。

### 3. **在 `main` 函数中访问模块**

```rust
fn main() {
    sausage_factory::make_sausage();
}
```

* `main` 函数尝试访问 `sausage_factory::make_sausage()`，这说明 `make_sausage()` 是公开的（`public`）。但 `get_secret_recipe()` 是私有的，不能在模块外部访问。
* **如何使函数公开**：如果我们希望 `get_secret_recipe()` 在外部也能被访问，需要将它声明为 `pub`，如：

  ```rust
  pub fn get_secret_recipe() -> String {
      String::from("Ginger")
  }
  ```

  这样 `get_secret_recipe()` 就变成了公共函数，可以在模块外部访问。

### 4. **访问模块中的函数**

当前代码中的访问方式是合法的，因为 `make_sausage()` 函数是私有的，只要它在模块内部调用 `get_secret_recipe()`，就没有问题。 `main` 函数通过 `sausage_factory::make_sausage()` 访问了 `make_sausage()`，这在 Rust 中是允许的。

### use and as

使用 use 可以引入模块的内容，使用 as 可以为引入的模块成员提供别名。

比如：
```rust
// modules2.rs
//
// You can bring module paths into scopes and provide new names for them with
// the 'use' and 'as' keywords. Fix these 'use' statements to make the code

mod delicious_snacks {
    // TODO: Fix these use statements
    pub use self::fruits::PEAR as fruit;
    pub use self::veggies::CUCUMBER as veggie;

    mod fruits {
        pub const PEAR: &'static str = "Pear";
        pub const APPLE: &'static str = "Apple";
    }

    mod veggies {
        pub const CUCUMBER: &'static str = "Cucumber";
        pub const CARROT: &'static str = "Carrot";
    }
}

fn main() {
    println!(
        "favorite snacks: {} and {}",
        delicious_snacks::fruit,
        delicious_snacks::veggie
    );
}
```















