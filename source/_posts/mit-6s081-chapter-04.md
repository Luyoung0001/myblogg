---
title: "MIT 6.S081 学习笔记：RISC-V 汇编与 traps"
date: 2023-11-24 15:33:41
tags:
  - "MIT 6.S081"
  - "xv6"
  - "operating systems"
categories:
  - "Operating Systems"
---

<!--more-->

〇、前言
本文主要完成**MIT 6.S081** 实验四：**traps**。
开始之前，切换分支：

```bash
 $ git fetch
 $ git checkout traps
 $ make clean
```
## RISC-V assembly (easy)
### Question requirements
It will be important to understand a bit of **RISC-V assembly**, which you were exposed to in 6.004. There is a file `user/call.c` in your xv6 repo. `make fs.img` compiles it and also produces a readable assembly version of the program in `user/call.asm`.
`c` 源代码：
```c
#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int g(int x) {
  return x+3;
}

int f(int x) {
  return g(x);
}

void main(void) {
  printf("%d %d\n", f(8)+1, 13);
  exit(0);
}

```
部分`asm`代码：

```c
...
int g(int x) {
   0:	1141                	addi	sp,sp,-16
   2:	e422                	sd	s0,8(sp)
   4:	0800                	addi	s0,sp,16
  return x+3;
}
   6:	250d                	addiw	a0,a0,3
   8:	6422                	ld	s0,8(sp)
   a:	0141                	addi	sp,sp,16
   c:	8082                	ret

000000000000000e <f>:

int f(int x) {
   e:	1141                	addi	sp,sp,-16
  10:	e422                	sd	s0,8(sp)
  12:	0800                	addi	s0,sp,16
  return g(x);
}
  14:	250d                	addiw	a0,a0,3
  16:	6422                	ld	s0,8(sp)
  18:	0141                	addi	sp,sp,16
  1a:	8082                	ret

000000000000001c <main>:

void main(void) {
  1c:	1141                	addi	sp,sp,-16
  1e:	e406                	sd	ra,8(sp)
  20:	e022                	sd	s0,0(sp)
  22:	0800                	addi	s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:	4635                	li	a2,13
  26:	45b1                	li	a1,12
  28:	00000517          	auipc	a0,0x0
  2c:	7d850513          	addi	a0,a0,2008 # 800 <malloc+0xe8>
  30:	00000097          	auipc	ra,0x0
  34:	62a080e7          	jalr	1578(ra) # 65a <printf>
  exit(0);
  38:	4501                	li	a0,0
  3a:	00000097          	auipc	ra,0x0
  3e:	298080e7          	jalr	664(ra) # 2d2 <exit>
  ...
```



Read the code in `call.asm` for the functions `g`, `f`, and `main`. The instruction manual for **RISC-V** is on the reference page. Here are some questions that you should answer (store the answers in a file answers-traps.txt):

- Q: Which registers contain arguments to functions? For example, which register holds `13` in main's call to `printf`?

- A: `a0~a7`, 事实上，当函数的参数超过 **8** 个时，就会保存到栈空间；否则，就会依次保存到寄存器中。从这里也可以看出，当可以使用寄存器的时候，我们不会使用内存，我们只在不得不使用内存的场景才使用它。从 `main`的汇编代码可以看到，`13` 被存到`a2`中。
- Q: Where is the call to function `f` in the assembly code for `main`? Where is the call to `g`? (Hint: the compiler may inline functions.)
- A: `main`函数里面没有`f` 的汇编代码，这是因为它被优化了。函数 `f `就是使传入的参数加 `3`后返回。考虑到编译器会进行内联优化，这就意味着一些显而易见的，编译时可以计算的数据会在编译时得出结果，而不是进行函数调用。其它同理。
- Q: At what address is the function printf located?
 ```c
void
printf(const char *fmt, ...)
{
 65a:	711d                	addi	sp,sp,-96
 65c:	ec06                	sd	ra,24(sp)
 65e:	e822                	sd	s0,16(sp)
 660:	1000                	addi	s0,sp,32
 ...
 }
```
- A:  **0x65a**.
- Q: What value is in the register `ra` just after the jalr to printf in main?

