---
title: MIT 6.S081学习笔记（第五章）
date: 2023-12-06 11:33:52
tags:
    - 学习
    - 笔记
    - 操作系统
    - MIT 6.S081
categories:
    - OS
    - 系统编程
    - Unix/Linux
---

<!--more-->

## 〇、前言
本文主要完成**MIT 6.S081** 实验五：**Copy-on-Write Fork for xv6**。
开始之前，切换分支：

```bash
$ git fetch
$ git checkout cow
$ make clean
```
## 一、问题
### Question requirements

> The `fork()` system call in **xv6** copies all of the parent process's user-space memory into the child. If the parent is large, copying can take a long time. Worse, the work is often largely wasted: `fork()` is commonly followed by `exec()` in the child, which discards the copied memory, usually without using most of it. On the other hand, if both parent and child use a copied page, and one or both writes it, the copy is truly needed.

### The solution

> Your goal in implementing copy-on-write (COW) `fork()` is to defer allocating and copying physical memory pages until the copies are actually needed, if ever.
COW `fork()` creates just a page table for the child, with **PTEs** for user memory pointing to the parent's physical pages. COW `fork()` marks all the user **PTEs** in both parent and child as read-only. When either process tries to write one of these COW pages, the CPU will force a page fault. The kernel page-fault handler detects this case, allocates a page of physical memory for the faulting process, copies the original page into the new page, and modifies the relevant **PTE** in the faulting process to refer to the new page, this time with the **PTE** marked writeable. When the page fault handler returns, the user process will be able to write its copy of the page.
COW `fork()` makes freeing of the physical pages that implement user memory a little trickier. A given physical page may be referred to by multiple processes' page tables, and should be freed only when the last reference disappears. In a simple kernel like xv6 this bookkeeping is reasonably straightforward, but in production kernels this can be difficult to get right; see, for example, Patching until the **COWs** come home.

### Here's a reasonable plan of attack.

>- Modify `uvmcopy()` to map the parent's physical pages into the child, instead of allocating new pages. Clear **PTE_W** in the **PTEs** of both child and parent for pages that have **PTE_W** set.
>- Modify `usertrap()` to recognize page faults. When a write page-fault occurs on a **COW** page that was originally writeable, allocate a new page with `kalloc()`, copy the old page to the new page, and install the new page in the **PTE** with **PTE_W** set. Pages that were originally read-only (not mapped PTE_W, like pages in the text segment) should remain read-only and shared between parent and child; a process that tries to write such a page should be killed.
>- Ensure that each physical page is freed when the last PTE reference to it goes away -- but not before. A good way to do this is to keep, for each physical page, a "reference count" of the number of user page tables that refer to that page. Set a page's reference count to one when `kalloc()` allocates it. Increment a page's reference count when fork causes a child to share the page, and decrement a page's count each time any process drops the page from its page table. `kfree()` should only place a page back on the free list if its reference count is zero. It's OK to to keep these counts in a fixed-size array of integers. You'll have to work out a scheme for how to index the array and how to choose its size. For example, you could index the array with the page's physical address divided by 4096, and give the array a number of elements equal to highest physical address of any page placed on the free list by `kinit()` in kalloc.c. Feel free to modify kalloc.c (e.g., `kalloc()` and `kfree()`) to maintain the reference counts.
>- Modify `copyout()` to use the same scheme as page faults when it encounters a **COW** page.


### Some hints

> - It may be useful to have a way to record, for each PTE, whether it is a COW mapping. You can use the **RSW** (reserved for software) bits in the **RISC-V** PTE for this.
>- `usertests -q` explores scenarios that cowtest does not test, so don't forget to check that all tests pass for both.
>- Some helpful macros and definitions for page table flags are at the end of `kernel/riscv.h`.
If a **COW** page fault occurs and there's no free memory, the process should be killed.


## 二、答案
这个实验的思路已经在上面提到过，题目给的解决思路相当清晰，只需要照着做就可以。
### 1、在 kernel/vm.c 中修改`uvmcopy()`
如果这个 **page** 是只读的，那么就不用管，直接映射在父进程就好了。如果这个 **page** 是可以写的，那么就需要将它标记为不可写（为了引发段错误，从而进一步处理），并且打上 **COW** 标记。
在`riscv.h`中加上**COW**标记定义：

```c
#define PTE_C (1L << 8) // copy-on-write
```

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  int flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    if((pa = PTE2PA(*pte)) == 0)
      panic("uvmcopy: address should exist");

    if(*pte & PTE_W){ // 如果可以写变成COW页
      *pte |= PTE_C;
      *pte &= ~PTE_W;
    }

    flags = PTE_FLAGS(*pte);
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
      printf("uvmcopy: mappages\n");
      goto err;
    }
    refcnt_inc((void *) pa);  // 增加引用计数。
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```
### 2、在 kernel/trap.c 中修改 `usertrap()`

```c
else if(r_scause() == 15) {
    // 找到当前错误页
    uint64 addr = r_stval();
    // 进行cow处理
    if(cowalloc(p->pagetable, addr) < 0){
      printf("alloc user page fault addr=%p\n", addr);
      setkilled(p);
    }
}
```
### 3、在kernel/kalloc.c中设置引用计数机制
先在 `riscv.h` 中定义一些宏：

```c
// cow
#define PG2REFIDX(_pa) ((((uint64)_pa) - KERNBASE) / PGSIZE)
#define MX_PGIDX PG2REFIDX(PHYSTOP)
#define PG_REFCNT(_pa) pg_refcnt[PG2REFIDX((_pa))]
```
建立引用计数机制：

```c
// cow
int pg_refcnt[MX_PGIDX];
struct {
  struct spinlock lock;
  int ref[PHYSTOP/PGSIZE];
} pageref;

