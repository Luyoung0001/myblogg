---
title: MIT 6.S081学习笔记（第七章）
date: 2023-12-13 23:37:58
tags:
    - 学习
    - 笔记
    - 操作系统
    - MIT 6.S081
categories: OS
---

<!--more-->

## 〇、前言
本文主要完成**MIT 6.S081** 实验七：**locks**。
开始之前，切换分支：

```bash
  $ git fetch
  $ git checkout lock
  $ make clean
  ```

## 一、Memory allocator (moderate)
### Question requirements

> The program user/kalloctest stresses xv6's memory allocator: three processes grow and shrink their address spaces, resulting in many calls to `kalloc` and `kfree`. `kalloc` and `kfree` obtain `kmem.lock`. **kalloctest** prints (as "#fetch-and-add") the number of loop iterations in acquire due to attempts to acquire a lock that another core **already holds**, for the `kmem lock` and a few other locks. The number of loop iterations in acquire is a **rough measure of lock contention**.

### Some hints

- You can use the constant **NCPU** from `kernel/param.h`;
- Let freerange give all free memory to the CPU running freerange;
- The function **cpuid** returns the current core number, but it's only safe to call it and use its result when interrupts are turned off. You should use `push_off()` and `pop_off()` to turn interrupts off and on.
- Have a look at the `snprintf` function in `kernel/sprintf.c` for string formatting ideas. It is OK to just name all locks "kmem" though.

###  Answer

这个实验比较简单，目标就是尽量避免这样一种情况：当线程 **A** 拿到锁时，**B** 尽量不要再去尝试获取锁。因此，我们要**重新设计**（改进）原来的内存分配机制，为每一个**CPU** 都创建一个内存链表，这样每个核心都不会竞争别的 **CPU** 的内存链表，它们有自己的链表。

但是我们不得不承认，当 **CPU0** 的内存链表为空时，它无法为跑在本 **CPU0** 上的线程分配内存。因此，它必须去和其它的 **CPU** 去沟通，让其它的 **CPU** 给它分配一些内存。

来看看原来的设计：

```c
// kernel/kalloc.c
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE);
  return (void*)r;
}
```
可以看到，原来的设计中，由于修改链表必须当做原子化操作，因此每次获取内存时，都要占用锁。由于并发性，其它 **CPU** 在尝试获取内存时，大概率会 `aquire()` 失败。

因此，我们需要重新设计该机制。

```c
// kernel/kalloc.c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];

char *kmem_lock_names[] = {
  "kmem_cpu_0",
  "kmem_cpu_1",
  "kmem_cpu_2",
  "kmem_cpu_3",
  "kmem_cpu_4",
  "kmem_cpu_5",
  "kmem_cpu_6",
  "kmem_cpu_7",
};

void
kinit()
{
  for(int i=0;i<NCPU;i++) { // 初始化锁，这是题目要求
    initlock(&kmem[i].lock, kmem_lock_names[i]);
  }
  freerange(end, (void*)PHYSTOP);
}
```

释放内存：

```c
// kernel/kalloc.c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  push_off(); // 题目说，调用 cpuid()必须关中断，关了之后就立即打开
  int cpu = cpuid();
  pop_off();

  acquire(&kmem[cpu].lock);
  // 头插法
  r->next = kmem[cpu].freelist;
  kmem[cpu].freelist = r;
  release(&kmem[cpu].lock);
}
```

内存分配：

