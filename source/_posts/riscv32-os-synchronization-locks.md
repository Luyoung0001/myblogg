---
title: "RISC-V32 OS 实验（十）：任务同步与自旋锁"
date: 2025-03-26 13:03:13
tags:
  - "RISC-V"
  - "lock"
  - "synchronization"
categories:
  - "Operating Systems"
---

## 前言

在前面，我们已经实现了抢占式任务，但是依然有很多的问题。

如果，两个任务不相关，也就是说没有数据上面的交互，那么是没有问题的。但是一旦涉及到访问共享资源，比如终端，就会出现问题：

```bash
Task 0: Running...
Task 0timer interruption!
tick: 288
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
timer interruption!
tick: 289
: Running...
Task 0: Running...
Task 0: Running...
```

可以看到，task0 执行的时候，突然被打断，接着 task1 执行，执行完后，task0 接着执行。这样导致终端显示混乱。

我们希望有一种机制，当 task0 打印的时候，如果没有打印完成，就算切换到 task1 后，那么它也什么都干不了，原地等待后，再次切换到 task0。

锁可以实现这种想法。

再比如，task0 和 task1 都对同一个变量实现 +1：

```C
int global_var = 0;

void user_task0(void) {
    uart_puts("Task 0: Created!\n");
    while (1) {
        global_var++;
        printf("task0: %d\n", global_var);
        task_delay(DELAY);
    }
}

void user_task1(void) {
    uart_puts("Task 1: Created!\n");
    while (1) {
        global_var++;
        printf("task1: %d\n", global_var);
        task_delay(DELAY);
    }
}
```

先不说 printf 占用终端引起的混乱问题：

```bash
task0: 3288
task0: 3289
task0: 3290
task0: 3291
task0: 3292
task0: 3293 <--- 被打断
timer interruption!
task1: 3295
task1: 3296
task1: 3297
task1: 3298
task1: 3299
task1: 3300
task1: 3301
task1: 3302
task1: 3303
```

task0 执行到一半，被打断，但是这时候并不知道是在什么时候打断的。可能是 task0 已经完成了 +1 操作后被打断，也有可能在执行 +1 的过程中被打断。根本原因是，+1 并不是一个原子操作，它分为好几条指令（原子操作）。

## 自旋锁

如果我们不上锁，可以预料的是，task0 的连续打印将会被打断：

```C

void user_task0(void) {
    uart_puts("Task 0: Created!\n");
    while (1) {
        uart_puts("Task 0: Begin ... \n");
        for (int i = 0; i < 5; i++) {
            uart_puts("Task 0: Running... \n");
            task_delay(DELAY);
        }
        uart_puts("Task 0: End ... \n");
    }
}

void user_task1(void) {
    uart_puts("Task 1: Created!\n");
    while (1) {
        uart_puts("Task 1: Begin ... \n");
        for (int i = 0; i < 5; i++) {
            uart_puts("Task 1: Running... \n");
            task_delay(DELAY);
        }
        uart_puts("Task 1: End ... \n");
    }
}
```
运行结果：

```bash
Task 0: Created!
Task 0: Begin ...
Task 0: Running...
Task 0: Running...
Task 0: Running...
timer interruption!
tick: 1
Task 1: Created!
Task 1: Begin ...
Task 1: Running...
Task 1: Running...
Task 1: Running...
timer interruption!
tick: 2
Task 0: Running...
Task 0: Running...
Task 0: End ...
Task 0: Begin ...
Task 0: Running...
timer interruption!
tick: 3
Task 1: Running...
Task 1: Running...
Task 1: End ...
Task 1: Begin ...
Task 1: Running...
timer interruption!
tick: 4
Task 0: Running...
QEMU: Terminated

```
可以看到，它们的任务可以随时被打断，我们想要的效果是，两个 task 在执行连续的打印任务时候，都一次性执行完。

自旋锁的实现比较简单，我们可以设置一个变量，上锁就是置 1，释放锁就是置 0。

os.h:

```C
extern struct spinlock lk;
extern void init_lock_c();
extern void spin_lock_c(struct spinlock* lock);
extern void spin_unlock_c(struct spinlock* lock);
```
lock.c:

```C
struct spinlock lk;

void init_lock_c() {
    lk.locked = 0;
}

void spin_lock_c(struct spinlock* lock) {
    for(;;) {
        if (lock->locked == 0) {
            lock->locked = 1;
            break;
        }
    }
}

void spin_unlock_c(struct spinlock* lock) {
    lock->locked = 0;
}
```

这样，我们就可以对任务上锁了：

```C
#define USE_LOCK_C

void user_task0(void) {
    uart_puts("Task 0: Created!\n");
    while (1) {
#ifdef USE_LOCK_C
        spin_lock_c(&lk);
#endif
        uart_puts("Task 0: Begin ... \n");
        for (int i = 0; i < 5; i++) {
            uart_puts("Task 0: Running... \n");
            task_delay(DELAY);
        }
        uart_puts("Task 0: End ... \n");
#ifdef USE_LOCK_C
		spin_unlock_c(&lk);
#endif
    }
}
```

