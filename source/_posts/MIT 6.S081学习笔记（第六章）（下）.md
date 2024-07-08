---
title: MIT 6.S081学习笔记（第六章）（下）
date: 2023-12-08 09:04:37
tags:
    - 学习
    - 笔记
    - 操作系统
    - MIT 6.S081
    - 文件系统
categories:
    - OS
    - 系统编程
    - Unix/Linux
---

<!--more-->

## 〇、前言
**MIT 6.S081** 实验六：**Multithreading**；
开始之前，切换分支：

```bash
 $ git fetch
  $ git checkout thread
  $ make clean
```

## 一、实验：Multithreading
## Uthread: switching between threads (moderate)
> In this exercise you will design the **context switch mechanism** for a user-level threading system, and then implement it. To get you started, your xv6 has two files `user/uthread.c` and `user/uthread_switch.S`, and a rule in the **Makefile** to build a uthread program. `uthread.c` contains most of a user-level threading package, and code for three simple test threads. The threading package is missing some of the code to create a thread and to switch between threads.

> Your job is to come up with a plan to create threads and save/restore registers to switch between threads, and implement that plan. When you're done, make grade should say that your solution passes the uthread test.

### Some Hints

- `thread_switch` needs to save/restore only the callee-save registers. Why?
- You can see the assembly code for uthread in `user/uthread.asm`, which may be handy for debugging.

这个实验思路很简单，本质上就是模拟 **xv6** 中的线程上下文切换机制。以下为代码：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

/* Possible states of a thread: */
#define FREE        0x0
#define RUNNING     0x1
#define RUNNABLE    0x2

#define STACK_SIZE  8192
#define MAX_THREAD  4


struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context ctx;
};
struct thread all_thread[MAX_THREAD];
struct thread *current_thread;
extern void thread_switch(uint64, uint64);

void
thread_init(void)
{
  // main() is thread 0, which will make the first invocation to
  // thread_schedule(). It needs a stack so that the first thread_switch() can
  // save thread 0's state.
  current_thread = &all_thread[0];
  current_thread->state = RUNNING;
}

void
thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread + 1;
  for(int i = 0; i < MAX_THREAD; i++){
    if(t >= all_thread + MAX_THREAD)
      t = all_thread;
    if(t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0) {
    printf("thread_schedule: no runnable threads\n");
    exit(-1);
  }

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->ctx, (uint64)&next_thread->ctx);
  } else
    next_thread = 0;
}

void
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->ctx.sp = (uint64)&t->stack + (STACK_SIZE - 1);
  t->ctx.ra = (uint64)func;
}

void
thread_yield(void)
{
  current_thread->state = RUNNABLE;
  thread_schedule();
}

volatile int a_started, b_started, c_started;
volatile int a_n, b_n, c_n;

void
thread_a(void)
{
  int i;
  printf("thread_a started\n");
  a_started = 1;
  while(b_started == 0 || c_started == 0)
    thread_yield();

  for (i = 0; i < 100; i++) {
    printf("thread_a %d\n", i);
    a_n += 1;
    thread_yield();
  }
  printf("thread_a: exit after %d\n", a_n);

  current_thread->state = FREE;
  thread_schedule();
}

void
thread_b(void)
{
  int i;
  printf("thread_b started\n");
  b_started = 1;
  while(a_started == 0 || c_started == 0)
    thread_yield();

  for (i = 0; i < 100; i++) {
    printf("thread_b %d\n", i);
    b_n += 1;
    thread_yield();
  }
  printf("thread_b: exit after %d\n", b_n);

  current_thread->state = FREE;
  thread_schedule();
}

void
thread_c(void)
{
  int i;
  printf("thread_c started\n");
  c_started = 1;
  while(a_started == 0 || b_started == 0)
    thread_yield();

  for (i = 0; i < 100; i++) {
    printf("thread_c %d\n", i);
    c_n += 1;
    thread_yield();
  }
  printf("thread_c: exit after %d\n", c_n);

  current_thread->state = FREE;
  thread_schedule();
}

int
main(int argc, char *argv[])
{
  a_started = b_started = c_started = 0;
  a_n = b_n = c_n = 0;
  thread_init();
  thread_create(thread_a);
  thread_create(thread_b);
  thread_create(thread_c);
  current_thread->state = FREE;
  thread_schedule();
  exit(0);
}

```
以及对 `user/uthread_swtch.S` 的改动：

```bash
	.text

	/*
         * save the old thread's registers,
         * restore the new thread's registers.
         */

	.globl thread_switch
thread_switch:
	/* YOUR CODE HERE */
	sd ra, 0(a0)
	sd sp, 8(a0)
	sd s0, 16(a0)
	sd s1, 24(a0)
	sd s2, 32(a0)
	sd s3, 40(a0)
	sd s4, 48(a0)
	sd s5, 56(a0)
	sd s6, 64(a0)
	sd s7, 72(a0)
	sd s8, 80(a0)
	sd s9, 88(a0)
	sd s10, 96(a0)
	sd s11, 104(a0)

	ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1)
    ld s1, 24(a1)
    ld s2, 32(a1)
    ld s3, 40(a1)
    ld s4, 48(a1)
    ld s5, 56(a1)
    ld s6, 64(a1)
    ld s7, 72(a1)
    ld s8, 80(a1)
    ld s9, 88(a1)
    ld s10, 96(a1)
    ld s11, 104(a1)

	ret    /* return to ra */