```bash
  30:	00000097          	auipc	ra,0x0
  34:	62a080e7          	jalr	1578(ra) # 65a <printf>
```

- A: 使用 `auipc ra,0x0` 将当前程序计数器 `pc` 的值存入 `ra` 中；`jalr	1578(ra) # 65a <printf>`  跳转到偏移地址 `printf `处，也就是 **0x65a** 的位置。

- Q: Run the following code.
```c
	unsigned int i = 0x00646c72;
	printf("H%x Wo%s", 57616, &i);
```

- Q1: What is the output?
- A1: **57616** 转换为 **16** 进制为 **e110**，所以格式化描述符 `%x` 打印出了它的 **16** 进制值。所以会打印出：**He110 World**。
- Q2: If the RISC-V were instead big-endian what would you set i to in order to yield the same output?
- A2：如果在小端（little-endian）处理器中，数据**0x00646c72** 的高字节存储在内存的高位，那么从内存低位，也就是低字节开始读取，对应的 ASCII 字符为 **rld**。
如果在 大端（big-endian）处理器中，数据 **0x00646c72** 的高字节存储在内存的低位，那么从内存低位，也就是高字节开始读取其 ASCII 码为 **dlr**。
所以如果大端序和小端序输出相同的内容 `i` ，那么在其为大端序的时候，`i` 的值应该为 **0x726c64**，这样才能保证从内存低位读取时的输出为 **rld** 。
- Q3: Would you need to change **57616** to a different value?
- 无论 **57616** 在大端序还是小端序，它的二进制值都为 **e110** 。大端序和小端序只是改变了多字节数据在内存中的存放方式，并不改变其真正的值的大小，所以 **57616** 始终打印为二进制 **e110** 。如果在大端序，`i` 的值应该为 **0x00646c72** 才能保证与小端序输出的内容相同。不用该变 **57616** 的值。

- Q: In the following code, what is going to be printed after `y=`? (note: the answer is not a specific value.) Why does this happen?: `printf("x=%d y=%d", 3);`
- A: 函数的参数是通过寄存器`a1`, `a2` 等来传递。如果 `prinf` 少传递一个参数，那么其仍会从一个确定的寄存器中读取其想要的参数值，但是我们并没有给出这个确定的参数并将其存储在寄存器中，所以函数将从此寄存器中获取到一个随机的不确定的值作为其参数。故而此例中，`y=`**后面的值我们不能够确定，它是一个垃圾值**。

## Backtrace (moderate)
### Question requirements

> For debugging it is often useful to have a backtrace: a list of the function calls on the stack above the point at which the error occurred.

### Some hints:
- Add the prototype for backtrace to `kernel/defs.h` so that you can invoke backtrace in `sys_sleep`.
- The GCC compiler stores the frame pointer of the currently executing function in the register `s0`. Add the following function to `kernel/riscv.h`:

```c
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```

and call this function in `backtrace` to read the current frame pointer. This function **uses in-line assembly to read `s0`**.
- These lecture notes have a picture of the layout of stack frames. Note that the return address lives at a fixed **offset (-8)** from the frame pointer of a stackframe, and that the saved frame pointer lives at fixed **offset (-16)** from the frame pointer.
- Xv6 allocates one page for each stack in the xv6 kernel at PAGE-aligned address. You can compute the top and bottom address of the stack page by using `PGROUNDDOWN(fp)` and `PGROUNDUP(fp)` (see `kernel/riscv.h`. These number are helpful for backtrace to terminate its loop.

Once your backtrace is working, call it from panic in `kernel/printf.c` so that you see the kernel's backtrace when it panics.

做这个实验，需要对函数调用栈有点了解。
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/3aa59c94df994ea2abf94ed26a00eb40_1720253710031.png?token=ANB4BCOA4PPU2SPL7RZ22UTGRD6UW)
[【原文】](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec05-calling-conventions-and-stack-frames-risc-v/5.5-stack)每一次我们调用一个函数，函数都会为自己创建一个**Stack Frame**，并且只给自己用。函数通过移动**Stack Pointer**来完成**Stack Frame**的空间分配。

