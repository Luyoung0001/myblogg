---
title: MIT 6.S081学习笔记（第九章）
date: 2023-12-23 22:22:25
tags: 学习 笔记 xv6 OS
categories: OS
---

<!--more-->

## 〇、前言
本文主要完成 **MIT 6.S081** 实验  **mmap** 。
开始之前，切换分支：

```bash
  $ git fetch
  $ git checkout mmap
  $ make clean
```

## Lab: mmap (hard)

### Question requirements

> The mmap and munmap system calls allow UNIX programs to exert detailed control over their address spaces. They can be used to share memory among processes, to map files into process address spaces, and as part of user-level page fault schemes such as the garbage-collection algorithms discussed in lecture. **In this lab you'll add mmap and munmap to xv6, focusing on memory-mapped files.**


> You should implement enough mmap and munmap functionality to make the **mmaptest** test program work. If **mmaptest** doesn't use a mmap feature, you don't need to implement that feature.

### Here are some hints:

>- Start by adding **_mmaptest** to **UPROGS**, and **mmap** and **munmap** system calls, in order to get `user/mmaptest.c` to compile.
>- Fill in the page table lazily, in response to page faults. That is, **mmap** should not allocate physical memory or read the file. Instead, **do that in page fault handling code in (or called by) usertrap**, as in the **lazy page allocation lab**. The reason to be lazy is to ensure **that mmap of a large file is fast, and that mmap of a file larger than physical memory is possible.**
>- Keep track of what **mmap** has mapped for each process. Define a structure corresponding to the **VMA** (virtual memory area) described in Lecture 15, recording the **address, length, permissions, file, etc**. for a virtual memory range created by mmap.
>- Since the xv6 kernel doesn't have a memory allocator in the kernel, it's OK to declare a **fixed-size array of VMAs** and allocate from that array as needed. **A size of 16 should be sufficient.**
>- Implement mmap: find an unused region in the process's address space in which to map the file, and add a VMA to the process's table of mapped regions. **The VMA should contain a pointer to a struct file for the file being mapped**; mmap should increase the file's reference count so that the structure doesn't disappear when the file is closed (hint: see filedup). Run mmaptest: the first mmap should succeed, **but the first access to the mmap-ed memory will cause a page fault and kill mmaptest.**
>- Add code to cause a page-fault in a **mmap-ed** region to allocate a page of physical memory, read 4096 bytes of the relevant file into that page, and map it into the user address space. Read the file with **readi**, which takes an offset argument at which to read in the file (but you will have to lock/unlock the inode passed to readi). Don't forget to set the permissions correctly on the page. Run **mmaptest**; it should get to the first munmap.
>Implement munmap: find the VMA for the address range and unmap the specified pages (hint: use **uvmunmap**). If munmap removes all pages of a previous mmap, it should decrement the reference count of the corresponding struct file. If an unmapped page has been modified and the file is mapped **MAP_SHARED**, write the page back to the file. Look at filewrite for inspiration.
> - Ideally your implementation would only write back **MAP_SHARED** pages that the program actually modified. The dirty bit (D) in the RISC-V PTE indicates whether a page has been written. However, **mmaptest** does not check that non-dirty pages are not written back; thus you can get away with writing pages back without looking at D bits.
> Modify exit to unmap the process's mapped regions as if munmap had been called. Run **mmaptest**; **mmap_test** should pass, but probably not **fork_test**.
> - Modify fork to ensure that the child has the same mapped regions as the parent. Don't forget to increment the reference count for a VMA's struct file. In the page fault handler of the child, it is OK to allocate a new physical page instead of sharing a page with the parent. The latter would be cooler, but it would require more implementation work. Run **mmaptest**; it should pass both **mmap_test** and **fork_test**.

### Answer

#### 1、准备工作
我们需要先添加必要的声明。

在 `kernel/syscall.h`：

```c
#define SYS_close  21
#define SYS_mmap   22
#define SYS_munmap 23
```

在 `kernel/syscall.c` ：

```c
extern uint64 sys_close(void);
extern uint64 sys_mmap(void);
extern uint64 sys_munmap(void);
...
static uint64 (*syscalls[])(void) = {
...
[SYS_close]   sys_close,
[SYS_mmap]    sys_mmap,
[SYS_munmap]  sys_munmap,
};
```

在 `user/usys.pl`：

