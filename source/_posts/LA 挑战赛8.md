---
title: LA 挑战赛：一次调试 bug 的记录
date: 2025-07-01 13:03:13
tags:
    - TLB
    - CPU
categories: 体系结构
---

## 问题

在 sc.w 执行后，我发现一切都很合规，但是 REF 跳进了一个 pc = 0x00204000 的指令，然而 DUT 还在若无其事的执行 sc.w 的下一条指令。这让我意识到了，可能是 sc.w 遇到了异常，REF 给出了正确的响应，然而 DUT 无动于衷。

## 分析过程

pc = 0x00204000 是什么异常的处理入口呢？

因为我是在启动 linux，因此必须对这个地址进行研究一下。在 linux 源码 `arch/loongarch/include/asm/loongarchregs.h` 中：

```C

#define EXCCODE_RSV		0	/* Reserved */
#define EXCCODE_TLBL		1	/* TLB miss on a load */
#define EXCCODE_TLBS		2	/* TLB miss on a store */
#define EXCCODE_TLBI		3	/* TLB miss on a ifetch */
#define EXCCODE_TLBM		4	/* TLB modified fault */
#define EXCCODE_TLBRI		5	/* TLB Read-Inhibit exception */
#define EXCCODE_TLBXI		6	/* TLB Execution-Inhibit exception */
#define EXCCODE_TLBPE		7	/* TLB Privilege Error */
```

可以看到，这是一个 TLB modified fault。这个异常是怎么发生的呢？我立即去看访存模块中的 PME 异常条件：

```Verilog
    // 0x14
    assign exu_excp_pme  = (st_b | st_h | st_w) && data_tlb_v && (csr_plv <= data_tlb_plv) && !data_tlb_d && data_addr_trans_en;
    assign excp_pme_num = exu_excp_pme ? 16'h4000 : 16'b0;
```

发现当有效位有效且权限合法且脏位不脏且是翻译模式时，就应该报这个错误。

我立即去看波形图：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/20250701_la_debug.png)

发现，符合所有的条件，但是 exu_excp_pme 为 0？原来我并没有将 sc_w 这个指令放到中间判断，这样修改就好了：

```Verilog
    // 0x14
    assign exu_excp_pme  = (st_b | st_h | st_w | sc_do) && data_tlb_v && (csr_plv <= data_tlb_plv) && !data_tlb_d && data_addr_trans_en;
    assign excp_pme_num = exu_excp_pme ? 16'h4000 : 16'b0;
```
这样，这个 bug 就修好了。

## 题外话: 关于脏位

我认为，这个脏位 d 本来的名字就有问题，它和脏位一点关系都没有，在硬件层次上，它更像一个权限检查位。

当访存指令访问某一个page 的时候，tlb 拿到给出的访存地址后，发现这个地址要访问的page 没被修改过，于是立即进入异常处理阶段，让软件判断是否有写的权限。如果有，返回后就继续执行，然后让其写成功并 d 位置 1，如果没有权限直接报段错误并杀死程序。

因此 这个脏位 d 其实就是权限位。

这个 d 位也是操作系统内核实现 COW(Copy on write)的关键。由于多个进程共享同一块物理内存（节省内存），但是当其中任何一个写入数据时，才复制出一个私有的物理页。

比如 fork() 子进程，初始和父进程公用相同物理页。如果是读，没什么事情。但是如果想写，就会进入 mpe 异常处理，检查该页是否标记为“写时复制”，如果是那么就分配一块新的物理页，然后把旧物理页的内容拷贝到新页，接着修改当前进程页表，把这个虚拟页映射到新物理页，并更新 d=1，返回后再次执行那条 store。