void
refcnt_inc(void *pa){
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("refcnt_inc");
  acquire(&pageref.lock);
  PG_REFCNT(pa)++;
  release(&pageref.lock);
}

void
refcnt_init(void *pa){
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("refcnt_init");
  acquire(&pageref.lock);
  PG_REFCNT(pa) = 1;
  release(&pageref.lock);
}

void
refcnt_dec(void *pa){
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("refcnt_dec");
  acquire(&pageref.lock);
  PG_REFCNT(pa)--;
  release(&pageref.lock);
}

int
get_refcnt(void *pa){
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("get_refcnt");
  acquire(&pageref.lock);
  int ret = PG_REFCNT(pa);
  release(&pageref.lock);
  return ret;
}
```

### 4、在 kernel/vm.c 中实现 `cowalloc()`
因为我们还需要在 `copyout()`中使用这个函数，因此我们要把这个函数的定义加入到 `defs.h`中：

```c
int cowalloc(pagetable_t pagetable, uint64 va);
```

思路就是当引用计数为 **1** （父进程初始时，对 **page** 的引用计数就是 **1**，因为 `uvmcopy()` 自增了它）时不变，直接恢复父进程的**写权限**。当引用计数大于 **1** 时，创建新的页表，赋值内容到新的页表，建立新的映射，更新标记位，并且去掉旧的映射。

```c
int
cowalloc(pagetable_t pagetable, uint64 va)
{
	// 进行一些检查
  if(va >= MAXVA)
    return -1;

  uint64 pa, new_pa, va_rounded;
  int flags;
  pte_t *pte = walk(pagetable, va, 0);

  if( pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0)
    return -1;

  flags = PTE_FLAGS(*pte);
  pa = PTE2PA(*pte);
  va_rounded = PGROUNDDOWN(va);
  // 安全检查
  if(!(*pte & PTE_C) && !(*pte & PTE_W))
    return -1;
  if( (*pte & PTE_W) || !(*pte & PTE_C))
    return 0;
  if(get_refcnt((void *) pa) > 1){
    if((new_pa = (uint64) kalloc()) == 0) // 申请一个物理页。
      panic("cowalloc: kalloc");
    memmove((void *)new_pa, (const void *) pa, PGSIZE);  // 将原物理页中的内容复制到新物理页中。
    uvmunmap(pagetable, va_rounded, 1, 1);  // 解除虚拟页和物理页的映射关系。
    flags &= ~PTE_C;  // 清除页表项中的 COW 位。
    flags |= PTE_W;  // 设置页表项中的 W 位。
    if(mappages(pagetable, va_rounded, PGSIZE, new_pa, flags) != 0){// 建立新的虚拟页和物理页的映射关系。
      kfree((void *)new_pa);
      return -1;
    }
    return 0;
  } else if(get_refcnt((void *) pa) == 1){
    *pte |= PTE_W;
    *pte &= ~PTE_C;
    return 0;
  }

  return -1;
}
```

### 5、 在 kernel/vm.c 中修改 `copyout()` 函数
从内核中将数据复制到用户 **page** 时，必须确保用户页是独立存在的。
```c
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    if(cowalloc(pagetable, va0) < 0)
      return -1;
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

### 6、修改其它函数
`kinit()`:
将引用计数数组设为 **0**：
```c
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&pageref.lock, "ref_cnt");
  memset(pageref.ref, 0, sizeof(pageref.ref));
  freerange(end, (void*)PHYSTOP);
}

```
`kfree()`:
引用计数大于 **1** 时自减，然后返回；等于 **1** 时，准备释放工作。释放前，置入垃圾信息防止泄露信息，然后放到 `freelist` 待用。
```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  if(get_refcnt(pa) > 1){
    refcnt_dec(pa);
    return;
  }

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```
`kalloc()`:
对于新分配的 **page**，将引用计数设为 **1**：
```c
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r) {
    kmem.freelist = r->next;
    refcnt_init((void*)r);
  }
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

因为这几个函数需要在其它地方使用，也需要修改 `defs.h`中：

```c
void            refcnt_inc(void *pa);
void            refcnt_dec(void *pa);
int             get_refcnt(void *pa);
```

实验到此完成。

## 三、测试

```bash
== Test running cowtest ==
$ make qemu-gdb
(13.4s)
== Test   simple ==
  simple: OK
== Test   three ==
  three: OK
== Test   file ==
  file: OK
== Test usertests ==
$ make qemu-gdb
(11.8s)
    (Old xv6.out.usertests failure log removed)
== Test   usertests: copyin ==
  usertests: copyin: OK
== Test   usertests: copyout ==
  usertests: copyout: OK
== Test   usertests: all tests ==
  usertests: all tests: OK
== Test time ==
time: OK
Score: 110/110
```
## 四、实验总结
该实验主要解决了在 **xv6** 操作系统中，通过实现**Copy-on-Write**（COW）**fork**，减少 `fork()` 系统调用所需的内存复制。这意味着在父进程和子进程之间共享物理页，并延迟物理内存分配和复制，直到有进程对共享的页进行写入操作。

首先，在实现过程中，需要修改并新增了几个关键函数，如`uvmcopy()`来映射父进程的物理页到子进程，以及`cowalloc()`来处理**COW**页的分配。同时，对于用户空间和内核空间之间的数据传递，也需要修改`copyout()` 函数以确保用户页的独立存在。

引入引用计数机制也是关键之举，它确保了物理页被正确释放。通过对物理页的引用计数进行管理，确保了正确的内存释放顺序，避免了可能的内存泄漏和悬空指针问题。


*全文完，感谢阅读。*

