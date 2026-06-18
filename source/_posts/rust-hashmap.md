---
title: "Rust 学习笔记（二）：HashMap 基础与实践"
date: 2025-09-26 13:03:13
tags:
  - "Rust"
  - "HashMap"
  - "collections"
categories:
  - "Programming Languages"
---

## HashMap

Rust 的 `HashMap` 是一个基于哈希表的键值对集合，它提供了高效的数据查找、插入和删除操作（平均时间复杂度 O(1)）。它是 Rust 标准库 `std::collections` 的一部分，使用前需要引入。

---

### 核心概念
1. **键值对 (Key-Value Pair)**： 每个元素由一个唯一的 `key` 和对应的 `value` 组成。
2. **所有权 (Ownership)**：
   - 插入时：`HashMap` 会取得键值对的所有权（对于实现了 `Copy` trait 的类型如 `i32` 会复制）。
   - 查询时：通常返回对值的引用（`Option<&V>`），避免所有权转移。
3. **哈希与相等性**：
   - 键的类型 `K` 必须实现 `Eq`（判断相等）和 `Hash`（计算哈希值）两个 trait。
   - 基础类型（如 `String`、`i32` 等）已自动实现，自定义类型可通过 `#[derive(PartialEq, Eq, Hash)]` 实现。

---

### 基础用法与实例

#### 1. 创建 HashMap
```rust
use std::collections::HashMap;

// 创建一个空 HashMap (需指定类型)
let mut scores: HashMap<String, i32> = HashMap::new();

// 使用迭代器和 collect 创建
let teams = vec!["Blue", "Red"];
let initial_scores = vec![10, 50];
let scores: HashMap<_, _> = teams.iter().zip(initial_scores.iter()).collect();
```

#### 2. 插入/更新数据
```rust
let mut map = HashMap::new();

// 插入数据
map.insert(String::from("Blue"), 10); // key: "Blue", value: 10
map.insert(String::from("Red"), 50);

// 更新数据（如果存在则覆盖）
map.insert(String::from("Blue"), 25); // 现在 "Blue" 的值是 25

// 只在键不存在时插入并且返回Value，如果存在返回 Value
map.entry(String::from("Yellow")).or_insert(50); // 插入
map.entry(String::from("Blue")).or_insert(50);    // 不会更新（因为已存在）
```

#### 3. 查询数据
```rust
let team_name = String::from("Blue");

// get() 返回 Option<&V>
match map.get(&team_name) {
    Some(score) => println!("{}: {}", team_name, score),
    None => println!("Team not found"),
}

// 遍历所有键值对
for (key, value) in &map {
    println!("{}: {}", key, value);
}
```

#### 4. 更新值（基于旧值）
```rust
let text = "hello world wonderful world";

let mut word_count = HashMap::new();

for word in text.split_whitespace() {
    // or_insert() 返回键值的可变引用 &mut V
    // 这同样是为了避免转移所有权
    let count = word_count.entry(word).or_insert(0);
    *count += 1; // 通过解引用修改值
}

println!("{:?}", word_count);
// 输出: {"world": 2, "hello": 1, "wonderful": 1}
```

#### 5. 删除数据
```rust
// 移除键值对
map.remove(&String::from("Red"));

// 清空 HashMap
map.clear();
```

---

### 常见场景与技巧

#### 1. 处理可能不存在的键
```rust
let score = map.get("Green").copied().unwrap_or(0); // 返回 0 如果不存在，这个和.entry 的区别是，.get 不会插入键值对
```

