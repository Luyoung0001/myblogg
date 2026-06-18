---
title: "PA4.1 思考：在 AbstractMachine 上运行 RT-Thread"
date: 2024-10-21 22:20:40
tags:
  - "RT-Thread"
  - "AbstractMachine"
  - "ysyx"
categories:
  - "Operating Systems"
---

<!--more-->

## 前言

前面谈到了异常响应机制，如果在上下文恢复的时候，做点手脚，比如别让它回到原来的上下文，而是回到了另一个进程的上下文，那么这就是进程切换，也是多道程序的基础。

## yield OS

前言就是 yield OS 的核心思想了，实现kcontext 后，别忘了修改 trap.S，使得 a0 这个返回值传到 sp。

```C
Context* kcontext(Area kstack, void (*entry)(void*), void* arg) {

    Context* ctp = (Context*)kstack.end - 1; // 开辟空间
    ctp->gpr[10] = (uintptr_t)arg;  // 传参到 a0
    ctp->mepc = (uintptr_t)entry;
    return ctp;
}
```



## RT-Thread

这个 RT-Thread 真不好啃，喝了一杯浓咖啡，直接从 中午不到 12 点弄到了晚上 6:50，我觉得主要问题是不熟悉RT-Thread 执行的整个流程，对讲义理解的不到位。

```C
rt_uint8_t *rt_hw_stack_init(void *tentry, void *parameter, rt_uint8_t *stack_addr, void *texit);
```

这个函数本来很简单，但是它的功能属实太刁难人了，如果只是 以stack_addr 为栈底创建一个入口为 tentry，参数为parameter的上下文， 并返回这个上下文结构的指针。那么就很简单：

```C
rt_uint8_t* rt_hw_stack_init(void* tentry,
                             void* parameter,
                             rt_uint8_t* stack_addr,
                             void* texit) {

    stack_addr = (rt_uint8_t*)(((uintptr_t)stack_addr + sizeof(uintptr_t) - 1) &
                               ~(sizeof(uintptr_t) - 1));


    Area stack_area = {.end = (rt_uint8_t*)stack_addr};
    rt_uint8_t* c =
        (rt_uint8_t*)kcontext(stack_area, tentry, parameter);

    return c;
}
```

但是，这里要求：若上下文对应的内核线程从 tentry 返回， 则调用 texit，RT-Thread会保证代码不会从 texit 中返回。这就复杂了，"一种方式是构造一个包裹函数, 让包裹函数来调用tentry, 并在tentry返回后调用texit, 然后将这个包裹函数作为kcontext()的真正入口, 不过这还要求我们将tentry, parameter和texit这三个参数传给包裹函数, 应该如何解决这个传参问题呢?"

我的想法是，只要将这个 mepc 直接指向 Wrapper_Func 就行了，参数传递过去就是 C 语言编程问题了。
首先实现参数结构体：
```C
typedef struct {
    void (*tentry)(void*);
    void* parameter;
    void (*texit)(void);
} thread_args_t;
```

然后就是 Wrapper_Func:

```C
static void context_wrapper(thread_args_t* args) {
    args->tentry(args->parameter);
    args->texit();

    // 不应该执行于此，RT_Thread 保证了 texit() 不会退出
    while (1)
        ;
}
```

然后，就是对它的改造了：

```C

rt_uint8_t* rt_hw_stack_init(void* tentry,
                             void* parameter,
                             rt_uint8_t* stack_addr,
                             void* texit) {

    stack_addr = (rt_uint8_t*)(((uintptr_t)stack_addr + sizeof(uintptr_t) - 1) &
                               ~(sizeof(uintptr_t) - 1));


    stack_addr -= sizeof(thread_args_t); // 在栈上分配参数结构体空间
    thread_args_t* args = (thread_args_t*)stack_addr; // 转型

    // 设置参数
    args->tentry = (void (*)(void*))tentry;
    args->parameter = parameter;
    args->texit = (void (*)(void))texit;

    Area stack_area = {.end = (rt_uint8_t*)stack_addr};  // 设置栈区域
    rt_uint8_t* c =
        (rt_uint8_t*)kcontext(stack_area, (void*)context_wrapper, args);

    return c;
}
```