```c
void *
kalloc(void)
{
  struct run *r;

  push_off();
  int cpu = cpuid();
  pop_off();

  acquire(&kmem[cpu].lock);

  if(!kmem[cpu].freelist) {
    int steal_left = 64;
    for(int i=0;i<NCPU;i++) {
      if(i == cpu) continue;

      acquire(&kmem[i].lock);
      struct run *rr = kmem[i].freelist;
      while(rr && steal_left) {
        kmem[i].freelist = rr->next;
        // 头插法
        rr->next = kmem[cpu].freelist;
        kmem[cpu].freelist = rr;

        rr = kmem[i].freelist;
        steal_left--;
      }
      release(&kmem[i].lock);
      if(steal_left == 0) break; // 转移完成
    }
  }

  r = kmem[cpu].freelist;
  if(r)
    kmem[cpu].freelist = r->next;
  release(&kmem[cpu].lock);	// 释放锁
  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

至此，这个实验就完成了：

```bash
== Test running kalloctest ==
$ make qemu-gdb
(186.2s)
== Test   kalloctest: test1 ==
  kalloctest: test1: OK
== Test   kalloctest: test2 ==
  kalloctest: test2: OK
== Test   kalloctest: test3 ==
  kalloctest: test3: OK
```

但是这个解决方案会导致死锁，比如：当 **cpu0**拿到本锁，之后，尝试获取 **cpu1** 的锁来进行偷取。同时，**cpu1**拿到本锁之后，尝试拿到 **cpu0** 的锁进行偷取。这样就导致了循环等待，发生了**死锁**。

解决方法也很简单，当 **cpu0** 拿到本锁之后，当它确认要偷取 **cpu1**时，可以释放自己的本锁。这样就避免了循环等待。但是，我们仍然必须保证窃取过程是原子化操作的，怎么办呢？我们可以考虑再加一把大一点的锁。

这样就得到了不会死锁的终极代码：

```c
struct {
  struct spinlock stealing_lock;
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU]; // 为每一个 cpu 分配一个 freelist

