---
title: 《OSTEP》信号量（chap31）
date: 2023-11-13 18:32:25
tags:
    - 操作系统
    - 笔记
    - 学习
categories: OS
---

<!--more-->

## 〇、前言
本文是对《OSTEP》第三十一章的实践与总结。
## 一、信号量
以下各个版本都是一个生产者-消费者模型，基于信号量实现，并且逐渐完善。

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#define MAX 10

int buffer[MAX];
int fill = 0;
int use = 0;

// 生产行为
void put(int value) {
    buffer[fill] = value;
    fill = (fill + 1) % MAX;
}

// 消费行为
int get() {
    int temp = buffer[use];
    use = (use + 1) % MAX;
    return temp;
}
sem_t *empty;
sem_t *full;

// 生产者
void *producer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        sem_wait(empty);
        put(i);
        sem_post(full);
    }
    return NULL;
}
// 消费者
void *consumer(void *arg) {
    int value = 0;
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        sem_wait(full);
        value = get();
        sem_post(empty);
        printf("消费:%d\n", value);
    }
    return NULL;
}
int main() {
    // sem_init(&empty, 0, MAX);
    // sem_init(&full, 0, 0);
    // 以上写法已经在 macOS 中废弃
    empty = sem_open("empty", O_CREAT, S_IRUSR | S_IWUSR, MAX);
    full = sem_open("full", O_CREAT, S_IRUSR | S_IWUSR, 0);
    pthread_t p1, p2;
    int loop = 10;
    pthread_create(&p1, NULL, producer, &loop);
    pthread_create(&p2, NULL, consumer, &loop);

    pthread_join(p1, NULL);
    pthread_join(p2, NULL);

    sem_close(empty);
    sem_close(full);

    sem_unlink("empty");
    sem_unlink("full");

    return 0;
}
```
运行结果：

```bash
****** chap31_信号量 % ./a
消费:0
消费:1
消费:2
消费:3
消费:4
消费:5
消费:6
消费:7
消费:8
消费:9
```
可以看到在单线程下（一个生产者线程，一个消费者线程）运行得很好。

```c
int main() {
...
    pthread_t p1, p2, p3;
    int loop1 = 10;
    int loop3 = 10;
    pthread_create(&p1, NULL, producer, &loop1);
    pthread_create(&p2, NULL, producer, &loop1);
    pthread_create(&p3, NULL, consumer, &loop3);
...
}
```
运行结果：

```bash
****** chap31_信号量 % ./a
消费:0
消费:1
消费:2
消费:3
消费:4
消费:5
消费:0
消费:6
消费:1
消费:7
```
***运行结果可能看不出什么***，但是生产者产生的**值会被覆盖**。这个情况是怎么发生的呢？

假设两个生产者（Pa和Pb）几乎同时调用`put()`。当Pa先运行，在f1行先加入第一条数据（fill=0），假设Pa在将fill计数器更新为1之前被中断，Pb开始运行，也在f1行给缓冲区的0位置加入一条数据，这意味着那里的老数据被覆盖。

因此，我们的解决思路是将 `put()`修饰成一个临界区。

```c
sem_t *empty;
sem_t *full;
sem_t *mutex;

