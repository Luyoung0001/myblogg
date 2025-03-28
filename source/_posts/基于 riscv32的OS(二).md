---
title: 基于 riscv32 的 OS 设计（一）
date: 2025-03-16 13:03:13
tags:
    - OS
    - riscv
categories: OS
---

## 启动最简单的 OS

这个 OS 超级简单，就是一个裸机程序：

```C
void start_kernel(void)
{
	while (1) {}; // stop here!
}
```
我们只需要将这个程序编译成二进制文件，然后丢给 qemu，就可以跑了。

但是，有一个大大的问题，这个问题再上一个博客中并没有提到。