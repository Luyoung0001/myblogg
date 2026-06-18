---
title: "OSTEP 读书笔记：条件变量（第 30 章）"
date: 2023-11-11 22:10:26
tags:
  - "OSTEP"
  - "condition variable"
  - "concurrency"
categories:
  - "Operating Systems"
---

<!--more-->

## 〇、前言
本文是对《OSTEP》第三十章的实践与总结。

## 一、条件变量

```c
#include <pthread.h>
#include <stdio.h>
#include <assert.h>

int buffer;
int count = 0; // 资源为空

// 生产,在 buffer 中放入一个值
void put(int value) {
    assert(count == 0);
    count = 1;
    buffer = value;
}
// 消费,取出 buffer 中的值
int get() {
    assert(count == 1);
    count = 0;
    return buffer;
}

/***********第一版本**********/
// 生产者
void *producer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        put(i);
    }
    return NULL;
}
// 消费者
void *consumer(void *arg) {
    while (1) {
        int temp = get();
        printf("消费的值:%d\n", temp);
    }
    return NULL;
}

int main() {
    pthread_t p1, p2;
    int arg = 100;
    pthread_create(&p1, NULL, producer, &arg);
    pthread_create(&p2, NULL, consumer, NULL);
    // 等待两个线程结束
    pthread_join(p1, NULL);
    pthread_join(p2, NULL);
    return 0;
}
```

运行：

```bash
*** chap30_条件变量 % gcc -o a con_prodece.c
*** chap30_条件变量 % ./a
Assertion failed: (count == 0), function put, file con_prodece.c, line 10.
消费的值:0
Assertion failed: (count == 1), function get, file con_prodece.c, line 16.
zsh: abort      ./a
```

可以看到，断言直接失败。

```c
pthread_cond_t cond;
pthread_mutex_t mutex;
/***********第二版本**********/
// 生产者
void *producer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        pthread_mutex_lock(&mutex);
        if (count == 1) {
            pthread_cond_wait(&cond, &mutex);
        }
        put(i);
        pthread_mutex_unlock(&mutex);
        pthread_cond_signal(&cond);
    }
    return NULL;
}
// 消费者
void *consumer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        pthread_mutex_lock(&mutex);
        if (count == 0) {
            pthread_cond_wait(&cond, &mutex);
        }
        int temp = get();
        pthread_mutex_unlock(&mutex);
        printf("消费:%d\n", temp);
        pthread_cond_signal(&cond);
    }
    return NULL;
}

int main() {

    // 初始化互斥锁和条件变量
    pthread_mutex_init(&mutex, NULL);
    pthread_cond_init(&cond, NULL);

    pthread_t p1, p2;
    int arg = 100;
    pthread_create(&p1, NULL, producer, &arg);
    pthread_create(&p2, NULL, consumer, &arg);
    // 等待两个线程结束
    pthread_join(p1, NULL);
    pthread_join(p2, NULL);

    // 销毁互斥锁和条件变量
    pthread_mutex_destroy(&mutex);
    pthread_cond_destroy(&cond);
    return 0;
}
```

运行结果：

```bash
*** chap30_条件变量 % ./a
消费:0
消费:1
消费:2
消费:3
...
消费:95
消费:96
消费:97
消费:98
消费:99
```
可以看到，在两个线程的情况下，工作的很好。

```c
/***********第三版本**********/
// 生产者
void *producer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        pthread_mutex_lock(&mutex);
        if (count == 1) {
            pthread_cond_wait(&cond, &mutex);
        }
        put(i);
        pthread_mutex_unlock(&mutex);
        pthread_cond_signal(&cond);
    }
    return NULL;
}
// 消费者
void *consumer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        pthread_mutex_lock(&mutex);
        if (count == 0) {
            pthread_cond_wait(&cond, &mutex);
        }
        int temp = get();
        pthread_mutex_unlock(&mutex);
        printf("消费:%d\n", temp);
        pthread_cond_signal(&cond);
    }
    return NULL;
}

int main() {

    // 初始化互斥锁和条件变量
    pthread_mutex_init(&mutex, NULL);
    pthread_cond_init(&cond, NULL);

    pthread_t p1, p2, p3;
    int arg = 100;
    int arg1 = 50;
    int arg2 = 50;
    pthread_create(&p1, NULL, producer, &arg);
    pthread_create(&p2, NULL, consumer, &arg1);
    pthread_create(&p3, NULL, consumer, &arg2);
    // 等待两个线程结束
    pthread_join(p1, NULL);
    pthread_join(p2, NULL);
    pthread_join(p3, NULL);
    // 销毁互斥锁和条件变量
    pthread_mutex_destroy(&mutex);
    pthread_cond_destroy(&cond);
    return 0;
}
```

运行结果：

```bash
*** chap30_条件变量 % gcc -o a con_prodece2.c
*** chap30_条件变量 % ./a
消费:0
消费:1
Assertion failed: (count == 1), function get, file con_prodece2.c, line 18.
zsh: abort      ./a
```
可以看到，再增加了一个消费线程之后，出现了断言错误。这是因为出现了假唤醒，使得某个线程醒来后，断言错误。解决方法很简单，直接将 `if()`换成 `while()`:

```c
/***********第三版本**********/
// 解决虚假唤醒
// 生产者
void *producer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        pthread_mutex_lock(&mutex);
        while (count == 1) {
            pthread_cond_wait(&cond, &mutex);
        }
        put(i);
        pthread_mutex_unlock(&mutex);
        pthread_cond_signal(&cond);
    }
    return NULL;
}
// 消费者
void *consumer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        pthread_mutex_lock(&mutex);
        while (count == 0) {
            pthread_cond_wait(&cond, &mutex);
        }
        int temp = get();
        pthread_mutex_unlock(&mutex);
        printf("消费:%d\n", temp);
        pthread_cond_signal(&cond); // 在解锁之后发出信号
    }
    return NULL;
}

int main() {

    // 初始化互斥锁和条件变量
    pthread_mutex_init(&mutex, NULL);
    pthread_cond_init(&cond, NULL);

    pthread_t p1, p2, p3;
    int arg = 100;
    int arg1 = 50;
    int arg2 = 50;
    pthread_create(&p1, NULL, producer, &arg);
    pthread_create(&p2, NULL, consumer, &arg1);
    pthread_create(&p3, NULL, consumer, &arg2);
    // 等待两个线程结束
    pthread_join(p1, NULL);
    pthread_join(p2, NULL);
    pthread_join(p3, NULL);
    // 销毁互斥锁和条件变量
    pthread_mutex_destroy(&mutex);
    pthread_cond_destroy(&cond);
    return 0;
}
```
运行结果：

```bash
luliang@shenjian chap30_条件变量 % ./a
消费:0
消费:2
消费:1
消费:3
消费:4
消费:5
...
```

可以看到会卡住，这其实是由于三个线程都睡眠导致的，这种情况是怎么发生的呢？
假设生产者唤醒了第一个消费者，消费者又恰巧唤醒了第二个生产者，第二个生产者被唤醒之后又睡眠。这样三个线程都在睡眠。解决问题的办法就是消费者只能唤醒生产者，生产者只能唤醒消费者。以下就是终极版本的代码：

```c
#include <assert.h>
#include <pthread.h>
#include <stdio.h>

int buffer;
int count = 0; // 资源为空
pthread_cond_t cond_consumer;
pthread_cond_t cond_procedure;
pthread_mutex_t mutex;

// 生产,在 buffer 中放入一个值
void put(int value) {
    assert(count == 0);
    count = 1;
    buffer = value;
}
// 消费,取出 buffer 中的值
int get() {
    assert(count == 1);
    count = 0;
    return buffer;
}

/***********第四版本**********/
// 假设生产者唤醒了第一个消费者,消费者又唤醒了第二个生产者,第二个生产者
// 之后又睡眠.这样三个线程都在睡眠.
// 核心问题就是,消费者只能唤醒生产者,生产者只能唤醒消费者.
// 生产者
void *producer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        pthread_mutex_lock(&mutex);
        while (count == 1) {
            pthread_cond_wait(&cond_procedure, &mutex);
        }
        put(i);
        pthread_mutex_unlock(&mutex);
        pthread_cond_signal(&cond_consumer);

    }
    return NULL;
}
// 消费者
void *consumer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        pthread_mutex_lock(&mutex);

        while (count == 0) {
            pthread_cond_wait(&cond_consumer, &mutex);
        }
        int temp = get();
        pthread_mutex_unlock(&mutex);
        pthread_cond_signal(&cond_procedure);
        printf("消费:%d\n", temp);
    }
    return NULL;
}

int main() {

    // 初始化互斥锁和条件变量
    pthread_mutex_init(&mutex, NULL);
    pthread_cond_init(&cond_consumer, NULL);
    pthread_cond_init(&cond_procedure, NULL);

    pthread_t p1, p2, p3;
    int arg = 100;
    int arg1 = 50;
    int arg2 = 50;
    pthread_create(&p1, NULL, producer, &arg);
    pthread_create(&p2, NULL, consumer, &arg1);
    pthread_create(&p3, NULL, consumer, &arg2);
    // 等待线程结束
    pthread_join(p1, NULL);
    pthread_join(p2, NULL);
    pthread_join(p3, NULL);
    // 销毁互斥锁和条件变量
    pthread_mutex_destroy(&mutex);
    pthread_cond_destroy(&cond_consumer);
    pthread_cond_destroy(&cond_procedure);
    return 0;
}
```
运行结果：

```bash
*** chap30_条件变量 % gcc -o a con_prodece3.c
*** chap30_条件变量 % ./a
消费:0
消费:2
消费:1
...
消费:97
消费:98
消费:99
```

可以看到运行得很好，成功地解决了并发、虚假唤醒以及全部线程都睡眠的情况。

## 二、总结
我们看到了引入锁之外的另一个重要同步原语：条件变量。当某些程序状态不符合要求时，通过允许线程进入休眠状态，条件变量使我们能够漂亮地解决许多重要的同步问题，包括著名的（仍然重要的）生产者/消费者问题，以及覆盖条件。

