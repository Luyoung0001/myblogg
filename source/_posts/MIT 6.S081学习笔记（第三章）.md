---
title: MIT 6.S081学习笔记（第三章）
date: 2023-11-22 17:19:13
tags: 学习 笔记 OS xv6
categories: OS
---

<!--more-->

## 〇、前言
本文主要完成**MIT 6.S081** 实验三：**page tables**。
开始之前，切换分支：
```bash
 $ git fetch
 $ git checkout pgtbl
 $ make clean
```
## 一、Speed up system calls (easy)
这个实验比底下两个都难，但是它的难度是简单？
### Question requirements

> Some operating systems (e.g., Linux) speed up certain system calls by sharing data in a read-only region between userspace and the kernel. This eliminates the need for kernel crossings when performing these system calls. To help you learn how to insert mappings into a page table, your first task is to implement this optimization for the getpid() system call in xv6.

这个实验就是让我来加速 `getpid()` 这个系统调用。

如果用户态调用系统调用（比如`getpid()`），那么就需要切换到内核态，这中间就会带来开销。对于加速系统调用，思路就是不让它切换到核心态，仍然能运行一些核心态才能运行的操作，比如获取某个进程的 pid（也就是这节的 `getpid()`，这当然是安全的无关紧要的越权，因为它只是读取 **PTE_R**，不修改 **PTE_W**，也不执行 **PTE_X**）。

在这里我们使用 `ugetpid(void)` 进行代替 `getpid()`：

```c
#ifdef LAB_PGTBL
int
ugetpid(void)
{
  struct usyscall *u = (struct usyscall *)USYSCALL;
  return u->pid;
}
#endif
```
可以看到它直接操作 `USYSCALL`，本质上是一个指针，进行强转之后，直接返回 `u->pid`。

所以只要为每一个进程分配一个 **USYSCALL** 页 ，这个页程序直接可以在用户态读取它。

```c
#ifdef LAB_PGTBL
#define USYSCALL (TRAPFRAME - PGSIZE)
struct usyscall {
  int pid;  // Process ID
};
#endif
```
这个结构已经被定义好了，我们只需要将它放到进程结构体中：

```c
struct proc{
    ...
    struct usyscall *usyscall;  
    ...
}
```
为它分配一定的空间，这个在初始化中就可以完成了：

```c
static struct proc * allocproc(void){
    ...
    found:
        ...
    // Allocate a speed up syscall page
    if ((p->usyscall = (struct usyscall *)kalloc()) == 0) {
        freeproc(p);
        release(&p->lock);
        return 0;
    }
    // An empty user page table.
    p->pagetable = proc_pagetable(p);
    ...
}
```
到此，完成了基本工作：
- 定义一个我们需要的结构体（已经定义好）；
- 将它增加到 `struct proc`字段；
- 为这个新的字段分配空间。

现在我们就要把这个 `USYSCALL` 页面映射到进程的页表上（建立 **PTE**，并设置 **PTE_R** 、**PTE_U** 标记 ）：

```c
pagetable_t proc_pagetable(struct proc *p){
    ...
    if (mappages(pagetable, USYSCALL, PGSIZE, (uint64)(p->usyscall),
                 PTE_R | PTE_U) < 0) {
        uvmunmap(pagetable, TRAMPOLINE, 1, 0);
        uvmunmap(pagetable, TRAPFRAME, 1, 0);
        uvmfree(pagetable, 0);
        return 0;
    }
    return pagetable;
}
```
进程在启动的时候，我们自然要对这个新的字段要赋值，也就是初始化：

```c
static struct proc * allocproc(void){
    ...
    p->usyscall->pid = p->pid;
    return p;
}
```
初始化之后，这样用户进程在用户态直接调用 `ugetpid()`，就直接获取 `struct proc` 字段就好了，不用再陷入了。

我们来看看测试程序是怎么判断的：