```c
entry("uptime");
entry("mmap");
entry("munmap");
```
在 `user/user.h`：

```c
void *mmap(void*,int,int,int,int,int);
int munmap(void*,int);
```
系统调用声明处理好后，接下来就得在内核中实现它。

这里涉及的操作系统基本概念是虚拟内存，**mmap** 用来将文件映射到内存上，**munmap** 将文件从某个进程的内存空间移除。

#### 2、代码实现

为了尽量使得 **map** 的文件使用的地址空间不要和进程所使用的地址空间产生冲突，将 **mmap** 映射进来的文件 **map** 到尽可能高的位置，也就是刚好在 **trapframe** 下面。并且若有多个 **mmap** 的文件，则向下生长，修改`kernel/memlayout.h`：

```c
...
//   TRAMPOLINE (the same page as in the kernel)
#define TRAPFRAME (TRAMPOLINE - PGSIZE)
#define MMAPEND TRAPFRAME
```

接下来定义 **vma** 结构体，其中包含了 **mmap** 映射的内存区域的各种必要信息，比如开始地址、大小、所映射文件、文件内偏移以及权限等，然后在 **proc** 结构体末尾为每个进程加上 **16** 个 **vma** 空槽（按照 **hints** 中建议）：

```c
// kernel/proc.h
struct vma {
  int valid;
  uint64 vastart;
  uint64 sz;
  struct file *f;
  int prot;
  int flags;
  uint64 offset;
};

#define NVMA 16

// Per-process state
struct proc {
  ...
  char name[16];               // Process name (debugging)
  struct vma vmas[NVMA];       // virtual memory areas
};

```

接下来在 `kernel/sysfile.c` 实现 **mmap** 系统调用，记得在头文件中加上`#include "memlayout.h"`：

```c
// 映射文件
uint64
sys_mmap(void)
{
  uint64 addr, sz, offset;
  int prot, flags, fd; struct file *f;
  // 获取参数
  argaddr(0, &addr);
  argaddr(1, &sz);
  argint(2, &prot);
  argint(3, &flags);
  argfd(4, &fd, &f);
  argaddr(5, &offset);

  // 安全检查
  if((!f->readable && (prot & (PROT_READ)))
     || (!f->writable && (prot & PROT_WRITE) && !(flags & MAP_PRIVATE)))
    return -1;

  sz = PGROUNDUP(sz);

  struct proc *p = myproc();
  struct vma *v = 0;
  uint64 vaend = MMAPEND; 
  for(int i=0;i<NVMA;i++) {
  	// 寻找空槽
    struct vma *vv = &p->vmas[i];
    if(vv->valid == 0) {
      if(v == 0) {
        v = &p->vmas[i];
        v->valid = 1;
      }
    } else if(vv->vastart < vaend) {
      vaend = PGROUNDDOWN(vv->vastart);
    }
  }

  if(v == 0){
    panic("mmap: no free vma");
  }
  // 设置
  v->vastart = vaend - sz;
  v->sz = sz;
  v->prot = prot;
  v->flags = flags;
  v->f = f; // assume f->type == FD_INODE
  v->offset = offset;
  // 增加文件链接计数
  filedup(v->f);
  return v->vastart;
}
```
上面的系统调用非常快，但是此时文件或许并不在内存中，访问的时候，大概率会出现**segment falt**，这时候我们需要再 **usertrap** 中进行处理：

```c
// kernel/trap.c
void
usertrap(void)
{
...
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    uint64 va = r_stval();
    if((r_scause() == 13 || r_scause() == 15)){ // vma 懒分配
      if(!vmatrylazytouch(va)) {
        goto unexpected_scause;
      }
    } else {
      unexpected_scause:
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
      printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
      p->killed = 1;
    }
  }
  ...
  usertrapret();
}
```