// 生产者
void *producer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        sem_wait(mutex);
        sem_wait(empty);
        put(i);
        sem_post(full);
        sem_post(mutex);
    }
    return NULL;
}
// 消费者
void *consumer(void *arg) {
    int value = 0;
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {
        sem_wait(full);
        value = get();
        sem_post(empty);
        printf("消费:%d\n", value);
    }
    return NULL;
}
int main(){
	...
	mutex = sem_open("mutex", O_CREAT, S_IRUSR | S_IWUSR, 1);
	...
}
```
运行结果：

```bash
****** chap31_信号量 % ./a
消费:0
消费:1
消费:2
消费:0
消费:3
消费:1
消费:4
消费:2
消费:5
消费:3
```
那如果继续增加一个消费者呢？

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#define MAX 10

int buffer[MAX];
int fill = 0;
int use = 0;

// 生产行为
void put(int value) {
    buffer[fill] = value;
    fill = (fill + 1) % MAX;
}

// 消费行为
int get() {
    int temp = buffer[use];
    use = (use + 1) % MAX;
    return temp;
}
sem_t *empty;
sem_t *full;
sem_t *mutex;

// 生产者
void *producer(void *arg) {
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {

        sem_wait(empty);
        sem_wait(mutex);
        put(i);
        sem_post(mutex);
        sem_post(full);
    }
    return NULL;
}
// 消费者
void *consumer(void *arg) {
    int value = 0;
    int loops = *((int *)arg);
    for (int i = 0; i < loops; i++) {

        sem_wait(full);
        value = get();
        sem_post(empty);

        printf("消费:%d\n", value);
    }
    return NULL;
}
int main() {
    empty = sem_open("empty", O_CREAT, S_IRUSR | S_IWUSR, MAX);
    full = sem_open("full", O_CREAT, S_IRUSR | S_IWUSR, 0);
    mutex = sem_open("mutex", O_CREAT, S_IRUSR | S_IWUSR, 1);
    pthread_t p1, p2, p3, p4;
    int loop1 = 3;
    int loop2 = 3;
    pthread_create(&p1, NULL, producer, &loop1);
    pthread_create(&p2, NULL, producer, &loop1);
    pthread_create(&p3, NULL, consumer, &loop2);
    pthread_create(&p4, NULL, consumer, &loop2);

    pthread_join(p1, NULL);
    pthread_join(p2, NULL);
    pthread_join(p3, NULL);
    pthread_join(p4, NULL);

    sem_close(empty);
    sem_close(full);
    sem_close(mutex);

    sem_unlink("empty");
    sem_unlink("full");
    sem_unlink("mutex");

    return 0;
}
```
运行结果：

```bash
****** chap31_信号量 % ./a
消费:0
消费:0
消费:1
消费:2
消费:1
消费:3
消费:2
消费:3
消费:5
消费:4
消费:6
消费:5
消费:7
消费:6
消费:4
消费:7
消费:8
消费:9
消费:8
消费:9
```

运行得非常好，有一个问题，会不会出现重复消费行为（把一个值消费两次）？看起来好像会。
但是记得 full 信号量，初始值为 0，这意味着每次只允许一个线程进行消费。这不仅和生产者达成了正确的执行序列，而且还间接得使得每次只能有一个线程在 `get()`。

## 二、读写锁
以下是利用信号量来实现一个读写锁。
数据在有线程读时，不能写。这意味着第一个线程读的时候，直接将写者锁占有，最后一个读者将写锁释放。以下是代码：

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#include <string.h>

// 读取的空间
char space[10] = "hello";

// 读者写者锁
typedef struct _rwlock_t {
    sem_t *lock;
    sem_t *writelock;
    int readers;
} rwlock_t;

// 初始化
void rwlock_init(rwlock_t *rw) {
    rw->readers = 0;
    rw->lock = sem_open("rw_lock", O_CREAT, S_IRUSR | S_IWUSR, 1);
    rw->writelock = sem_open("rw_writelock", O_CREAT, S_IRUSR | S_IWUSR, 1);
}
// 获得读锁(实际上就不需要上锁,上锁只是为了修改 rw)
void rwlock_acquire_readlock(rwlock_t *rw) {
    sem_wait(rw->lock);
    rw->readers++;
    if (rw->readers == 1) {
        sem_wait(rw->writelock); // 第一个读者需要抢占写锁
    }
    sem_post(rw->lock);
}
// 释放读锁(实际上就不需要上锁,上锁只是为了修改 rw)
void rwlock_release_readlock(rwlock_t *rw) {
    sem_wait(rw->lock);
    rw->readers--;
    if (rw->readers == 0) {
        sem_post(rw->writelock); // 最后一个读者需要释放写锁
    }
    sem_post(rw->lock);
}
// 获得写锁
void rwlock_acquire_writelock(rwlock_t *rw) { sem_wait(rw->writelock); }
// 释放写锁
void rwlock_release_writelock(rwlock_t *rw) { sem_post(rw->writelock); }

