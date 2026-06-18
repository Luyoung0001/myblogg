---
title: "xv6 系统调用笔记（下）：sleep、wakeup 与同步机制"
date: 2023-12-11 14:47:30
tags:
  - "xv6"
  - "system call"
  - "sleep"
categories:
  - "Operating Systems"
---

<!--more-->

## 〇、前言
本文将会结合源代码谈论 **sleep**、**wakeup** 这两个系统调用。

## 一、sleep()系统调用
以下是`sleep()`函数源码：
```c
// Atomically release lock and sleep on chan.
// Reacquires lock when awakened.
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();

  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.

  acquire(&p->lock);  //DOC: sleeplock1
  release(lk);

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  release(&p->lock);
  acquire(lk);
}
```

先来看看 **lost wakeup** 问题。当一个进程在 `sleep()` 时，如果 `sleep()` 了一半，状态还没来得及修改为**SLEEPING**，这时候发生了中断，并且被某些进程调用了 `wakeup()`，那么这个 `wakeup()`肯定不能把这个进程唤醒。而且，在被中断恢复后，它将永远等不到唤醒，因为唤醒已经错过。所以，在这里必须要正**不可中断性**和**操作先后性**。因为在下面就会看到 `wakeup()` 只唤醒状态为 **SLEEPING** 的进程。

因此我们必须保证，`sleep()` 是一个原子操作，在 `sleep()` 执行过程中，要么执行完全，要么没有被执行。所以这里必须加一个进程锁。所以在下面就会看到 `wakeup()` 中也会尝试获取休眠的进程锁。

在持有进程锁的时候，将进程的状态设置为 **SLEEPING** 并记录**sleep channel**，之后再调用 `sched()` 函数，这个函数中会再调用 `swtch()` 函数（而这会返回到 `scheduler()`函数中），此时 `sleep()` 函数中仍然持有了进程的锁，`wakeup()` 仍然不能做任何事情。

因此在 sleep()之后，这个锁必须释放。我们来看看细节：

```c
void
scheduler(void)
{
 ...
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0; // 返回的位置，此刻继续执行
      }
      release(&p->lock);
 ...
}
```
在这里，它会继续执行上一次执行到的位置，即 `c->proc = 0`，然后执行 `release(&p->lock)`，也就是释放锁，而且释放的是 `sleep()` 中的当前进程的锁。（这一点不是很好理解，可以理解为用的上一个进程的代码释放当前进程的锁？总之，这些代码就冰冷冷的放在内存里，被 **pc** 不断地指一遍又一遍）。更有意思的是，在 sched() 函数返回之后，继续运行：

```c
void
sleep(void *chan, struct spinlock *lk)
{
  ...
  sched();

  // Tidy up.
  p->chan = 0; // 就绪执行的位置

  // Reacquire original lock.
  release(&p->lock);
  acquire(lk);
}
```

这里 `release(&p->lock)`  实际上释放的是 `scheduler()` 中选中的进程的锁。

所以在调度器线程释放进程锁之后，`wakeup()` 才能终于获取进程的锁，发现它正在 **SLEEPING**状态，并唤醒它。

这里的效果是由之前定义的一些规则确保的，这些规则包括了：
- 调用 **sleep** 时需要持有**condition lock**，这样 **sleep** 函数才能知道相应的锁；
- **sleep**函数只有在获取到进程的锁 `p->lock`之后，才能释放 **condition lock**；
- **wakeup**需要同时持有两个锁才能查看进程。


## 二、wakeup()调用
以下是 `wakeup()` 的源码：

```c
// Wake up all processes sleeping on chan.
// Must be called without any p->lock.
void
wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    if(p != myproc()){
      acquire(&p->lock);
      if(p->state == SLEEPING && p->chan == chan) {
        p->state = RUNNABLE;
      }
      release(&p->lock);
    }
  }
}
```
可以看到它的工作很简单，检查两个条件之后，就修改进程的状态为 **RUNNABLE**。


## 三、总结
这篇文章详细地介绍了 xv6 操作系统中的 `sleep()` 和 `wakeup()` 系统调用的实现原理以及相关的内部工作机制。主要强调了在 `sleep()` 中的原子操作性，确保了操作的完整性，以及在 `wakeup()` 中唤醒休眠进程的方式。

### 关于 `sleep()`：

强调了 `sleep()` 操作的原子性，使用进程锁确保 `sleep()` 操作是一个原子操作，避免了 **"lost wakeup"** 问题的发生。
通过释放持有的锁，让出 CPU 控制权，进入 **SLEEPING** 状态，然后释放进程锁，使得其他进程能够继续运行。
调度器在合适的时机恢复了进程的执行，完成 `sleep()` 操作。
### 关于 `wakeup()`：

`wakeup()` 通过遍历进程列表，并获取每个进程的锁，查看处于 **SLEEPING** 状态且 **sleep channel** 匹配的进程，将其状态设置为 **RUNNABLE**，唤醒进程。

整体上，这篇文章清晰地解释了 `sleep()` 和 `wakeup()` 这两个关键系统调用的工作原理和实现细节，突出了在并发环境下确保原子性操作和避免死锁的重要性。

*全文完，感谢阅读。*





