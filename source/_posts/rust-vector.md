---
title: "Rust 学习笔记（三）：Vector 基础与常用操作"
date: 2025-09-26 13:03:13
tags:
  - "Rust"
  - "Vec"
  - "collections"
categories:
  - "Programming Languages"
---

## 一、Vector 基础特性

1. **动态大小**：可在运行时增长或缩小
2. **堆分配**：元素存储在堆内存中
3. **类型安全**：所有元素必须是相同类型 `T`
4. **所有权管理**：Vector 拥有其元素的所有权

## 二、创建 Vector 的多种方式

### 1. 使用 `Vec::new()`
```rust
let mut v: Vec<i32> = Vec::new(); // 指定类型
v.push(1); // 添加元素
```

### 2. 使用 `vec!` 宏（推荐）
```rust
let v = vec![1, 2, 3]; // 自动推断为 Vec<i32>
let zeros = vec![0; 5]; // [0, 0, 0, 0, 0]
```

### 3. 从迭代器创建
```rust
let v: Vec<_> = (1..=5).collect(); // [1, 2, 3, 4, 5]
```

### 4. 预分配容量
```rust
let mut v = Vec::with_capacity(10); // 容量10，长度0
println!("容量: {}, 长度: {}", v.capacity(), v.len()); // 10, 0
```

## 三、核心操作与常用方法

### 1. 添加元素
```rust
let mut fruits = vec!["apple"];
fruits.push("banana"); // 尾部添加
fruits.insert(1, "orange"); // 插入到索引1位置
// ["apple", "orange", "banana"]
```

### 2. 移除元素
```rust
let mut nums = vec![10, 20, 30, 40];
let last = nums.pop(); // 移除并返回最后一个元素: Some(40)
let removed = nums.remove(1); // 移除索引1的元素: 20
// nums 现在是 [10, 30]
```

### 3. 访问元素
```rust
let v = vec!["red", "green", "blue"];

// 安全访问（推荐）
if let Some(color) = v.get(1) {
    println!("第二个颜色: {}", color); // green
}

// 直接索引（可能 panic）
let first = &v[0]; // "red"
// let invalid = &v[5]; // panic!

// 最后一个元素
let last = v.last().unwrap(); // "blue"
```

### 4. 修改元素
```rust
let mut scores = vec![65, 70, 85];
if let Some(score) = scores.get_mut(1) {
    *score += 5; // 修改第二个元素
}
// scores 现在是 [65, 75, 85]
```

## 四、遍历与迭代

### 1. 只读迭代
```rust
let numbers = vec![10, 20, 30];
for num in &numbers {
    println!("{}", num);
}
```

### 2. 可变迭代
```rust
let mut temps = vec![22.5, 18.7, 25.3];
for temp in &mut temps {
    *temp += 1.0; // 每个温度加1度
}
```

### 3. 获取索引和值
```rust
let colors = vec!["red", "green", "blue"];
for (index, color) in colors.iter().enumerate() {
    println!("索引 {}: {}", index, color);
}
```

### 4. 使用迭代器方法
```rust
let nums = vec![1, 2, 3, 4, 5];
let sum: i32 = nums.iter().sum();
let doubled: Vec<_> = nums.iter().map(|x| x * 2).collect();
```

## 五、内存管理与性能优化

### 1. 容量与长度
```rust
let mut v = Vec::with_capacity(5);
println!("初始: 长度={}, 容量={}", v.len(), v.capacity()); // 0, 5

v.extend([1, 2, 3, 4, 5].iter());
println!("满容: 长度={}, 容量={}", v.len(), v.capacity()); // 5, 5

v.push(6);
println!("扩容后: 长度={}, 容量={}", v.len(), v.capacity()); // 6, 10
```

### 2. 手动管理内存
```rust
let mut v = vec![1, 2, 3, 4];

// 释放多余容量
v.shrink_to_fit(); // 容量变为4

// 预分配空间
v.reserve(100); // 确保至少能添加100个元素
println!("新容量: {}", v.capacity()); // >=104
```