rwlock_t rwlock;

// 写线程
void *write_t(void *arg) {
    rwlock_acquire_writelock(&rwlock);
    strcpy(space, arg);
    rwlock_release_writelock(&rwlock);
    return NULL;
}

// 读线程
void *read_t(void *arg) {
    rwlock_acquire_readlock(&rwlock);
    printf("%s\n", space);
    rwlock_release_readlock(&rwlock);
    return NULL;
}

int main() {
    rwlock_init(&rwlock);
    char arg[10] = "1111111";
    // 创建 5 个读线程和 5 个写线程
    pthread_t p_r1, p_r2, p_r3, p_r4, p_r5;
    pthread_t p_w1, p_w2, p_w3, p_w4, p_w5;

    pthread_create(&p_r1, NULL, read_t, NULL);
    pthread_create(&p_r2, NULL, read_t, NULL);

    pthread_create(&p_w1, NULL, write_t, arg);
    arg[2] = '2';
    pthread_create(&p_w2, NULL, write_t, arg);

    arg[5] = '5';
    pthread_create(&p_w5, NULL, write_t, arg);

    pthread_create(&p_r3, NULL, read_t, NULL);
    pthread_create(&p_r4, NULL, read_t, NULL);
    arg[3] = '3';
    pthread_create(&p_w3, NULL, write_t, arg);
    arg[4] = '4';
    pthread_create(&p_w4, NULL, write_t, arg);
    pthread_create(&p_r5, NULL, read_t, NULL);

    pthread_join(p_r1, NULL);
    pthread_join(p_r2, NULL);
    pthread_join(p_r3, NULL);
    pthread_join(p_r4, NULL);
    pthread_join(p_r5, NULL);
    pthread_join(p_w1, NULL);
    pthread_join(p_w2, NULL);
    pthread_join(p_w3, NULL);
    pthread_join(p_w4, NULL);
    pthread_join(p_w5, NULL);
    sem_close(rwlock.lock);
    sem_close(rwlock.writelock);

    sem_unlink("rw_lock");
    sem_unlink("rw_writelock");

    return 0;
}
```
运行结果：
```bash
****** chap31_信号量 % ./a
hello
hello
1121151
1121151
1123451
```

可以顺利运行。
## 三、哲学家就餐问题

```c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>
// 模拟四位哲学家就餐,使得他们能顺利就餐,并且不会有人挨饿.
// 拿到餐具的人,吃完饭后就立即放下餐具,让给其他人(看起来不是那么卫生)

// 设置餐具

sem_t forks[5];
// 初始化这些叉子
void forks_init(){
    for(int i = 0; i < 5; i++){
        sem_init(&forks[i],0,1);
    }
}

// 假设哲学家编号为 p,则他拿的叉子(左右两个叉子)编号应该为:
int left(int p){
    return p;
}
int right(int p){
    return (p+1)%5;
}

// 拿到叉子,准备吃饭
void getforks(int p){
    if(p == 4){
        sem_wait(&forks[right(p)]);
        sem_wait(&forks[left(p)]);
        // 拿到了两个叉子就能吃饭了
        printf("%d号哲学家开始就餐.\n",p);
        sleep(2);
    }else{
    sem_wait(&forks[left(p)]);
    sem_wait(&forks[right(p)]);
    // 拿到了两个叉子就能吃饭了
    printf("%d号哲学家开始就餐.\n",p);
    sleep(2);
    }
}
// 放下餐具
void putforks(int p){
    printf("%d号哲学家完成就餐,并放下了叉子.\n",p);
    sem_post(&forks[left(p)]);
    sem_post(&forks[right(p)]);
}