char *kmem_lock_names[] = {
  "kmem_cpu_0",
  "kmem_cpu_1",
  "kmem_cpu_2",
  "kmem_cpu_3",
  "kmem_cpu_4",
  "kmem_cpu_5",
  "kmem_cpu_6",
  "kmem_cpu_7",
};
char *kmem_stealing_lock_names[] = {
  "kmem_cpu_00",
  "kmem_cpu_11",
  "kmem_cpu_22",
  "kmem_cpu_33",
  "kmem_cpu_44",
  "kmem_cpu_55",
  "kmem_cpu_66",
  "kmem_cpu_77",
};
```

初始化：

```c
void
kinit()
{
  for(int i = 0; i < NCPU; i++){  // 修改初始化代码
    initlock(&kmem[i].lock,kmem_lock_names[i]);
    initlock(&kmem[i].stealing_lock,kmem_stealing_lock_names[i]);
  }
  freerange(end, (void*)PHYSTOP);
}
```

内存分配：

```c
void *
kalloc(void)
{
  struct run *r;
  push_off();
  int cpu = cpuid();
  pop_off();

  acquire(&kmem[cpu].lock);
  if(!kmem[cpu].freelist){ // 如果链表为空,就偷
    // 获取大锁
    acquire(&kmem[cpu].stealing_lock);
    // 释放本锁
    release(&kmem[cpu].lock);

    int steal_left = 64;
    for(int i = 0; i < NCPU; i++){
      if(i == cpu) continue; // 不要偷自己的内存

      acquire(&kmem[i].lock);
      struct run *rr = kmem[i].freelist;
      while(rr && steal_left){
        kmem[i].freelist = rr->next; // 取出一块内存

        rr->next = kmem[cpu].freelist; // 插入
        kmem[cpu].freelist = rr;

        rr = kmem[i].freelist;
        steal_left--;
      }
      release(&kmem[i].lock);
      if(steal_left == 0) break; // 转移完成
    }
    acquire(&kmem[cpu].lock);
    release(&kmem[cpu].stealing_lock);
  }

  r = kmem[cpu].freelist;
  if(r)
    kmem[cpu].freelist = r->next;
  release(&kmem[cpu].lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

我们再看看是否会有死锁。当 **cpu0** 拿到本锁后，先拿到本**steal** 锁，开始窃取 **cpu1** 的内存时，获取 **cpu1**本锁，然后修改**cpu1** 链表。**cpu1** 如果此时持有本锁，那么窃取等待。而 **cpu1** 拿着本锁，就会轻松窃取 **cpu0** 的内存，而没有任何阻碍。完美解决了死锁问题！

但是还有一个非常关键的问题！那就是当 **cpu0** 完全放弃本锁后，在修改 **cpu0** 链表的过程中，发生了进程调度，比如 `kfree()` 被调用，那么就会发生错误！这如何解决？

答案是不用解决，**因为 xv6 保证了，只要某一个线程还没有释放完所有的锁，进程就不会调度**。这就是那个大锁 **stealing_lock**的妙处，它使得整个操作变得原子化。详见[关于这一个问题的讨论](https://github.com/Miigon/blog/issues/8)。

修改后，自然通过了测试：

```bash
== Test running kalloctest ==
$ make qemu-gdb
(195.1s)
== Test   kalloctest: test1 ==
  kalloctest: test1: OK
== Test   kalloctest: test2 ==
  kalloctest: test2: OK
== Test   kalloctest: test3 ==
  kalloctest: test3: OK
```

## 二、Buffer cache (hard)
### Question requirements

> If multiple processes use the file system intensively, they will likely contend for `bcache.lock`, which protects the disk block cache in `kernel/bio.c`. `bcachetest` creates several processes that repeatedly read different files in order to generate contention on `bcache.lock`.

> You will likely see different output, but the number of acquire loop iterations for the `bcache lock` will be **high**. If you look at the code in `kernel/bio.c`, you'll see that `bcache.lock` protects the list of cached block buffers, the reference count (b->refcnt) in each block buffer, and the identities of the cached blocks (`b->dev` and `b->blockno`).

> Modify the `block cache` so that the number of acquire loop iterations for all locks in the bcache is **close to zero** when running `bcachetest`. Ideally the sum of the counts for all locks involved in the block cache should be zero, but it's OK if the sum is **less than 500**. Modify `bget` and `brelse` so that concurrent lookups and releases for different blocks that are in the bcache are unlikely to conflict on locks (e.g., **don't all have to wait for bcache.lock**). You must maintain the invariant that at most one copy of each block is cached.

### Some hints
> - It is OK to use a fixed number of buckets and not resize the hash table dynamically. Use a prime number of buckets (e.g., 13) to reduce the likelihood of hashing conflicts.
> - Searching in the hash table for a buffer and allocating an entry for that buffer when the buffer is not found must be atomic.
> - Remove the list of all buffers (bcache.head etc.) and instead time-stamp **buffers using the time of their last use** (i.e., using **ticks** in `kernel/trap.c`). With this change `brelse` doesn't need to acquire the bcache lock, and `bget` can select the least-recently used block based on the time-stamps.
> - It is OK to serialize eviction in `bget` (i.e., the part of `bget` that selects a buffer to re-use when a lookup misses in the cache).
> - Your solution might need to **hold two locks in some cases**; for example, **during eviction you may need to hold the bcache lock and a lock per bucket.** Make sure you avoid deadlock.
> - When replacing a block, you might move a `struct buf` from one bucket to another bucket, because the new block hashes to a different bucket. You might have a tricky case: **the new block might hash to the same bucket as the old block**. Make sure you avoid deadlock in that case.
> - Some debugging tips: implement bucket locks but leave the global bcache.lock acquire/release at the beginning/end of `bget` to serialize the code. Once you are sure it is correct without race conditions, remove the global locks and deal with concurrency issues. You can also run make **CPUS=1** qemu to test with one core.

在一个缓存系统中，特别是像 **bcache** 这样的共享缓存，多个进程（或者多个 CPU）都可以同时访问和操作相同的区块。

在这种情况下，为每个 **CPU** 预先分配专属的页可能并不可行，因为这些页会被多个进程或 **CPU** 共享，无法简单地将其分配给单个处理器使用。这需要更复杂的管理和同步机制来确保多个进程或 **CPU** 可以安全地访问和使用这些共享的区块缓存。

这里锁竞争优化就不能简单降低分享了（比如上一个实验就是降低分享，将链表私有化），只能通过减少在关键区中停留的时间（对应“大锁化小锁”，降低锁的粒度）来优化了。

### 源码分析
先来看看原始的方案：

```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;
```
这里通过一个链表，将所有的 **buf** 连接起来。**head** 是一个头结点，`head.next` 是最近使用的数据块，`head.prev` 是最久未使用的数据块。

初始化：

```c
void
binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

  // Create linked list of buffers
  bcache.head.prev = &bcache.head;
  bcache.head.next = &bcache.head;
  for(b = bcache.buf; b < bcache.buf+NBUF; b++){
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    initsleeplock(&b->lock, "buffer");
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
}
```

这个函数初始化了整个 **buffer**，形成了一个双向链表结构。

最关键的是这个函数：

```c
// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}
```

首先获取整个 **buffer** 的锁，然后依次遍历，当要查询的数据块吻合时，修改引用计数，并且释放 **buffer** 的锁，获取这个数据块的锁（要准备写入数据）。

如果没有命中，就向前遍历，这也是最久未使用的块。找到 `b->refcnt == 0` 的块，然后设置一些信息，释放大锁获取小锁，执行下一步对数据块的操作。

如果所有的缓冲都被占用了，怎么办？它是通过直接中断程序来结束的。直接 `panic()`，这显然有点不负责任。

```c
// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  virtio_disk_rw(b, 1);
}
```
把这个块(**b**)拿到以后，先进行安全检查，然后就执行写入操作了。

```c
// Release a locked buffer.
// Move to the head of the most-recently-used list.
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  acquire(&bcache.lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }

  release(&bcache.lock);
}
```

这是释放快的操作，第一步就是安全检查，然后释放掉这个块的锁（因为我们不对它内部进行操作了），但是要对整个 **buffer** 进行修改，所以要上大锁，并将引用计数自减。

当它的引用计数为 **0** 时，就该释放掉了。这里为了**保持最近使用到最近最久未使用**的序列，先将 **b** 踢除，然后把它又插在了 `head->next`的位置，也就是最近未使用的位置。最后将大锁释放掉，就完成了释放 **b** 快的操作。

```c
void
bpin(struct buf *b) {
  acquire(&bcache.lock);
  b->refcnt++;
  release(&bcache.lock);
}

void
bunpin(struct buf *b) {
  acquire(&bcache.lock);
  b->refcnt--;
  release(&bcache.lock);
}
```

这两个函数很简单，就是封装的引用计数自增自减函数。

可以看到整个结构是，当要操作某个 **buf** 块的属性以及整个 **buffer** 时，需要大锁；当要往 **buf** 块中写数据时，需要小锁，理解锁的层次结构很重要。

### 源码重新设计

原版 xv6 的设计中，使用双向链表存储所有的区块缓存，每次尝试获取一个区块 **blockno** 的时候，会遍历链表，如果目标区块已经存在缓存中则直接返回，如果不存在则选取一个最近最久未使用的，且引用计数为 **0** 的 **buf** 块作为其区块缓存，并返回。

新的改进方案，可以建立一个从 **blockno** 到 **buf** 的哈希表，并为每个桶单独加锁。**仅有在两个进程同时访问的区块同时哈希到同一个桶的时候，才会发生锁竞争**。当桶中的**空闲buf** 不足的时候，从其他的桶中获取 **buf**。

首先重新设计 **bcache** 结构体：

```c
struct {

  struct buf buf[NBUF];
  struct spinlock eviction_lock; // 大锁
  struct buf bufmap[NBUFMAP_BUCKET];   // 桶
  struct spinlock bufmap_locks[NBUFMAP_BUCKET]; // 每一个桶的锁
} bcache;
```
这个结构体定义了一个大锁，并定义了桶组，还有桶锁组。

```c
void
binit(void)
{
  // 初始化缓冲桶s
  for(int i = 0; i < NBUFMAP_BUCKET; i++){
    initlock(&bcache.bufmap_locks[i],"bcache_bufmap");
    bcache.buf[i].next = 0;
  }
  initlock(&bcache.eviction_lock, "bcache_eviction");
  // 初始化缓冲区
  for(int i = 0; i < NBUF; i++){
    struct buf *b = &bcache.buf[i];
    initsleeplock(&b->lock, "buffer");
    b->lastuse = 0;
    b->refcnt = 0;
    // 将所有的缓冲区全部放进 bcache.bufmap[0]
    b->next = bcache.bufmap[0].next;
    bcache.bufmap[0].next = b;
  }
}
```

将所有的桶的 **next指针** 悬空，再将所有的缓冲区链到 **桶0** 中。

接下来就是 **bget** 函数的改写了，先用**伪代码**写出框架：

```c
// 由设备号、块号得到桶key(哈希生成函数)
// 拿到 桶key 的锁
// 循环遍历 桶key:
for(b = bcache.bufmap[key].next;b;b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.bufmap_locks[key]);
      acquiresleep(&b->lock); // 获取睡眠锁
      return b;
    }
  }
// 释放 桶key 的锁，防止持有锁时又获取其它桶的锁
// 获取大锁

// 由于中途放弃过锁,可能被其它 CPU 已经将该要映射的块映射过,防止重复映射，再检查一次
  for(b = bcache.bufmap[key].next;b;b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      acquire(&bcache.bufmap_locks[key]);
      b->refcnt++;
      release(&bcache.bufmap_locks[key]);
      release(&bcache.eviction_lock); // 释放大锁
      acquiresleep(&b->lock); // 获取睡眠锁
      return b;
    }
  }
// 如果依然没有命中，循环循环找目标 buf
// 定义一个最久未使用的前一个结点：struct buf *before_least = 0;
uint holding_bucket = -1;
for(int i = 0; i < NBUFMAP_BUCKET; i++){
	// 获取小锁（此时持有大锁）
    acquire(&bcache.bufmap_locks[i]);
    int newfound = 0;
    // 选中一个最久未使用的（ticks大的）且引用计数为 0 的 buf块
    for(b = &bcache.bufmap[i]; b->next; b = b->next) {
      if(b->next->refcnt == 0 && (!before_least || b->next->lastuse < before_least->next->lastuse)) {
        before_least = b;
        newfound = 1;
      }
    }
    if(!newfound) {
      release(&bcache.bufmap_locks[i]);
    } else {
    	// 如果选中的桶更新了，就释放前一个桶的锁，继续持有更新的桶的锁,这养还能防止其它 cpu 对这个桶引用数+1，从而导致驱逐错误
      if(holding_bucket != -1) release(&bcache.bufmap_locks[holding_bucket]);
      holding_bucket = i;
      // keep holding this bucket's lock....
    }
  }
  // 如果遍历完了所有的桶，发现没有任何 buff 块可用，继续 panic().
  // b 就是我们要找的缓冲块
  b = before_least->next;
  // 我们此时找到了一个桶，这个桶的编号并不是我们映射到的 key，那么将这个桶的 buf 转移到 key 桶中就好
  if(holding_bucket != key) {
    // 从原来的桶中转移
    before_least->next = b->next;
    // 现在释放
    release(&bcache.bufmap_locks[holding_bucket]);
    // 拿到 key锁，头插法直接插入
    acquire(&bcache.bufmap_locks[key]);
    b->next = bcache.bufmap[key].next;
    bcache.bufmap[key].next = b;
  }
  // 设置
  b->dev = dev;
  b->blockno = blockno;
  b->refcnt = 1;
  b->valid = 0;
  release(&bcache.bufmap_locks[key]);
  release(&bcache.eviction_lock);	// 释放大锁
  acquiresleep(&b->lock);
  return b;

```

以上就是伪代码和注释，以下为修改好的完整代码以及其它几个函数的代码。

`kernel/bio.c`：

```c
// Buffer cache.
//
// The buffer cache is a linked list of buf structures holding
// cached copies of disk block contents.  Caching disk blocks
// in memory reduces the number of disk reads and also provides
// a synchronization point for disk blocks used by multiple processes.
//
// Interface:
// * To get a buffer for a particular disk block, call bread.
// * After changing buffer data, call bwrite to write it to disk.
// * When done with the buffer, call brelse.
// * Do not use the buffer after calling brelse.
// * Only one process at a time can use a buffer,
//     so do not keep them longer than necessary.


#include "types.h"
#include "param.h"
#include "spinlock.h"
#include "sleeplock.h"
#include "riscv.h"
#include "defs.h"
#include "fs.h"
#include "buf.h"

// bucket number for bufmap
#define NBUFMAP_BUCKET 13
// hash function for bufmap
#define BUFMAP_HASH(dev, blockno) ((((dev)<<27)|(blockno))%NBUFMAP_BUCKET)

struct {

  struct buf buf[NBUF];
  struct spinlock eviction_lock; // 大锁
  struct buf bufmap[NBUFMAP_BUCKET];   // 桶s
  struct spinlock bufmap_locks[NBUFMAP_BUCKET]; // 桶锁s
} bcache;

void
binit(void)
{
  for(int i = 0; i < NBUFMAP_BUCKET; i++){
    initlock(&bcache.bufmap_locks[i],"bcache_bufmap");
    bcache.buf[i].next = 0;
  }
  initlock(&bcache.eviction_lock, "bcache_eviction");

  for(int i = 0; i < NBUF; i++){
    struct buf *b = &bcache.buf[i];

    initsleeplock(&b->lock, "buffer");
    b->lastuse = 0;
    b->refcnt = 0;

    b->next = bcache.bufmap[0].next;
    bcache.bufmap[0].next = b;
  }
}

// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;
  uint key = BUFMAP_HASH(dev,blockno);
  acquire(&bcache.bufmap_locks[key]);

  // hit cache
  for(b = bcache.bufmap[key].next;b;b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.bufmap_locks[key]);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // no hitting cache

  release(&bcache.bufmap_locks[key]);

  acquire(&bcache.eviction_lock);

  for(b = bcache.bufmap[key].next;b;b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      acquire(&bcache.bufmap_locks[key]);
      b->refcnt++;
      release(&bcache.bufmap_locks[key]);
      release(&bcache.eviction_lock); // 释放大锁
      acquiresleep(&b->lock); // 获取睡眠锁
      return b;
    }
  }

  // 依然没有命中缓存
  struct buf *before_least = 0;
  uint holding_bucket = -1;

  for(int i = 0; i < NBUFMAP_BUCKET; i++){
    int newfound = 0;
    acquire(&bcache.bufmap_locks[i]);
    for(b = &bcache.bufmap[i]; b->next; b = b->next){
      if(b->next->refcnt == 0 && (!before_least || b->next->lastuse < before_least->next->lastuse)){
        before_least = b;
        newfound = 1;
      }
    }
    if(!newfound){
      release(&bcache.bufmap_locks[i]);
    }else{
      if(holding_bucket != -1) release(&bcache.bufmap_locks[holding_bucket]);
      holding_bucket = i;
    }
  }

  if(!before_least){
    panic("bget: no buffers!");
  }

  b = before_least->next;

  if(holding_bucket !=key){
    before_least->next = b->next;
    release(&bcache.bufmap_locks[holding_bucket]);
    acquire(&bcache.bufmap_locks[key]);

    b->next= bcache.bufmap[key].next;
    bcache.bufmap[key].next = b;
  }
  // 设置参数
  b->dev = dev;
  b->blockno = blockno;
  b->refcnt = 1;
  b->valid = 0;
  release(&bcache.bufmap_locks[key]);
  release(&bcache.eviction_lock);
  acquiresleep(&b->lock);
  return b;
}

// Return a locked buf with the contents of the indicated block.
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b, 0);
    b->valid = 1;
  }
  return b;
}

// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  virtio_disk_rw(b, 1);
}

// Release a locked buffer.
// Move to the head of the most-recently-used list.
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  uint key = BUFMAP_HASH(b->dev,b->blockno);

  acquire(&bcache.bufmap_locks[key]);
  b->refcnt--;
  if (b->refcnt == 0) { // 事实上,不需做任何事情,更新 lastuse 就好
    b->lastuse = ticks;
  }

  release(&bcache.bufmap_locks[key]);
}

void
bpin(struct buf *b) {
  uint key = BUFMAP_HASH(b->dev, b->blockno);

  acquire(&bcache.bufmap_locks[key]);
  b->refcnt++;
  release(&bcache.bufmap_locks[key]);
}

void
bunpin(struct buf *b) {
  uint key = BUFMAP_HASH(b->dev, b->blockno);

  acquire(&bcache.bufmap_locks[key]);
  b->refcnt--;
  release(&bcache.bufmap_locks[key]);
}

```



当锁被应用时，它与之前的实验相似。持有大锁意味着线程不会被抢占。因此，即使多个线程同时请求相同的 `blockno`，并且所有线程碰巧通过了“块缓存是否存在”的初始检查并发现不存在，只有**第一个线程**进入由 `eviction_lock` 保护的“驱逐和重新分配区域”才会执行驱逐和重新分配。

只有第一个线程进入并执行了驱逐和重新分配后释放了 `eviction_lock`，`blockno` 的缓存才从不存在转变为存在。后续命中相同 `blockno` 的线程会被第二次“`blockno` 缓存是否存在”的检查拦截，并直接返回已分配的缓存 `buf`，而不会重新执行驱逐和重新分配。

这种方法的优势在于确保在查找过程中不会发生死锁，并避免了一个块生成多个缓存的极端情况。然而，它引入了一个全局的 `eviction_lock`，降低了原本并发的桶遍历过程的并行性。此外，每次缓存未命中都会增加跨桶的遍历成本。

然而，缓存未命中本身是一个相对罕见的事件。对于这些缓存未命中，因为后续操作需要从磁盘获取数据，磁盘读取时间会比跨桶查找的成本高出几个数量级。因此，我认为这种方法的额外开销是可以接受的。

至此，实验就完成了：

```bash
== Test running kalloctest ==
$ make qemu-gdb
(195.1s)
== Test   kalloctest: test1 ==
  kalloctest: test1: OK
== Test   kalloctest: test2 ==
  kalloctest: test2: OK
== Test   kalloctest: test3 ==
  kalloctest: test3: OK
== Test kalloctest: sbrkmuch ==
$ make qemu-gdb
kalloctest: sbrkmuch: OK (29.8s)
== Test running bcachetest ==
$ make qemu-gdb
(61.1s)
== Test   bcachetest: test0 ==
  bcachetest: test0: OK
== Test   bcachetest: test1 ==
  bcachetest: test1: OK
== Test usertests ==
$ make qemu-gdb
usertests: OK (161.1s)
== Test time ==
time: OK
Score: 80/80
```

## 三、总结

> - don't share if you don't have to
> - start with a few coarse-grained locks
> - instrument your code -- which locks are preventing parallelism?
> - use fine-grained locks only as needed for parallel performance
> - use an automated race detector

参考：[锁优化](https://blog.miigon.net/posts/s081-lab8-locks/#memory-allocator-moderate)、[讨论](https://github.com/Miigon/blog/issues/8)。

*全文完，感谢阅读。*















