在 `kernel/sysfile.c`中实现：
```c
// 找到与传入的虚拟地址匹配的 VMA 结构
struct vma *findvma(struct proc *p, uint64 va) {
  for(int i=0;i<NVMA;i++) {
    struct vma *vv = &p->vmas[i];
    if(vv->valid == 1 && va >= vv->vastart && va < vv->vastart + vv->sz) {
      return vv;
    }
  }
  return NULL;
}

// 这里做的是真正的映射：将文件映射到内存中
int vmatrylazytouch(uint64 va) {
  struct proc *p = myproc();
  // 在本进程中找
  struct vma *v = findvma(p, va);
  if(v == 0) {
 	// 失败
    return 0;
  }
  
  // 分配内存
  void *pa = kalloc();
  if(pa == 0) {
    panic("vmalazytouch: kalloc");
  }
  memset(pa, 0, PGSIZE);
  
  // 从硬盘中将文件读到内存
  begin_op();
  ilock(v->f->ip);
  readi(v->f->ip, 0, (uint64)pa, v->offset + PGROUNDDOWN(va - v->vastart), PGSIZE);
  iunlock(v->f->ip);
  end_op();

  // set appropriate perms, then map it.
  int perm = PTE_U;
  if(v->prot & PROT_READ)
    perm |= PTE_R;
  if(v->prot & PROT_WRITE)
    perm |= PTE_W;
  if(v->prot & PROT_EXEC)
    perm |= PTE_X;

  if(mappages(p->pagetable, va, PGSIZE, (uint64)pa, perm) < 0) {
    panic("vmalazytouch: mappages");
  }
  return 1;
}
```

接下来实现 `munmap()` 系统调用：

```c
// kernel/sysfile.c

uint64
sys_munmap(void)
{
  uint64 addr, sz;
  argaddr(0, &addr);
  argaddr(1, &sz);

  struct proc *p = myproc();
  struct vma *v = findvma(p, addr);
  if(v == 0) {
    return -1;
  }

  // 安全检查
  if(addr > v->vastart && addr + sz < v->vastart + v->sz) {
    return -1;
  }
  // 对齐
  uint64 addr_aligned = addr;
  if(addr > v->vastart) {
    addr_aligned = PGROUNDUP(addr);
  }

  int nunmap = sz - (addr_aligned-addr); // nbytes to unmap
  if(nunmap < 0)
    nunmap = 0;

  vmaunmap(p->pagetable, addr_aligned, nunmap, v); // custom memory page unmap routine for mmapped pages.

// 有重叠
  if(addr <= v->vastart && addr + sz > v->vastart) { 
    v->offset += addr + sz - v->vastart;
    v->vastart = addr + sz;
  }
  v->sz -= sz;

  if(v->sz <= 0) {
    fileclose(v->f);
    v->valid = 0;
  }

  return 0;
}
```

确保要释放的范围不会在 VMA 区域中间“挖洞”，然后根据计算得到的释放起始地址和字节数，调用自定义的 **vmaunmap** 方法进行实际的内存释放操作。这样的设计确保了在释放内存页时对映射的精确控制，以及在需要时将数据写回磁盘。

将释放内存页的操作封装到 `vm.c` 中的 **vmaunmap** 中，这样可以**更好地模块化代码**。这个函数需要考虑脏位，有脏位就需要被写回。

```c
// kernel/riscv.h

#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // 1 -> user can access
#define PTE_G (1L << 5) // global mapping
#define PTE_A (1L << 6) // accessed
#define PTE_D (1L << 7) // dirty
```

在 `kernel/riscv.h` 中设置脏位。
```c
// kernel/vm.c
#include "fcntl.h"
#include "spinlock.h"
#include "sleeplock.h"
#include "file.h"
#include "proc.h"

void
vmaunmap(pagetable_t pagetable, uint64 va, uint64 nbytes, struct vma *v)
{
  uint64 a;
  pte_t *pte;

  for(a = va; a < va + nbytes; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      continue;
      // 这行代码的作用是确保要取消映射的页表项是一个叶子节点，而不是中间节点或其他不应该被取消映射的节点。
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("sys_munmap: not a leaf");
    if(*pte & PTE_V){
      uint64 pa = PTE2PA(*pte);
      // 脏且共享，写回
      if((*pte & PTE_D) && (v->flags & MAP_SHARED)) { 
        begin_op();
        ilock(v->f->ip);
        uint64 aoff = a - v->vastart; // offset relative to the start of memory range
        if(aoff < 0) { // if the first page is not a full 4k page
          writei(v->f->ip, 0, pa + (-aoff), v->offset, PGSIZE + aoff);
        } else if(aoff + PGSIZE > v->sz){  // if the last page is not a full 4k page
          writei(v->f->ip, 0, pa, v->offset + aoff, v->sz - aoff);
        } else { // full 4k pages
          writei(v->f->ip, 0, pa, v->offset + aoff, PGSIZE);
        }
        iunlock(v->f->ip);
        end_op();
      }
      kfree((void*)pa);
      *pte = 0;
    }
  }
}
```