// 就餐
void *eat(void *arg){
    int p = *((int*)arg);
    getforks(p);
    putforks(p);
    return NULL;
}


int main(){
    forks_init();
    pthread_t p[5];
    int i0 = 0;
    pthread_create(&p[0],NULL,eat,&i0);
    int i1 = 1;
    pthread_create(&p[1],NULL,eat,&i1);
    int i2 = 2;
    pthread_create(&p[2],NULL,eat,&i2);
    int i3 = 3;
    pthread_create(&p[3],NULL,eat,&i3);
    int i4 = 4;
    pthread_create(&p[4],NULL,eat,&i4);

    for(int i = 0; i < 5; i++){
        pthread_join(p[i],NULL);
        sem_close(&forks[i]);
    }
    return 0;
}
```
运行结果（这里是在 Linux 平台）：
```bash
******:~/OSTEP_notes_linux_version/chap31_信号量# gcc -o a phi_eat.c -lrt -lpthread
******:~/OSTEP_notes_linux_version/chap31_信号量# ./a
0号哲学家开始就餐.
2号哲学家开始就餐.
0号哲学家完成就餐,并放下了叉子.
2号哲学家完成就餐,并放下了叉子.
4号哲学家开始就餐.
1号哲学家开始就餐.
4号哲学家完成就餐,并放下了叉子.
1号哲学家完成就餐,并放下了叉子.
3号哲学家开始就餐.
3号哲学家完成就餐,并放下了叉子.
```
可以看到所有的哲学家都能顺利完成就餐。这里解决的核心思路就是，最后一个哲学家拿刀叉的顺序和别的哲学家不一样，这在理论上不会造成死锁。
试分析一下（p 代表哲学家，f 代表叉子），易知假设在 t 时刻是一个环形等待：
- p0 占据 f0 请求 f1;
- p1 占据 f1 请求 f2;
- p2 占据 f2 请求 f3;
- p3 占据 f3 请求 f0;

画出资源配置图之后，很容易求出它的可达性矩阵：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/99c48ab024d14f309e5a0db4e8417e64_1720253717123.png?token=ANB4BCLAN3VN3QWJJS2WG2TGRD6VC)

因此，可达性矩阵为：
$$
  P =  \left(
   \begin{matrix}
   1 & 1 & 1 & 1  \\
      1 & 1 & 1 & 1  \\
   1 & 1 & 1 & 1  \\
     1 & 1 & 1 & 1  \\
  \end{matrix}
  \right)
$$
强分图也就是:
$$
  P\bigwedge P^T =  \left(
   \begin{matrix}
   1 & 1 & 1 & 1  \\
      1 & 1 & 1 & 1  \\
   1 & 1 & 1 & 1  \\
     1 & 1 & 1 & 1  \\
  \end{matrix}
  \right)
$$

因此，在 t 时刻处于死锁状态，这与我们的经验相一致。

## 四、实现一个信号量

这里利用锁和条件变量来实现一个信号量。

```c
#include <pthread.h>

// 自己定义信号量
typedef struct {
    int value;
    pthread_cond_t cond;
    pthread_mutex_t lock;
}zem_t;

// 初始化
void zem_init(zem_t *zem,int value){
    zem->value = value;
    pthread_cond_init(&zem->cond,NULL);
    pthread_mutex_init(&zem->lock,NULL);
}

// wait
void zem_wait(zem_t *zem){
    pthread_mutex_lock(&zem->lock);
    while(zem->value <= 0){
        pthread_cond_wait(&zem->cond,&zem->lock);
    }
    zem->value--;
    pthread_mutex_unlock(&zem->lock);
}

// post
void *zem_post(zem_t *zem){
    pthread_mutex_lock(&zem->lock);
    zem->value++;
    pthread_cond_signal(&zem->cond);
    pthread_mutex_unlock(&zem->lock);
}

```

这确实是相当优雅。