```c
void
ugetpid_test()
{
  int i;

  printf("ugetpid_test starting\n");
  testname = "ugetpid_test";

  for (i = 0; i < 64; i++) {
    int ret = fork();
    if (ret != 0) {
      wait(&ret);
      if (ret != 0)
        exit(1);
      continue;
    }
    if (getpid() != ugetpid())
      err("missmatched PID");
    exit(0);
  }
  printf("ugetpid_test: OK\n");
}
```
很明显，`if (getpid() != ugetpid())` 分别**陷入**和 **不陷入** 获取 `pid`，通过对比给结果。

那么有一个问题，进程在结束的时候，也应该将 PTE 释放掉：

```c
static void freeproc(struct proc*p){
    ...
    if (p->usyscall) kfree((void *)p->usyscall);
    p->usyscall = 0;
}
```
以及在 `proc_freepagetable()` 中解除映射，不然会报错（因为页表已经失效了）：

```c
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  ...
  uvmunmap(pagetable, USYSCALL, 1, 0);
  uvmfree(pagetable, sz);
}
```
运行一下，没什么问题：

```bash
******:~/xv6-labs-2023# ./grade-lab-pgtbl ugetpid
make: 'kernel/kernel' is up to date.
== Test pgtbltest == (2.4s) 
== Test   pgtbltest: ugetpid == 
  pgtbltest: ugetpid: OK 
```
### Which other xv6 system call(s) could be made faster using this shared page? Explain how.

任何直接或间接调用 `copyout()` 的系统调用都会被加速，因为它节省了复制数据的时间。此外，纯粹用于信息检索的系统调用，比如本节中提到的 `getpid()`，也会更快。**这是因为不再需要陷入操作系统，对应的数据可以在用户模式下读取**。

## 二、Print a page table (easy)
### Question requirements

> Define a function called vmprint(). It should take a pagetable_t argument, and print that pagetable in the format described below. Insert if(p->pid==1) vmprint(p->pagetable) in exec.c just before the return argc, to print the first process's page table. You receive full credit for this part of the lab if you pass the pte printout test of make grade.

### Some hints:

- You can put vmprint() in `kernel/vm.c`.
- Use the macros at the end of the file `kernel/riscv.h`.
- The function freewalk may be inspirational.
- Define the prototype for vmprint in `kernel/defs.h` so that you can call it from `exec.c`.
- Use %p in your printf calls to print out full 64-bit hex PTEs and addresses as shown in the example.


很简单，只需要按照提示做，就可以了。先在 `exec.c`加入：

```c
int exec(char *path, char **argv){
  ...
  proc_freepagetable(oldpagetable, oldsz);
  // 增加的代码
  if(p->pid==1) 
    vmprint(p->pagetable);
  return argc; // this ends up in a0, the first argument to main(argc, argv)

 bad:
 ...
}
```
思路就是递归遍历**PTE**以及标记位 **PTE_V**，在 `vm.c`加入：

```c
void printpgtb(pagetable_t pagetable, int depth){
  // 遍历
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V){
      printf("..");
      for(int j=0;j<depth;j++) {
        printf(" ..");
      }
      printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
      // 如果页表条目有效且没有读取、写入或执行权限，则继续递归调用 printpgtb 函数，打印下一级页表的内容
      if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0) {
        uint64 child = PTE2PA(pte);
        printpgtb((pagetable_t)child, depth+1);
      }
    }
  }
}

void vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);
  printpgtb(pagetable, 0);
}
```
也别忘了在 `defs.h`加函数声明：

```c
// vm.c
...
int             copyinstr(pagetable_t, char *, uint64, uint64);
void            vmprint(pagetable_t);
```

## Detect which pages have been accessed (hard)
### Question requirements
> Some garbage collectors (a form of automatic memory management) can benefit from information about which pages have been accessed (read or write). In this part of the lab, you will add a new feature to xv6 that detects and reports this information to userspace by inspecting the access bits in the RISC-V page table. The RISC-V hardware page walker marks these bits in the PTE whenever it resolves a TLB miss.