对于**Stack**来说，是从**高地址开始向低地址使用**。所以栈总是向下增长。当我们想要创建一个新的**Stack Frame**的时候，总是对当前的**Stack Pointer**做**减法**。一个函数的**Stack Frame**包含了保存的寄存器，本地变量，并且，如果函数的参数多于8个，额外的参数会出现在**Stack**中。所以**Stack Frame**大小并不总是一样，即使在这个图里面看起来是一样大的。不同的函数有不同数量的本地变量，不同的寄存器，所以**Stack Frame**的大小是不一样的。但是有关**Stack Frame**有两件事情是确定的：
- **Return address**总是会出现在**Stack Frame**的第一位
- 指向前一个**Stack Frame**的指针也会出现在栈中的固定位置


有关**Stack Frame**中有两个重要的寄存器，第一个是**SP（Stack Pointer）**，它指向**Stack**的底部并代表了当前**Stack Frame**的位置。第二个是**FP（Frame Pointer）**，它指向当前**Stack Frame**的顶部。因为**Return address**和指向前一个**Stack Frame**的指针都在当前**Stack Frame**的固定位置，所以可以通过当前的**FP**寄存器寻址到这两个数据。

我们保存前一个**Stack Frame**的指针的原因是为了让我们能跳转回去。所以当前函数返回时，我们可以将前一个**Frame Pointer**存储到**FP**寄存器中。所以我们使用**Frame Pointer**来操纵我们的**Stack Frames**，并确保我们总是指向正确的函数。

所以，思路就是拿到指向当前 **Frame Pointer**的指针，然后在一个适当的空间中遍历所有的**Frame Pointer**，并在整个过程中打印每一个栈帧中的返回地址 **Return address**。

因此在`printf.c`中加上：

```c
void backtrace(void) {
    uint64 fp = r_fp();
    uint64 top = PGROUNDUP(fp);
    uint64 bottom = PGROUNDDOWN(fp);
    for (;fp >= bottom && fp < top; fp = *((uint64 *) (fp - 16))) {
        printf("%p\n", *((uint64 *) (fp - 8)));    // 输出当前栈中返回地址
    }
}
```
在 `panic` 函数中加上对它的调用：

```c
void
panic(char *s)
{
  ...
  backtrace();
  panicked = 1; // freeze uart output from other CPUs
  for(;;)
    ;
}
```

在 `kernel/defs.h` 中添加该函数声明：

```c
...
void            backtrace(void);
```

在 `kerne/riscv.h` 中添加 `r_sp`函数。

```c
static inline uint64
r_sp()
{
  uint64 x;
  asm volatile("mv %0, sp" : "=r" (x) );
  return x;
}
```

在 `kernel/sysproc.c` 中的 `sys_sleep` 函数中添加该函数调用：

```c
void sys_sleep(void){
    ...
    backtrace();
    ...
}
```
这样就好了。

## Alarm (hard)
### Question requirements

> In this exercise you'll add a feature to xv6 that periodically alerts a process as it uses CPU time. This might be useful for compute-bound processes that want to limit how much CPU time they chew up, or for processes that want to compute but also want to take some periodic action. More generally, you'll be implementing a primitive form of **user-level interrupt/fault handlers**; you could use something similar to handle page faults in the application, for example. Your solution is correct if it passes `alarmtest` and `usertests`.