完美。

还有两个函数：

```C
void rt_hw_context_switch_to(rt_ubase_t to);
void rt_hw_context_switch(rt_ubase_t from, rt_ubase_t to);
```

这两个函数的功能几乎一样。难点是，我们要传参，但是切换上下文的动作其实是 yield 之后。在事件处理回调函数ev_handler()中识别出 EVENT_YIELD 事件后, 再返回 to，顺便处理 from。

但是，ev_handler()长这个样子：

```C
static Context* ev_handler(Event e, Context* c) {
    rt_ubase_t* para;
    switch (e.event) {
        default:
            printf("Unhandled event ID = %d\n", e.event);
            assert(0);
    }

    return c;
}
```

很明显，它没有直接驾驭 to、from 的手段。怎么办？我们可以借助全局变量，，，，这是一种下乘的手段。类似于这样：

```C
rt_ubase_t g_to,g_from;
static Context* ev_handler(Event e, Context* c) {
    switch (e.event) {
        case EVENT_YIELD:
            if (from) {
                *((Context**)from) = c;
            }
            c = *(Context**)to;  // 解引用，拿到一级指针
            break;
        default:
            printf("Unhandled event ID = %d\n", e.event);
            assert(0);
    }

    return c;
}
```
swicth:

```C
void rt_hw_context_switch_to(rt_ubase_t to) {
    g_to = to;
    yield();
}

void rt_hw_context_switch(rt_ubase_t from, rt_ubase_t to) {
    g_to = to;
    g_from = from;
    yield();
}
```

是不是很丑陋？能跑吗？能跑：

```Bash
am-apps.data.size = d, am-apps.bss.size = d
heap: [0x80156000 - 0x88000000]

 \ | /
- RT -     Thread Operating System
 / | \     5.0.1 build Oct 21 2024 21:55:52
 2006 - 2022 Copyright by RT-Thread team
[I/utest] utest is initialize success.
[I/utest] total utest testcase num: (0)
Hello RISC-V!
msh />help
RT-Thread shell commands:
date             - get date and time or set (local timezone) [year month day hour min sec]
list             - list objects
version          - show RT-Thread version information
clear            - clear the terminal screen
free             - Show the memory usage in the system.
ps               - List threads in the system.
help             - RT-Thread shell help.
tail             - print the last N - lines data of the given file
echo             - echo string to file
df               - disk free
umount           - Unmount the mountpoint
mount            - mount <device> <mountpoint> <fstype>
mkfs             - format disk with file system
mkdir            - Create the DIRECTORY.
pwd              - Print the name of the current working directory.
cd               - Change the shell working directory.
rm               - Remove(unlink) the FILE(s).
cat              - Concatenate FILE(s)
mv               - Rename SOURCE to DEST.
cp               - Copy SOURCE to DEST.
ls               - List information about the FILEs.
utest_run        - utest_run [-thread or -help] [testcase name] [loop num]
utest_list       - output all utest testcase
memtrace         - dump memory trace information
memcheck         - check memory data
am_fceux_am      - AM fceux_am
am_snake         - AM snake
am_typing_game   - AM typing_game
am_microbench    - AM microbench
am_hello         - AM hello

msh />date
[W/time] Cannot find a RTC device!
local time: Thu Jan  1 08:00:00 1970
timestamps: 0
timezone: UTC+8
msh />version

 \ | /
- RT -     Thread Operating System
 / | \     5.0.1 build Oct 21 2024 21:55:52
 2006 - 2022 Copyright by RT-Thread team
msh />free
total    : 132816792
used     : 33583304
maximum  : 33583304
available: 99233488
msh />ps
thread                   pri  status      sp     stack size max used left tick  error
------------------------ ---  ------- ---------- ----------  ------  ---------- ---
tshell                    20  running 0x000000a0 0x00001000    21%   0x0000000a OK
sys workq                 23  ready   0x000000a0 0x00002000    01%   0x0000000a OK
tidle0                    31  ready   0x000000a0 0x00004000    00%   0x00000020 OK
timer                      4  suspend 0x000000f0 0x00004000    01%   0x0000000a OK
main                      10  close   0x000000e0 0x00000800    19%   0x00000014 OK
msh />pwd
/
msh />ls
No such directory
msh />memtrace

memory heap address:
name    : heap
total   : 0x132816792
used    : 0x33583304
max_used: 0x33583336
heap_ptr: 0x80156048
lfree   : 0x8215d110
heap_end: 0x87fffff0

--memory item information --
[0x80156048 -   32M] NONE
[0x82156058 -   12K] NONE
[0x82159260 -   176] NONE
[0x82159320 -    2K] NONE
[0x82159b30 -    16] main
[0x82159b50 -    72] main
[0x82159ba8 -   176] main
[0x82159c68 -    8K] main
[0x8215bc78 -   952] main
[0x8215c040 -   176] main
[0x8215c100 -    4K] main
[0x8215d110 -   94M]
msh />memcheck
msh />utest_list
[I/utest] Commands list :
msh />
```