### Some hints

- Read `pgaccess_test()` in `user/pgtbltest.c` to see how pgaccess is used.
- Start by implementing `sys_pgaccess()` in `kernel/sysproc.c`.
- You'll need to parse arguments using `argaddr()` and `argint()`.
- For the output bitmask, it's easier to store a temporary buffer in the kernel and copy it to the user (via `copyout()`) after filling it with the right bits.
- It's okay to set an upper limit on the number of pages that can be scanned.
`walk()` in `kernel/vm.c` is very useful for finding the right **PTEs**.
- You'll need to define **PTE_A**, the access bit, in `kernel/riscv.h`. Consult the RISC-V privileged architecture manual to determine its value.
- Be sure to clear **PTE_A** after checking if it is set. Otherwise, it won't be possible to determine if the page was accessed since the last time `pgaccess()` was called (i.e., the bit will be set forever).
- `vmprint()` may come in handy to debug page tables.

### 思路：
- 手动添加访问标志；
- 获取一些参数；
- 利用参数对页表项遍历，对访问标志进行提取，提取到 `bitmask` 中；
- 将 `bitmask`返回。

这个实验和第一个加速系统调用实验类似。首先需要手动设置一些标记 **PTE_A**。
在`kernel/riscv.h`中加入访问标志：
```c
#define PTE_A (1L << 6)
```
获取参数，需要获取的参数就是要检查的页起始地址、页数、用户空间地址（答案返回的地方）。

```c
static uint64
argraw(int n)
{
  struct proc *p = myproc();
  switch (n) {
  case 0:
    return p->trapframe->a0;
  case 1:
    return p->trapframe->a1;
  case 2:
    return p->trapframe->a2;
  case 3:
    return p->trapframe->a3;
  case 4:
    return p->trapframe->a4;
  case 5:
    return p->trapframe->a5;
  }
  panic("argraw");
  return -1;
}
```
`argraw()`的作用就是获取用户进程`trap`内核之前的寄存器的信息。`a0`保存的是用户进程的起始地址（虚拟地址），`a1` 保存的是进程的页面数。

```c
  uint64 startaddr;
  int npage;
  uint64 useraddr;
  argaddr(0, &startaddr);
  argint(1, &npage);
  argaddr(2, &useraddr);
```
整个函数如下：

```c
int
sys_pgaccess(void)
{
  //  parse arguments using argaddr() and argint().
  uint64 startaddr;
  int npage;
  uint64 useraddr;
  argaddr(0, &startaddr);
  argint(1, &npage);
  argaddr(2, &useraddr);
  uint64 bitmask = 0;
  // 取反
  uint64 complement = ~PTE_A;
  
  struct proc *p = myproc();
  for (int i = 0; i < npage; ++i) {
    pte_t *pte = walk(p->pagetable, startaddr+i*PGSIZE, 0);
    if (*pte & PTE_A) {
      bitmask |= (1 << i);
      // 清空标记
      *pte &= complement;
    }
  }
  // copyout
  copyout(p->pagetable, useraddr, (char *)&bitmask, sizeof(bitmask));
  return 0;
}
```
运行测试：

> == Test pgtbltest == 
$ make qemu-gdb
(3.6s) 
== Test   pgtbltest: ugetpid == 
  pgtbltest: ugetpid: OK 
== Test   pgtbltest: pgaccess == 
  pgtbltest: pgaccess: OK 
== Test pte printout == 
$ make qemu-gdb
pte printout: OK (0.8s) 
== Test answers-pgtbl.txt == 
answers-pgtbl.txt: OK 
== Test usertests == 
$ make qemu-gdb
(8.2s) 
== Test   usertests: all tests == 
  usertests: all tests: OK 
== Test time == 
time: OK 
Score: 46/46

*全文完，感谢阅读。*























