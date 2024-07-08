---
title: xv6 文件系统面临的问题
date: 2023-12-20 15:47:03
tags:
    - xv6
    - 操作系统
    - OS
    - 笔记
categories: OS
---

<!--more-->

## 〇、前言
本文将讨论 **xv6** 文件系统面临的挑战。

## 一、xv6 文件系统面临的挑战
### 1、cache eviction
在这个情景中，假设正在进行的事务（**transaction**）导致了对第 45 块（**block 45**）的更新。但在这个过程中，缓冲池已经被填满了，所以需要撤回第 45 块。这个撤回操作意味着将第 45 块写回磁盘。然而，这里存在的问题是：如果写回磁盘后发生了系统崩溃，会破坏被称为"write ahead rule"的规则。这个规则指出，在更新文件系统块之前，所有相关块的数据都要先写入日志中。这确保了在任何更改应用到实际块之前，有一个可追溯的记录，从而可以在发生崩溃或故障时进行恢复。

在这里，提到了一个函数叫做 `bpin()`，其作用是将块固定在缓冲池中，**通过增加块缓存的引用计数来避免缓冲池撤回相应的块**。当所有数据都写入日志后，可以使用 `unpin()` 函数在缓冲池中取消对块的固定。这是一个复杂的过程，需要精确地控制缓冲池中块的固定和释放。

在数据库系统中，对日志和缓冲池的管理至关重要，确保数据的一致性和可恢复性，特别是在事务处理和系统崩溃情况下。这种复杂性和管理是为了确保数据的安全性和一致性。

### 2、适配 log 的大小
在xv6中，日志的大小被限制为 **30** 个块。这个限制是为了确保任何文件系统操作都能够完全适配日志空间。

如果一个文件系统操作尝试写入超过 **30** 个块，这意味着其中一部分内容需要直接写入到文件系统区域，这会违反写前日志（**write ahead rule**）的规则。这条规则要求所有文件系统操作都必须在日志空间中完成。

xv6选择 30 作为日志的大小，因为在分析所有文件系统操作时发现，它们涉及的写操作数量远小于 30。**大多数操作只涉及几个块的写入，例如创建一个文件只包含了 5 个块的写入。绝大多数操作都只涉及少量块的写入。**

但是，可以想象到一些可能会涉及大量块写入的操作。例如，如果调用了 write 系统调用并传入了大量数据（比如 1MB），这将对应着写入 1000 个块，**这就会严重违反了之前提到的"文件系统操作必须适配日志大小"的规则。**

因此，这种情况下，可能需要一种机制来处理大规模写入操作，以确保其不会破坏日志大小的限制，并且仍然符合文件系统操作必须在日志空间中完成的要求。**可能的解决方案包括对大文件写入进行拆分，或者采取其他策略来保持符合日志大小的限制。**

以下为部分源码：

```c
// Write to file f.
// addr is a user virtual address.
int
filewrite(struct file *f, uint64 addr, int n)
{
  ...
  } else if(f->type == FD_INODE){
    // write a few blocks at a time to avoid exceeding
    // the maximum log transaction size, including
    // i-node, indirect block, allocation blocks,
    // and 2 blocks of slop for non-aligned writes.
    // this really belongs lower down, since writei()
    // might be writing a device like the console.
    int max = ((MAXOPBLOCKS-1-1-2) / 2) * BSIZE;
    int i = 0;
    while(i < n){
      int n1 = n - i;
      if(n1 > max)
        n1 = max;

      begin_op();
      ilock(f->ip);
      if ((r = writei(f->ip, 1, addr + i, f->off, n1)) > 0)
        f->off += r;
      iunlock(f->ip);
      end_op();

      if(r != n1){
        // error from writei
        break;
      }
      i += r;
    }
    ...

  return ret;
}
```
这个函数名为 `filewrite`，它接收一个文件结构体指针 `f`、一个用户虚拟地址 `addr` 和一个整数 `n`。这个函数用于向文件写入数据，并返回写入的字节数或者出错时返回 -1。

函数首先检查文件是否可写，如果不可写则返回 -1。然后根据文件类型执行相应的写操作：

- 如果文件类型是管道（**FD_PIPE**），则调用 `pipewrite` 函数写入数据。
- 如果文件类型是设备（**FD_DEVICE**），则通过设备号查找设备并调用相应的设备写函数。
- 如果文件类型是 **inode**（**FD_INODE**），则执行对 **inode** 对应文件的写操作。

对于 **inode** 类型的文件，函数会分批次写入数据，以避免超过最大日志事务大小。它计算了一个 `max` 变量，该变量表示每次写入的最大字节数，确保写入的数据量不会超过最大日志事务大小。然后进入一个循环，在每次循环中调用 `writei` 函数向 inode 写入数据。如果成功写入数据，函数会更新文件偏移量 `f->off`，然后继续下一次循环。如果发生写入错误或写入字节数与预期不符，函数会中断循环，返回当前已经写入的字节数或者 -1。

如果写入的字节数超过了 **max**，它会将整个 **nl** 进行拆分，多次写入。**每次写入当做一个事务处理**。

### 3、并发系统调用
在并发文件系统调用中，有一个挑战是确保事务的一致性。假设我们有两个并发的事务（t0 和 t1），它们同时在进行中，**但是 log 空间已经被用尽了，两个事务都还没有完成**。

现在的问题是：能否提交任何一个事务？答案是否定的。**因为如果提交了其中一个部分完成的事务，那么就违反了写前日志（write ahead rule），而且 log 也没有发挥应有的作用。**因此，必须确保多个并发事务合并后也适应于 log 的大小。**在没有完成一个文件系统操作之前，必须保证总的可能写入 log 数量不超过 log 区域的大小，才能允许另一个文件系统操作开始。**