但是，讲义不断强调，这很危险，尤其是多线程的时候。能跑只是因为nemu 是单线程，运气罢了。

怎么改？讲义给了提示：

`能否不使用全局变量来实现上下文的切换呢?

同样地, 我们需要寻找一种不会被多个线程共享的存储空间. 不过对于调用rt_hw_context_switch()的线程来说, 它的栈正在被使用, 往其中写入数据可能会被覆盖, 甚至可能会覆盖已有数据, 使当前线程崩溃. to的栈虽然当前不使用, 也不会被其他线程共享, 但需要考虑如何让ev_handler()访问到to的栈, 这又回到了我们一开始想要解决的问题.

除了栈之外, 还有没有其他不会被多个线程共享的存储空间呢? 嘿嘿, 其实前文也已经提到过它了, 那就是PCB! 因为每个线程对应一个PCB, 而一个线程不会被同时调度多次, 所以通过PCB来传递信息也是一个可行的方案. 要获取当前线程的PCB, 自然是用current指针了.

在RT-Thread中, 可以通过调用rt_thread_self()返回当前线程的PCB. 阅读RT-Thread中PCB结构体的定义, 我们发现其中有一个成员user_data, 它用于存放线程的私有数据, 这意味着RT-Thread中调度相关的代码必定不会使用这个成员, 因此它很适合我们用来传递信息. 不过为了避免覆盖user_data中的已有数据, 我们可以先把它保存在一个临时变量中, 在下次切换回当前线程并从rt_hw_context_switch()返回之前再恢复它. 至于这个临时变量, 当然是使用局部变量了, 毕竟局部变量是在栈上分配的, 完美!`

不错的方法：

```C
static Context* ev_handler(Event e, Context* c) {
    rt_thread_t current;
    rt_ubase_t* para;

    switch (e.event) {
        case EVENT_YIELD:
            current = rt_thread_self();
            para = (rt_ubase_t*)current->user_data;
            rt_ubase_t to = para[0];
            rt_ubase_t from = para[1];
            if (from) {
                *((Context**)from) = c;
            }
            c = *(Context**)to;  // 解引用，拿到一级指针
            break;
        case EVENT_IRQ_TIMER:
            return c;
        default:
            printf("Unhandled event ID = %d\n", e.event);
            assert(0);
    }

    return c;
}
```

swicth:

```C
void rt_hw_context_switch_to(rt_ubase_t to) {
    // 利用 user_data PCB 成员 user_data 传参
    rt_ubase_t temp_ud;  // 当前栈上
    rt_ubase_t user_data[2];
    rt_thread_t current = rt_thread_self();
    temp_ud = current->user_data;

    user_data[0] = to;
    current->user_data = (rt_ubase_t)user_data;
    yield();

    current->user_data = temp_ud;
}

void rt_hw_context_switch(rt_ubase_t from, rt_ubase_t to) {
    rt_ubase_t temp_ud;
    rt_ubase_t user_data[2];
    rt_thread_t current = rt_thread_self();
    temp_ud = current->user_data;


    user_data[0] = to;
    user_data[1] = from;

    current->user_data = (rt_ubase_t)user_data;

    yield();

    current->user_data = temp_ud;
}
```

这样就完美了，全局变量就不会飞了。当 swicth 获取到当前线程后：