试着跑一下：

```bash
Task 0: Created!
Task 0: Begin ...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 1: Created!
Task 0: Running...
Task 0: Running...
Task 0: End ...
Task 0: Begin ...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
QEMU: Terminated
```

可以发现，这根本实现不了锁。原因很简单，当 task0 获取到锁后，如果被时钟中断打断，task1 执行后，发现锁已经被拿到了，就空转。返回到 task0 后，继续执行。这时候 task0 释放掉锁后，时钟中断没有发生，task0 继续执行。这就导致，task0 可能连续执行很多次。

如果我们把时钟中断的时间间隔调整到足够小（20），以至于 task0 没完成一次打印都能切换到 task1 以让 task1 获取锁，尤其是 task0 释放掉锁后：

```bash
Task 0: Created!
Task 0: Begin ...
Task 0: Running...
Task 1: Created!
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: End ...
Task 1: Begin ...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: End ...
Task 1: Begin ...
Task 1: Running...
Task 1: Running...
```

这下勉强可以了。但是中断的时间片越小，系统的效率越低。

还有一种可能，如果时间片足够小，那么就会导致一种现象：锁坏了。怎么发生的呢？

比如当 task0 进入到锁之后，刚要上锁，这时候被打断；task1 进入锁之后，发现锁是开的，那么它就会进入临界区，执行任务。如果 task1 接着被打断进入 task0，那么 task0 就会重复上锁。锁就不起作用：

```bash
TTask 0: Created!
ask 1: Created!
Task 1: Begin ...
Task 1: Running...
Task 0: Begin ...
Task 0: Running...
Task 0: Running...
Task 1: Running...
Task 0: Running...
Task 1: Running...
Task 0: Running...
Task 1: Running...
Task 0: Running...
Task 1: Running...
Task 0: End ...
Task 0: Begin ...
Task 0: Running...
Task 1: End ...
Task 1: Begin ...
Task 1: Running...
QEMU: Terminated
```

根本原因是什么？

那就是查看锁以及上锁这两个动作可以被打断，这就是问题所在。

很遗憾，不管我们利用普通的指令实现的锁多么巧妙，都没用，都可以被打断。好消息是 ISA 考虑到了这种情况，它提供了硬件原子指令以让 OS 实现原子操作。

如果让我设计，那么就直接**上锁并保存状态**，反正**上锁并保存状态**是等幂操作，不管执行多少次都一样。检查锁的上锁前的状态，如果是 0，那么就接着执行；如果是 1，那么就继续**上锁并保存状态**。

本质上，就是原子性地获取锁这个资源，不管有没有获取成功，先获取了再说。

## test_and_set

`__sync_lock_test_and_set` 和 `__sync_lock_release` 是 GCC（GNU Compiler Collection）提供的内置函数，用于实现原子操作，尤其在多线程编程中用于实现自旋锁等同步机制。

### **1. `__sync_lock_test_and_set(ptr, value)`**

* **功能：**
    * 这是一个原子操作，用于将 `value` 存储到 `ptr` 指向的内存位置，并返回该内存位置的原始值。
    * “原子”意味着这个操作在执行期间不会被其他线程中断，确保了操作的完整性。
* **参数：**
    * `ptr`：一个指向要操作的内存位置的指针。
    * `value`：要存储到内存位置的值。
* **返回值：**
    * 返回 `ptr` 指向的内存位置的原始值（即，在存储 `value` 之前的值）。
* **在 `spin_lock_a` 中的应用：**
    * 在 `spin_lock_a` 函数中，`__sync_lock_test_and_set(&lock->locked, 1)` 用于尝试获取自旋锁。
    * 如果 `lock->locked` 的原始值为 0，表示锁未被占用，函数会将 `lock->locked` 设置为 1（表示锁已被占用），并返回 0，从而成功获取锁。
    * 如果 `lock->locked` 的原始值为 1，表示锁已被占用，函数会将 `lock->locked` 设置为 1，并返回 1，然后循环重试，直到成功获取锁。

### **2. `__sync_lock_release(ptr)`**

* **功能：**
    * 这是一个原子操作，用于将 0 存储到 `ptr` 指向的内存位置，从而释放锁。
* **参数：**
    * `ptr`：一个指向要操作的内存位置的指针。
* **返回值：**
    * 无返回值。
* **在 `spin_unlock_a` 中的应用：**
    * 在 `spin_unlock_a` 函数中，`__sync_lock_release(&lock->locked)` 用于释放自旋锁。
    * 它将 `lock->locked` 设置为 0，允许其他线程获取锁。