为了解决这个问题，**xv6** 通过**限制并发文件系统操作的数量来确保事务的一致性**。在 `begin_op` 中，会检查当前有多少个文件系统操作正在进行。如果正在进行的文件系统操作过多，就会通过 `sleep` 停止当前文件系统操作的执行，并等待所有其他文件系统操作都执行完毕并提交之后再唤醒。这里的其他文件系统操作会一起提交，有时也被称为“group commit”。这**样的机制确保了多个操作要么全部发生，要么全部没有发生，保证了整体事务的一致性和可恢复性**：
```c
// called at the start of each FS system call.
void
begin_op(void)
{
  acquire(&log.lock);
  while(1){
    if(log.committing){
      sleep(&log, &log.lock);
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```
如果log正在commit过程中，那么就等到log提交完成，因为我们不能在 **install log** 的过程中修改log（`log.outstanding += 1`）；其次，如果当前操作是允许并发的操作个数的后一个，那么当前操作可能会超过log区域的大小，我们也需要 `sleep()` 并等待所有之前的操作结束；最后，如果当前操作可以继续执行，需要将log的`outstanding`字段加1，最后再退出函数并执行文件系统操作。

换句话说，我们的事务不是随便就能开始的，是需要满足一定的条件之后，才能展开。

```c
// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    do_commit = 1;
    log.committing = 1;
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```

同样的道理，我们事务 **commit** 也是需要满足一定条件的。首先将 `log.outstanding--`，做一些安全检查，然后确保等待 commit 的事务为 0（`log.outstanding == 0`），如果不是 0，那么其它的事务线程可能卡在了 `begin_op()`，唤醒之后，让它们继续执行（极有可能是由于 log 空间不足引起的睡眠，现在`log.outstanding--`，自然就可以唤醒了）。

这里设计地非常好，有很多细节需要注意。首先，一个单线程的事务处理流程，应该是事务开始后，能立即结束。也就是说， `log.outstanding == 1` 之后 `log.outstanding == 0`。但是为了并发，`end_op()` 中途释放了一下锁：

```c
// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  ...
  release(&log.lock); // 释放锁

  if(do_commit){
  ...
}
```

释放锁，导致其它并行的线程可以此时提交开始事务，然而，其它的线程却只能卡在 `if(log.committing)`，之后睡眠，释放锁。还记得第一个事务吗？由于第一个事务在 `log.outstanding--`之后，值为0（随后修改`do_commit = 1; log.committing = 1`）。因此可以执行 `if(do_commit){}`，提交事务。提交完成之后，随机一个 `begin_op`线程被唤醒，继续开始事务，完了之后就释放锁。

这样大量的事务都会开始，`log.outstanding++` 会导致 **log** 长度快速增长，之后 **log** 空间就会满。这时，某个事务开始时就会由于空间满了而 **sleep**，然后 `end_op` 开始执行，但是 `log.outstanding` 的值很大，只能释放锁，然后大量的 `end_op` 开始执行。在它们中的某一个执行到了`log.standing--、else{wakeup(&log)}`，由于 **log** 空间满了而卡主的那个线程就会被唤醒。当很多 `log.standing--` 被执行后，终于有一个线程可以执行`do_commit = 1; log.committing = 1`了（`outstanding 为 0`了），之后放弃锁。由于 `do_commit` 全局共享，因此它们中的任何一个事务线程都会执行`if(do_commit){}`，从而完成事务的提交。

这里需要对释放锁和获取锁有很清晰的认识，不然是不可能理清这里面的逻辑关系的。比如某个线程在 `begin_op中sleep()`，它被唤醒之后继续执行，但是此时它并没有持有 `log.lock`，它接下来怎么释放？这里面就必须认识到，线程被唤醒之后，它会重新获取它失去的锁。至于它怎么获取的，前面几章已经仔仔细细地i讨论过了，就不赘述了。

以上两段分别从一个事务提交到大量事务提交解释了 **log** 的部分工作机制。



## 二、总结（大模型总结）
这篇文章涵盖了 **xv6** 文件系统面临的挑战，主要分为三个方面：

### 1. 缓存淘汰（cache eviction）
- 介绍了在进行事务处理时可能遇到的缓存空间被填满的情况，导致需要撤回某些块并将其写回磁盘的问题。
- 提到了缓冲池管理的复杂性，特别是关于如何在日志和缓冲池之间管理块的固定和释放，以确保数据的一致性和可恢复性。

### 2. 适配日志大小
- 解释了 **xv6** 中日志大小限制为 30 个块的原因，即通过分析文件系统操作发现，绝大多数操作只涉及少量块的写入，保持日志较小。
- 强调了某些大规模写入操作可能导致破坏日志大小限制的问题，提到了可能的解决方案，如对大文件写入进行拆分等。

### 3. 并发系统调用
- 讨论了并发文件系统调用面临的挑战，强调了确保事务一致性的重要性。
- 描述了 **xv6** 如何通过限制并发文件系统操作的数量来确保事务的一致性，通过在 `begin_op` 和 `end_op` 中的锁机制和等待机制来实现。
- 分析了可能出现的并发操作引发的挑战和解决方案，包括从事务开始到结束的复杂线程交互和协作，以保证整体事务的一致性和可靠性。

这篇文章的总结可以强调 **xv6** 文件系统在处理缓存、适配日志大小和管理并发系统调用时所面临的挑战和解决方案，突出了数据一致性和可恢复性的重要性。

*全文完，感谢阅读。*