> You should add a new `sigalarm(interval, handler)` system call. If an application calls `sigalarm(n, fn),` then after every **n** "ticks" of CPU time that the program consumes, the kernel should cause application function `fn` to be called. When `fn` returns, the application should resume where it left off. A tick is a fairly arbitrary unit of time in xv6, determined by how often a hardware timer generates interrupts. If an application calls `sigalarm(0, 0)`, the kernel should stop generating periodic alarm calls.
You'll find a file `user/alarmtest.c` in your xv6 repository. Add it to the Makefile. It won't compile correctly until you've added `sigalarm` and `sigreturn` system calls (see below).

### 实验的步骤如下：
这个实验本质上就是，当一个进程运行的时候，设置一个 **n-ticks**, 然后这个进程会产生时钟中断。当发生 **n** 次时钟中断时，就会进入到 `handler()` 函数处理流程。**这个实验可以很好的控制一个进程的时钟中断次数（CPU 时间）。**

### 1、在 struct proc in `kernel/proc.h` 中添加字段

```c
struct proc {
  // ......
  int alarm_interval;          // n-ticks
  void(*alarm_handler)();      // Alarm handler
  int alarm_ticks;             // How many ticks left before next alarm goes off
  struct trapframe *alarm_trapframe;  // handler 之前要保存现场
  int alarm_goingoff;          // 是否已经处理完并返回？
};
```

### 2、在allocproc() in `kernel/proc.c`中初始化

```c
static struct proc*
allocproc(void)
{
  // ......

found:
  p->pid = allocpid();
  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }
  // Allocate a trapframe page for alarm_trapframe.
  if((p->alarm_trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }
  p->alarm_interval = 0;
  p->alarm_handler = 0;
  p->alarm_ticks = 0;
  p->alarm_goingoff = 0;
  return p;
}
```

### 3、实现 usertrap() in `kernel/trap.c` 的实验功能

```c
void
usertrap(void)
{
  ...
  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2){
    if(p->alarm_interval){  // 如果设置了'定时'
      if(--p->alarm_ticks <= 0) { // 如果剩余'时间'不足,进入 handler()
        if(!p->alarm_goingoff){   // 确保没有被 handler()处理过
          p->alarm_ticks = p->alarm_interval; // 恢复 alarm_ticks
          *p->alarm_trapframe = *p->trapframe; // 拷贝,准备进入 handler()
          p->trapframe->epc = (uint64)p->alarm_handler; // 跳到handler()
          p->alarm_goingoff = 1;  // 表示 handler()已经执行过了
        }
      }
    }
    yield();
  }
...
}
```
这样，这个实验就算做完了。最后，再完善下一些必要和不必要的细节。
### 4、补充其它函数
这两个函数是必要的，因为这是获取参数以及设置 `struct proc`字段的过程，它是实验通过的前提：
```c
// lab4-3
uint64 sys_sigalarm(void) {
  int n;  // get param n-ticks
  uint64 fn; // get param handler()
  argint(0, &n);
  argaddr(1, &fn);

  // set params
  struct proc *p = myproc();
  p->alarm_interval = n;
  p->alarm_handler = (void(*)())fn;
  p->alarm_ticks = n; // left equils n of course before start!

  return 0;
}
uint64 sys_sigreturn(void) {
  struct proc *p = myproc();
  *p->trapframe = *p->alarm_trapframe;
  p->alarm_goingoff = 0;
  return p->alarm_trapframe->a0; // return the return value of handler(), or cannot pass the test3()
}
```
回收进程：
```c
static void
freeproc(struct proc *p)
{
  ...
  // lab4-3
  if(p->alarm_trapframe)
    kfree((void*)p->alarm_trapframe);
  // lab4-3
  p->alarm_interval = 0;
  p->alarm_handler = 0;
  p->alarm_ticks = 0;
  p->alarm_goingoff = 0;

  p->state = UNUSED;

}
```

这个实验的难度比之前见到的任何一个实验都难且有意思，**相当于手动实现了一次进程切换**，是内核陷入的低级版本。

*全文完，感谢阅读。*