```

## Using threads (moderate)

> **n** this assignment you will explore parallel programming with threads and locks using a hash table. You should do this assignment on a real Linux or MacOS computer (not xv6, not qemu) that has multiple cores. Most recent laptops have multicore processors.

### Some hints
> - The numbers you see may differ from this sample output by a factor of two or more, depending on how fast your computer is, whether it has multiple cores, and whether it's busy doing other things.
> - ph runs two benchmarks. First it adds lots of keys to the hash table by calling `put()`, and prints the achieved rate in `puts` per second. The it fetches keys from the hash table with `get()`. It prints the number keys that should have been in the hash table as a result of the puts but are missing (zero in this case), and it prints the number of gets per second it achieved.

熟悉多线程编程的话，直接**上两个锁**就完成了，但为了性能问题，要将临界区的范围控制到**很小**，不然第二个测试过不了。

声明两个锁：
```c
pthread_mutex_t get_lock;
pthread_mutex_t put_lock;
```
这两个锁记得初始化：

```c
int
main(int argc, char *argv[])
{
  pthread_t *tha;
  void *value;
  double t1, t0;
  pthread_mutex_init(&put_lock, NULL);
  pthread_mutex_init(&get_lock, NULL);
  ...
}

```

`put()`、`get()`中设置临界区：
```c
static
void put(int key, int value)
{
  int i = key % NBUCKET;


  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    pthread_mutex_lock(&put_lock);
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&put_lock);
  }


}

static struct entry*
get(int key)
{
  int i = key % NBUCKET;

  pthread_mutex_lock(&get_lock);
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key) break;
  }
  pthread_mutex_unlock(&get_lock);

  return e;
}
```

## Barrier(moderate)

> In this assignment you'll implement a **barrier**: a point in an application at which all participating threads must wait until all other participating threads reach that point too. You'll use **pthread condition variables**, which are a sequence coordination technique similar to xv6's `sleep` and `wakeup`.

### Some hints
- `pthread_cond_wait` releases the **mutex** when called, and re-acquires the mutex before returning.

- We have given you `barrier_init()`. Your job is to implement `barrier()` so that the panic doesn't occur. We've defined `struct barrier` for you; its fields are for your use.

- You have to deal with a succession of barrier calls, each of which we'll call a **round**. `bstate.round` records the current round. You should increment `bstate.round` each time all threads have reached the barrier.
- You have to handle the case in which one thread races around the loop before the others have exited the barrier. In particular, you are re-using the `bstate.nthread` variable from one round to the next. Make sure that a thread that leaves the barrier and races around the loop doesn't increase `bstate.nthread` while a previous round is still using it.

实现一个**barrier**，每个线程在**barrier**处等待，直到所有的线程都已经运行到了这一点，即一共运行 **nthread** 次，**nthread** 由 `main()` 函数参数提供。**barrier**通过**pthread**库提供的条件变量来实现。

**pthread**提供了两个API，`pthread_cond_wait(&cond, &mutex)` 使线程在条件变量**cond**上等待，并释放**mutex**锁。`pthread_cond_broadcast(&cond)` 用于唤醒所有在条件变量**cond**上等待的线程。

```c
static void
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread++;
  if (bstate.nthread >= nthread) {
    bstate.round++;
    pthread_cond_broadcast(&bstate.barrier_cond);
    bstate.nthread = 0;
  } else {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);

}
```

## 实验总结：

这个实验涉及多个部分：

### Multithreading - Uthread: Switching Between Threads
这部分任务是设计并实现用户级线程系统的上下文切换机制。代码中包含了一个简单的线程包，但缺少创建线程和线程之间切换的部分。主要涉及 `uthread.c` 和 `uthread_switch.S` 两个文件的实现，要完成线程创建、线程调度和上下文切换等功能。

### Using Threads
这个部分需要在具有多个核心的真实操作系统上完成。任务是使用线程和锁实现一个哈希表，并进行基准测试。需要处理临界区问题，确保性能的同时完成 `put()` 和 `get()` 操作。

### Barrier
这个部分要求实现一个屏障（barrier），通过 **pthread** 提供的条件变量实现。线程在 **barrier** 处等待，直到所有线程都到达该点。需要考虑多个线程之间的同步问题，确保所有线程都调用了 **barrier** 并适当地增加 **bstate.round**。

整体而言，这个实验涉及了多线程编程、上下文切换机制设计、锁的使用、以及线程同步问题的解决。需要理解和实现线程的创建、调度、上下文切换，以及在多线程环境下处理临界区和同步问题。


*全文完，感谢阅读。*