## 六、高级用法

### 1. 存储不同类型（使用枚举）
```rust
enum WebEvent {
    PageLoad,
    KeyPress(char),
    Click { x: i64, y: i64 },
}

let events = vec![
    WebEvent::PageLoad,
    WebEvent::KeyPress('a'),
    WebEvent::Click { x: 100, y: 200 },
];
```

### 2. 多维 Vector
```rust
// 创建 3x3 矩阵
let mut matrix = vec![];
for i in 0..3 {
    matrix.push(vec![i*3 + 1, i*3 + 2, i*3 + 3]);
}

// 访问元素
println!("中心元素: {}", matrix[1][1]); // 5
```

### 3. 向量切片
```rust
let numbers = vec![1, 2, 3, 4, 5];
let slice: &[i32] = &numbers[1..4]; // [2, 3, 4]

// 切片作为函数参数
fn print_slice(slice: &[i32]) {
    for num in slice {
        print!("{} ", num);
    }
}
```

### 4. 排序和搜索
```rust
let mut words = vec!["banana", "apple", "cherry"];
words.sort(); // ["apple", "banana", "cherry"]

let index = words.binary_search(&"banana").unwrap(); // 1
```

## 七、常见使用场景

### 1. 读取文件内容
```rust
use std::fs;

fn read_lines(filename: &str) -> Vec<String> {
    fs::read_to_string(filename)
        .expect("读取文件失败")
        .lines()
        .map(|s| s.to_string())
        .collect()
}
```

### 2. 命令行参数处理
```rust
fn main() {
    let args: Vec<String> = std::env::args().collect();
    println!("程序名: {}", args[0]);

    if args.len() > 1 {
        println!("第一个参数: {}", args[1]);
    }
}
```

### 3. 高效数据处理
```rust
// 筛选素数
fn primes_up_to(n: u32) -> Vec<u32> {
    let mut sieve = vec![true; (n+1) as usize];
    sieve[0] = false;
    sieve[1] = false;

    for i in 2..=(n as f64).sqrt() as u32 {
        if sieve[i as usize] {
            for j in (i*i..=n).step_by(i as usize) {
                sieve[j as usize] = false;
            }
        }
    }

    sieve.iter().enumerate()
        .filter(|(_, &is_prime)| is_prime)
        .map(|(i, _)| i as u32)
        .collect()
}
```

## 八、性能注意事项

1. **push/pop 操作**：平均 O(1) 时间复杂度
2. **插入/删除中间元素**：O(n) 时间复杂度
3. **索引访问**：O(1) 时间复杂度
4. **搜索未排序数据**：O(n) 时间复杂度
5. **扩容成本**：当容量不足时，需要重新分配内存和复制元素

## 九、最佳实践

1. **优先使用 `vec!` 宏**：简洁且高效
2. **预分配容量**：当知道大致大小时，使用 `Vec::with_capacity()`
3. **安全访问**：优先使用 `get()` 而不是直接索引
4. **迭代代替索引**：更安全且通常更高效
5. **复用内存**：使用 `clear()` 清空内容但保留容量

```rust
let mut buffer = Vec::with_capacity(1024);
// 第一次使用
buffer.extend(b"Hello, world!".iter().copied());

// 清空后复用
buffer.clear();
buffer.extend(b"Reused memory".iter().copied());
```

## 十、与其他集合对比

| 特性             | Vector (`Vec<T>`)       | 数组 (`[T; N]`)       | 切片 (`&[T]`)        |
|------------------|-------------------------|-----------------------|----------------------|
| **大小**         | 动态可变               | 固定                 | 动态但不可变        |
| **内存位置**      | 堆                     | 栈                   | 借用                |
| **所有权**        | 拥有元素               | 拥有元素             | 借用元素            |
| **索引访问**      | O(1)                   | O(1)                 | O(1)                |
| **添加元素**      | O(1) 均摊              | 不可能               | 不可能              |
| **适用场景**      | 动态集合、未知大小数据 | 固定大小、性能关键区 | 函数参数、视图      |