因此，可以这样实现：

```C
void spin_lock_a(struct spinlock* lock) {
    while (__sync_lock_test_and_set(&lock->locked, 1) != 0)
        ;
}
void spin_unlock_a(struct spinlock* lock) {
    __sync_lock_release(&lock->locked);
}
```

这样，就彻底解决了之前的问题。也可以看看反汇编文件：

```asm
800039ec <spin_lock_a>:
800039ec:	fe010113          	addi	sp,sp,-32
800039f0:	00812e23          	sw	s0,28(sp)
800039f4:	02010413          	addi	s0,sp,32
800039f8:	fea42623          	sw	a0,-20(s0)
800039fc:	00000013          	nop
80003a00:	fec42703          	lw	a4,-20(s0) <---
80003a04:	00100793          	li	a5,1
80003a08:	0cf727af          	amoswap.w.aq	a5,a5,(a4)
80003a0c:	fe079ae3          	bnez	a5, 80003a00 <spin_lock_a+0x14>
80003a10:	00000013          	nop
80003a14:	00000013          	nop
80003a18:	01c12403          	lw	s0,28(sp)
80003a1c:	02010113          	addi	sp,sp,32
80003a20:	00008067          	ret

80003a24 <spin_unlock_a>:
80003a24:	fe010113          	addi	sp,sp,-32
80003a28:	00812e23          	sw	s0,28(sp)
80003a2c:	02010413          	addi	s0,sp,32
80003a30:	fea42623          	sw	a0,-20(s0)
80003a34:	fec42783          	lw	a5,-20(s0)
80003a38:	0f50000f          	fence	iorw,ow
80003a3c:	0807a02f          	amoswap.w	zero,zero,(a5)
80003a40:	00000013          	nop
80003a44:	01c12403          	lw	s0,28(sp)
80003a48:	02010113          	addi	sp,sp,32
80003a4c:	00008067          	ret
```

可以看到，`__sync_lock_test_and_set` 被编译成 4 条指令，发生四个动作：
- 将 `lk.lock` 的值的地址放到 a4；
- 将 1 放入到 a5;
- 交换这两个值；
- 如果 lock 的原始值为 0，就继续，否则循环这 4 个步骤。

这个有问题吗？如果 task0 执行到 `amoswap.w.aq	a5,a5,(a4)` 被打断，此时 a5 的值为 0，存放 `lk.lock` 的值为 1；task1 的 a4 的地址的值就是 `lk.lock` 的值，也就是 1。交换之后，都是 1，也就进入 spin 了。task1 再次被打断，恢复上下文后进入 task0，接着执行 `bnez	a5,80003a00`，a5 寄存器为 0，继续执行任务，是没有任何问题的。

之前讨论过，锁成功的关键是查看锁的状态和上锁这两个动作不可被打断。 `amoswap.w.aq` 直接读取且上锁，且没有**寄存器-内存一致性**的问题。

## 继续讨论抢占式调度的锁

思考一个问题，如果系统的时间片很小很小（小到每执行一条指令，都会被时钟中断打断一次），那么 task0 执行完并释放锁后，task1 一定能成功获取锁。但是这么小的时间片肯定是行不通的，因为每执行一条任务中的指令，都要执行数百条和中断有关的指令，CPU 的利用率（执行有效代码的比例）将会低到不可接受。

但是如果将时间片延长到一个合理的区间，就会遇到一个严峻的问题：task0 结束释放锁后，这时候没有时间中断，task0 继续执行。如果运气不好 task1 可能很久都无法获取到锁。

这导致了一个严重的问题。考虑一个情景，如果AB两个任务协同工作，B 必须在 A 完成任务并释放锁时及时执行，A 也必须在 B 完成任务并释放锁时及时执行。按照上面的做法，B 并不会达到这个目的。

以 Linux 为例，如果时间片太大，A 执行完后释放锁，这时候时钟中断并没有发生，A 空转一段时间后，时间中断发生，B 获取锁后，继续空转。这样会浪费大量的时间。因此，时间片大小的设置，应该考虑任务的类型。


## 锁的另一种实现方式

task0 获取锁之后，如果关闭中断，那么 task0 就可以顺利执行到结束，之后打开中断。

```C
int spin_lock() {
    w_mstatus(r_mstatus() & ~MSTATUS_MIE);
    return 0;
}

int spin_unlock() {
    w_mstatus(r_mstatus() | MSTATUS_MIE);
    return 0;
}
```

运行结果：

```bash
Task 0: Created!
Task 0: Begin ...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: End ...
Task 1: Created!
Task 1: Begin ...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: End ...
Task 0: Begin ...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: Running...
Task 0: End ...
Task 1: Begin ...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: Running...
Task 1: End ...
```

这样也是一种方法，但是这已经偏离了锁本身含义的范畴，这只能保证任务执行过程中不被打断。