这个例子中，get() 返回一个 `Option<&T>`，copied() 将 `Option<&T>` 转变成 `Option<T>`，unwrap(）直接将 `Option<T>` 中的 T 取出来。

#### 2. 合并两个 HashMap
```rust
let mut map1 = HashMap::from([("a", 1), ("b", 2)]);
let map2 = HashMap::from([("b", 3), ("c", 4)]);

map1.extend(map2); // map2 的所有权被转移
// 结果: {"a": 1, "b": 3, "c": 4} (map2 的值覆盖相同键)
```

#### 3. 自定义键类型
```rust
#[derive(Debug, Hash, Eq, PartialEq)]
struct Student {
    id: u32,
    name: String,
}

let mut students = HashMap::new();
students.insert(
    Student { id: 1, name: "Alice".to_string() },
    "Grade A"
);

println!("{:?}", students.get(&Student { id: 1, name: "Alice".to_string() }));
```

---

### 注意事项
1. **性能影响**：哈希冲突可能降低性能（Rust 默认使用抗碰撞的 SipHash 算法）。
2. **内存开销**：HashMap 会预分配内存以提升效率。
3. **线程安全**：默认非线程安全，多线程需用 `Mutex<HashMap>` 或 `RwLock<HashMap>`。

---

### 完整示例：单词计数器
```rust
use std::collections::HashMap;

fn main() {
    let text = "apple banana orange apple banana";
    let mut counter = HashMap::new();

    for word in text.split_whitespace() {
        *counter.entry(word).or_insert(0) += 1;
    }

    println!("Word counts:");
    for (word, count) in &counter {
        println!("{}: {}", word, count);
    }
}
// 输出:
// apple: 2
// banana: 2
// orange: 1
```

### 问题1：这个 unwrap() 取出来的 T 是拷贝还是引用？

`unwrap()` 返回的是 **值的所有权（owned value）**，而不是引用。

### `unwrap()` 的本质
```rust
// Option 的 unwrap 实现（简化版）
impl<T> Option<T> {
    pub fn unwrap(self) -> T {
        match self {
            Some(val) => val,  // 直接返回 T 本身
            None => panic!("called `Option::unwrap()` on a `None` value"),
        }
    }
}
```

### 所有权转移的三种情况

#### 1. 基础类型（实现 `Copy` trait）
```rust
let num: Option<i32> = Some(42);
let unwrapped_num = num.unwrap();  // 发生拷贝（因为 i32 实现了 Copy）

println!("{}", unwrapped_num); // 42
// 可以继续使用 num，因为 Option<i32> 也实现了 Copy
```

#### 2. 堆分配类型（未实现 `Copy`）
```rust
let name: Option<String> = Some("Alice".to_string());
let unwrapped_name = name.unwrap();  // 所有权转移！

println!("{}", unwrapped_name); // "Alice"
// 不能再使用 name！因为所有权已被转移
// println!("{:?}", name); // 编译错误：value borrowed after move
```

#### 3. 引用类型（特殊处理）
```rust
let data = "Hello".to_string();
let data_ref: Option<&String> = Some(&data);

let unwrapped_ref = data_ref.unwrap(); // 返回 &String（引用）

println!("{}", unwrapped_ref); // "Hello"
// 仍可继续使用 data，因为只借用了引用
```

### 关键行为总结

| 情况                     | `unwrap()` 返回类型 | 所有权变化               | 后续使用限制             |
|--------------------------|----------------------|--------------------------|--------------------------|
| `Option<T>` (T: Copy)    | `T` (值拷贝)        | 原变量仍可用             | 无限制                   |
| `Option<T>` (T: !Copy)   | `T` (所有权转移)     | 原变量失效               | 不能再用原 Option        |
| `Option<&T>`             | `&T` (借用)         | 仅借用                  | 需遵守借用规则           |
| `Result<T, E>`           | 同上规则             | 同上规则                 | 同上规则                 |

### 对比其他方法的返回值

| 方法          | 返回类型          | 所有权状态       | 是否可能 panic |
|---------------|-------------------|------------------|----------------|
| `unwrap()`    | `T`               | 所有权转移       | 是             |
| `unwrap_ref()`| `&T`              | 借用             | 是             |
| `as_ref()`    | `Option<&T>`      | 借用             | 否             |
| `copied()`    | `Option<T>`       | 拷贝（需 T: Copy）| 否             |
| `cloned()`    | `Option<T>`       | 克隆（需 T: Clone）| 否            |

### 实际内存演示

```rust
let value = String::from("Heap Data"); // 在堆上分配内存

// 场景 1：直接 unwrap
let opt = Some(value);
let unwrapped = opt.unwrap(); // 所有权从 opt 转移到 unwrapped

// 此时内存关系：
// unwrapped -> "Heap Data" (所有者)
// opt 已失效

// 场景 2：使用 as_ref 避免转移
let opt2 = Some(String::from("Another"));
let ref_view = opt2.as_ref().unwrap(); // 返回 &String

// 此时：
// opt2 仍是所有者
// ref_view 只是借用者
```

### 最佳实践建议

1. **需要所有权时**
   直接使用 `unwrap()` 获取值
   ```rust
   let config: Option<String> = /* ... */;
   let config_value = config.unwrap(); // 获得完全控制权
   ```

2. **只需读取值时**
   使用 `as_ref().unwrap()` 避免转移
   ```rust
   let name: Option<String> = Some("Alice".into());
   print_name(name.as_ref().unwrap()); // 仅借用
   // name 仍可用！
   ```

3. **需要修改值但保留 Option 时**
   使用 `as_mut().unwrap()`
   ```rust
   let mut counter: Option<u32> = Some(0);
   *counter.as_mut().unwrap() += 1; // 修改内部值
   ```

4. **处理引用容器时**
   使用 `copied()`/`cloned()` 明确所有权
   ```rust
   let numbers = [1, 2, 3];
   let opt_ref: Option<&i32> = numbers.get(1);
   let value: i32 = opt_ref.copied().unwrap(); // 安全拷贝
   ```

### 总结图
```
调用 unwrap() 时：
    如果 T 是值类型 → 返回 T (拷贝)
    如果 T 是堆类型 → 返回 T (所有权转移)
    如果 T 是引用 → 返回 &T (借用)

+---------------------+
|    Option<T>        |
|  +---------------+  |
|  | Some(value)   |--[unwrap()]--> value (所有权转移)
|  +---------------+  |
|  | None          |--[unwrap()]--> PANIC!
|  +---------------+  |
+---------------------+
```






