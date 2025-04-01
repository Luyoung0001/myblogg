---
title: 基于 riscv32 的 OS 设计：完结
date: 2025-04-01 13:03:13
tags:
    - OS
    - riscv
categories: OS
---

## 总结

这十三篇博客是学习 [循序渐进，学习开发一个RISC-V上的操作系统 - 汪辰 - 2021春](https://www.bilibili.com/video/BV1Q5411w7z5?vd_source=dfa664e0d7eeb6d8f8d0b215db38c05c&spm_id_from=333.788.videopod.episodes) 的笔记，今日终于完结了。

学完之后，收获很大，如果要是做过 riscv CPU 设计那效果更佳。

如果不考虑性能（需要精巧的数据结构和算法），OS 的核心就是模式转换。soc 提供了各种设备，启动的时候，得初始化这些设备，之后就能利用这些设备了，比如 CLINT、PLIC、UART 等。

ISA 也提供了很多规范以支持 OS 的特性或高级特性。

由于时间有限，课程并没有牵扯到虚拟内存、内存保护等，不过这依然是不错的一个入门教程。

