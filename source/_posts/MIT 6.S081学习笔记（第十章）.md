---
title: MIT 6.S081学习笔记（第十章）
date: 2023-12-24 18:41:41
tags:
    - 学习
    - 笔记
    - 操作系统
    - MIT 6.S081
    - 文件系统
categories: OS
---

<!--more-->

## 〇、前言
本文主要完成 **MIT 6.S081** 实验 net 。
开始之前，切换分支：

```bash
  $ git fetch
  $ git checkout net
  $ make clean
```

## Lab: networking（hard）

### Question requirements

> Your job is to complete `e1000_transmit()` and `e1000_recv()`, both in `kernel/e1000.c`, so that the driver can transmit and receive packets. You are done when make grade says your solution passes all the tests.

### e1000_transmit()

对于发送数据，得明白数据往哪里发。
首先需要拿到一个可用的缓冲的下标，然后根据下标获取 **buffer** 的描述符，我们往这个描述符中设置我们的信息就可以了。结合 **hints**，可以轻易的写好代码：

```c
int
e1000_transmit(struct mbuf *m)
{

  acquire(&e1000_lock);

  uint32 ind = regs[E1000_TDT]; // 下一个可用的 buffer 的下标
  // 拿到描述符
  struct tx_desc *desc = &tx_ring[ind];
  // 安全检查
  if(!(desc->status & E1000_TXD_STAT_DD)) {
    release(&e1000_lock);
    return -1;
  }

  // 释放原来的信息
  if(tx_mbufs[ind]) {
    mbuffree(tx_mbufs[ind]);
    tx_mbufs[ind] = 0;
  }

  // 设置相关信息
  desc->addr = (uint64)m->head;
  desc->length = m->len;
  desc->cmd = E1000_TXD_CMD_EOP | E1000_TXD_CMD_RS;
  // 填入要发送的信息
  tx_mbufs[ind] = m;

  // 更新下一个槽位下标
  regs[E1000_TDT] = (regs[E1000_TDT] + 1) % TX_RING_SIZE;

  release(&e1000_lock);
  return 0;
}
```

### e1000_recv()

当接收到来自 E1000 网卡的数据包时，`e1000_recv` 函数起到处理这些数据包的作用。这个函数会持续地检查并处理已到达并准备好被软件接收的数据包。这是通过以下步骤实现的：

1. 读取接收尾指针寄存器并加 1 取余，以确定下一个软件可以读取的数据包在接收队列中的索引。
2. 检查描述符的状态位 `E1000_RXD_STAT_DD`，确认数据包已被硬件处理完毕，可以被内核处理。如果未被处理，停止处理。
3. 对接收缓冲区中待处理的数据包设置长度，并将其传递给网络栈的 `net_rx()` 函数进行解封装。
4. 为了替换掉已被处理的接收缓冲区，调用 `mbufalloc()` 分配一个新的缓冲区，同时更新描述符指向新的缓冲区，并将描述符的状态字段清零。
5. 更新接收尾指针 `RDT` 指向最后一个已被软件处理的描述符。

在这个过程中，考虑到可能到达的数据包超过队列大小的情况，通过使用循环确保一次中断触发后网卡软件会一直将可解封装的数据传递到网络栈。在这里没有使用锁来保护访问数据的原因是，该函数只会被中断处理函数调用，且不会出现对共享数据结构的并发访问。同时，避免使用 `e1000_lock` 是为了避免在接收到 ARP 报文时可能发生的死锁情况。

```c
static void
e1000_recv(void)
{
  while(1) {
  	// 获取到软件可以读取的位置, 也就是接收且未被软件处理的第一个数据帧在接收队列的索引
  	uint32 ind = (regs[E1000_RDT] + 1) % RX_RING_SIZE;
    // 获取缓冲描述符
    struct rx_desc *desc = &rx_ring[ind];
    // 检查
    if(!(desc->status & E1000_RXD_STAT_DD)) {
      return;
    }

    rx_mbufs[ind]->len = desc->length;
    // 传递到上层
    net_rx(rx_mbufs[ind]);

    // 因为这个缓冲区还在使用，因此需要分配并设置新的 mbuf，供给下一次轮到该下标时使用
    rx_mbufs[ind] = mbufalloc(0);
    desc->addr = (uint64)rx_mbufs[ind]->head;
    desc->status = 0;

    regs[E1000_RDT] = ind;
  }
}
```
### 测试

```bash
== Test running nettests ==
$ make qemu-gdb
(5.1s)
== Test   nettest: ping ==
  nettest: ping: OK
== Test   nettest: single process ==
  nettest: single process: OK
== Test   nettest: multi-process ==
  nettest: multi-process: OK
== Test   nettest: DNS ==
  nettest: DNS: OK
== Test time ==
time: OK
Score: 100/100
```

## 总结（大模型）
实现对于完成 MIT 6.S081 实验中的网络部分非常详细和全面。`e1000_transmit()` 函数有效地将数据包发送到 E1000 网卡中，涵盖了缓冲区描述符的正确设置和释放，并确保了发送队列的并发安全性。`e1000_recv()` 函数则能够持续地处理接收到的数据包，在循环中检查描述符的状态并进行数据包处理、网络栈传递和接收队列更新等操作。

在测试中，你展示了通过运行 `nettest` 的各项测试，包括 ping、单进程和多进程测试以及 DNS 请求，证明了代码的正确性和可靠性，最终得分为 100/100。

整体来说，你对这部分实验内容的理解和解释非常透彻，完整地涵盖了代码的实现逻辑和测试验证的过程。