```C
switch (e.event) {
        case EVENT_YIELD:
            current = rt_thread_self();
            para = (rt_ubase_t*)current->user_data;
            rt_ubase_t to = para[0];
            rt_ubase_t from = para[1];
            if (from) {
                *((Context**)from) = c;
            }
            c = *(Context**)to;  // 解引用，拿到一级指针
            break;
        case EVENT_IRQ_TIMER:
            return c;
        default:
            printf("Unhandled event ID = %d\n", e.event);
            assert(0);
    }
```

便可以轻松愉快地处理了。

再跑一下：

```bash
am-apps.data.size = d, am-apps.bss.size = d
heap: [0x80156000 - 0x88000000]

 \ | /
- RT -     Thread Operating System
 / | \     5.0.1 build Oct 21 2024 21:55:52
 2006 - 2022 Copyright by RT-Thread team
rt_hw_context_switch_to:to = 0x80148fd8
rt_hw_context_switch:to = 0x8215929c
[I/utest] utest is initialize success.
[I/utest] total utest testcase num: (0)
Hello RISC-V!
rt_hw_context_switch:to = 0x8215c07c
msh />help
RT-Thread shell commands:
date             - get date and time or set (local timezone) [year month day hour min sec]
list             - list objects
version          - show RT-Thread version information
clear            - clear the terminal screen
free             - Show the memory usage in the system.
ps               - List threads in the system.
help             - RT-Thread shell help.
tail             - print the last N - lines data of the given file
echo             - echo string to file
df               - disk free
umount           - Unmount the mountpoint
mount            - mount <device> <mountpoint> <fstype>
mkfs             - format disk with file system
mkdir            - Create the DIRECTORY.
pwd              - Print the name of the current working directory.
cd               - Change the shell working directory.
rm               - Remove(unlink) the FILE(s).
cat              - Concatenate FILE(s)
mv               - Rename SOURCE to DEST.
cp               - Copy SOURCE to DEST.
ls               - List information about the FILEs.
utest_run        - utest_run [-thread or -help] [testcase name] [loop num]
utest_list       - output all utest testcase
memtrace         - dump memory trace information
memcheck         - check memory data
am_fceux_am      - AM fceux_am
am_snake         - AM snake
am_typing_game   - AM typing_game
am_microbench    - AM microbench
am_hello         - AM hello

msh />date
[W/time] Cannot find a RTC device!
local time: Thu Jan  1 08:00:00 1970
timestamps: 0
timezone: UTC+8
msh />version

 \ | /
- RT -     Thread Operating System
 / | \     5.0.1 build Oct 21 2024 21:55:52
 2006 - 2022 Copyright by RT-Thread team
msh />free
total    : 132816792
used     : 33583304
maximum  : 33583304
available: 99233488
msh />ps
thread                   pri  status      sp     stack size max used left tick  error
------------------------ ---  ------- ---------- ----------  ------  ---------- ---
tshell                    20  running 0x000000a0 0x00001000    21%   0x0000000a OK
sys workq                 23  ready   0x000000a0 0x00002000    01%   0x0000000a OK
tidle0                    31  ready   0x000000a0 0x00004000    00%   0x00000020 OK
timer                      4  suspend 0x00000120 0x00004000    03%   0x0000000a OK
main                      10  close   0x00000110 0x00000800    23%   0x00000014 OK
msh />pwd
/
msh />ls
No such directory
msh />memtrace

memory heap address:
name    : heap
total   : 0x132816792
used    : 0x33583304
max_used: 0x33583336
heap_ptr: 0x80156048
lfree   : 0x8215d110
heap_end: 0x87fffff0

--memory item information --
[0x80156048 -   32M] NONE
[0x82156058 -   12K] NONE
[0x82159260 -   176] NONE
[0x82159320 -    2K] NONE
[0x82159b30 -    16] main
[0x82159b50 -    72] main
[0x82159ba8 -   176] main
[0x82159c68 -    8K] main
[0x8215bc78 -   952] main
[0x8215c040 -   176] main
[0x8215c100 -    4K] main
[0x8215d110 -   94M]
msh />memcheck
msh />utest_list
[I/utest] Commands list :

```

嗯，不错，说明实现正确。