最后需要做的，是在 `proc.c` 中添加处理进程 **vma** 的各部分代码。

- 让 **allocproc** 初始化进程的时候，将 vma 槽都清空；
- **freeproc** 释放进程时，调用 **vmaunmap** 将所有 **vma** 的内存都释放，并在需要的时候写回磁盘；
- **fork** 时，拷贝父进程的所有 **vma**，但是不拷贝物理页（也没必要）。

```c
// kernel/proc.c

static struct proc*
allocproc(void)
{
  // ......

  // Clear VMAs
  for(int i=0;i<NVMA;i++) {
    p->vmas[i].valid = 0;
  }

  return p;
}

// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  
  for(int i = 0; i < NVMA; i++) {
    struct vma *v = &p->vmas[i];
    vmaunmap(p->pagetable, v->vastart, v->sz, v);
  }
  
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}

// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int
fork(void)
{
  ...
  // copy vmas created by mmap.
  // actual memory page as well as pte will not be copied over.
  for(i = 0; i < NVMA; i++) {
    struct vma *v = &p->vmas[i];
    if(v->valid) {
      np->vmas[i] = *v;
      filedup(v->f);
    }
  }
...
  return pid;
}

```
由于 **mmap** 映射的页并不在 **[0, p->sz)** 范围内，所以其页表项在 **fork** 的时候并不会被拷贝。我们只拷贝了 **vma** 项到子进程，这样子进程尝试访问 **mmap** 页的时候，会重新触发懒加载，重新分配物理页以及建立映射。

最后在 `kernel/defs.h`中加入函数声明：

```c
struct vma*     findvma(struct proc *p, uint64 va);
int             vmatrylazytouch(uint64 va);
void            vmaunmap(pagetable_t, uint64, uint64, struct vma *);
```


这样，这个实验就结束了：

```bash
== Test running mmaptest == 
$ make qemu-gdb
(6.6s) 
== Test   mmaptest: mmap f == 
  mmaptest: mmap f: OK 
== Test   mmaptest: mmap private == 
  mmaptest: mmap private: OK 
== Test   mmaptest: mmap read-only == 
  mmaptest: mmap read-only: OK 
== Test   mmaptest: mmap read/write == 
  mmaptest: mmap read/write: OK 
== Test   mmaptest: mmap dirty == 
  mmaptest: mmap dirty: OK 
== Test   mmaptest: not-mapped unmap == 
  mmaptest: not-mapped unmap: OK 
== Test   mmaptest: two files == 
  mmaptest: two files: OK 
== Test   mmaptest: fork_test == 
  mmaptest: fork_test: OK 
== Test usertests == 
$ make qemu-gdb
usertests: OK (102.5s) 
== Test time == 
time: OK 
Score: 140/140
```

## 总结（大模型总结）
这个实验的核心是实现对文件的内存映射，让文件内容可以直接映射到进程的地址空间中，允许对文件内容进行直接的读写操作。在实现 **mmap** 和 **munmap** 这两个系统调用时，需要注意：

1. **Lazy Loading**: 实现了懒加载，即在 **mmap** 中不立即分配物理内存或读取文件内容，而是在发生页错误时根据需要进行操作。这可以提高效率，特别是对于大文件或超过物理内存大小的文件映射。
  
2. **VMA 结构**: 使用 **VMA**（Virtual Memory Area）结构记录映射的文件信息，包括起始地址、大小、权限、文件等信息。每个进程维护一个 **VMA** 数组，用于跟踪其映射的文件区域。

3. **文件写回**: 在 **munmap** 时需要注意，如果文件是共享的（**MAP_SHARED**），并且有脏页（**PTE_D**），则需要将脏页的内容写回文件。

4. **进程 fork**: 在 **fork** 中需要复制父进程的 **VMA**，但不复制物理页。子进程访问 mmap 区域时会触发页错误，执行懒加载。

5. **进程 exit**: 当进程退出时，需要释放所有的 **VMA** 区域，并根据需要将修改过的文件页写回文件。

实验的目标是确保实现 **mmap** 和 **munmap** 功能，使得测试程序 **mmaptest** 能够成功运行，并通过一系列测试，包括对 **mmap** 和 **munmap** 的功能、文件的读写、进程 **fork** 和 **exit** 的正确性检验等。












