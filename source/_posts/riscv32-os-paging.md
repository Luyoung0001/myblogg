---
title: "RISC-V32 OS 实验（三）：内存分页机制"
date: 2025-03-18 13:03:13
tags:
  - "RISC-V"
  - "paging"
  - "memory management"
categories:
  - "Operating Systems"
---

## 内存的组织

kernel 跑起来之后，要在 kernel 中运行一些程序，这些程序有的会使用堆区、有的仅仅使用栈区。因此，组织这些内存很重要。

初始的组织，要使用链接脚本。链接脚本告诉编译器 OS 的代码段、数据段、bss 等位置在哪里。另外，编译器还提供了可以在 ld 脚本中使用的一些命令，比如一个简单的脚本如下：

```asm
#include "platform.h"
OUTPUT_ARCH( "riscv" )
ENTRY( _start )

MEMORY
{
	ram   (wxa!ri) : ORIGIN = 0x80000000, LENGTH = LENGTH_RAM
}

SECTIONS
{

	.text : {
		PROVIDE(_text_start = .);
		*(.text .text.*)
		PROVIDE(_text_end = .);
	} >ram

	.rodata : {
		PROVIDE(_rodata_start = .);
		*(.rodata .rodata.*)
		PROVIDE(_rodata_end = .);
	} >ram

	.data : {
		. = ALIGN(4096);
		PROVIDE(_data_start = .);
		*(.sdata .sdata.*)
		*(.data .data.*)
		PROVIDE(_data_end = .);
	} >ram

	.bss :{
		PROVIDE(_bss_start = .);
		*(.sbss .sbss.*)
		*(.bss .bss.*)
		*(COMMON)
		PROVIDE(_bss_end = .);
	} >ram

	PROVIDE(_memory_start = ORIGIN(ram));
	PROVIDE(_memory_end = ORIGIN(ram) + LENGTH(ram));

	PROVIDE(_heap_start = _bss_end);
	PROVIDE(_heap_size = _memory_end - _heap_start);
}

```
这段脚本定义了以下几个关键方面：

1.  **目标架构 (OUTPUT_ARCH):**
    * `OUTPUT_ARCH( "riscv" )`：指定目标架构为RISC-V。这意味着链接器将生成适用于RISC-V处理器的可执行文件。

2.  **入口点 (ENTRY):**
    * `ENTRY( _start )`：定义程序的入口点为`_start`符号。这是程序执行的第一个指令的地址。

3.  **内存布局 (MEMORY):**
    * `MEMORY { ram (wxa!ri) : ORIGIN = 0x80000000, LENGTH = LENGTH_RAM }`：定义了一个名为`ram`的内存区域，其起始地址为`0x80000000`，长度由`LENGTH_RAM`宏定义。`wxa!ri`指定了内存区域的属性，包括可写（w）、可执行（x）、地址可访问（a）、只读（!r）和初始化的（i）。

4.  **段 (SECTIONS):**
    * `.text : { ... } >ram`：将`.text`段（包含程序的可执行指令）放置在`ram`内存区域。`PROVIDE`指令定义了`_text_start`和`_text_end`符号，它们分别表示`.text`段的起始和结束地址。
    * `.rodata : { ... } >ram`：将`.rodata`段（包含只读数据）放置在`ram`内存区域。`PROVIDE`指令定义了`_rodata_start`和`_rodata_end`符号，表示`.rodata`段的起始和结束地址。
    * `.data : { ... } >ram`：将`.data`段（包含已初始化的可写数据）放置在`ram`内存区域。`. = ALIGN(4096)`确保`.data`段的起始地址是4096字节对齐的。`PROVIDE`指令定义了`_data_start`和`_data_end`符号，表示`.data`段的起始和结束地址。
    * `.bss :{ ... } >ram`：将`.bss`段（包含未初始化的可写数据）放置在`ram`内存区域。`PROVIDE`指令定义了`_bss_start`和`_bss_end`符号，表示`.bss`段的起始和结束地址。`COMMON`段也被放置在`.bss`段中。
    * `PROVIDE(_memory_start = ORIGIN(ram));`
    * `PROVIDE(_memory_end = ORIGIN(ram) + LENGTH(ram));`
        * 定义了`_memory_start`和`_memory_end`符号，它们分别表示`ram`内存区域的起始和结束地址。
    * `PROVIDE(_heap_start = _bss_end);`
    * `PROVIDE(_heap_size = _memory_end - _heap_start);`
        * 定义了`_heap_start`符号，它表示堆的起始地址，堆紧跟在`.bss`段之后。
        * 定义了`_heap_size`符号，它表示堆的大小，即`ram`内存区域的结束地址减去堆的起始地址。

这个链接器脚本定义了RISC-V程序的内存布局，包括代码、只读数据、已初始化数据、未初始化数据和堆的放置位置，以及相关的符号，这些符号可以在程序中使用。

上面定义了一些符号，比如 `_heap_start`，尽管这些符号可以直接在 C文件中直接使用，但是仍然存在一些问题。比如，链接器脚本主要用于控制链接过程，它定义的符号本质上是内存地址和大小的标识，而不是 C 语言中的变量。

因此，我们必须把这些符号重新定义一遍，存放在 .rodata 中：

```asm
#define SIZE_PTR .word

.section .rodata
.global HEAP_START
HEAP_START: SIZE_PTR _heap_start

.global HEAP_SIZE
HEAP_SIZE: SIZE_PTR _heap_size

.global TEXT_START
TEXT_START: SIZE_PTR _text_start
...
```

这样，我们就可以在 C语言中使用这些全局符号了：

```C
extern ptr_t TEXT_START;
extern ptr_t TEXT_END;
...

ptr_t _heap_start_aligned = _align_page(HEAP_START);
...
```

## 简单的页面分配

有了简单的内存布局之后，我们就可以在 heap 上划分 page 了。

每一个 page 要由另一个区域指明是否处于已分配，这个区域位于 heap_start，之后才是真正的 paga 区。

因此目前需要考虑的事情有：
- page 多大；
- 怎么设计数据结构来指明 page 是否空闲；
- page 分配的方法；
- page 释放的方法；

经过考虑，page 如下：

```C
static ptr_t _alloc_start = 0;
static ptr_t _alloc_end = 0;
static uint32_t _num_pages = 0;

#define PAGE_SIZE 4096
#define PAGE_ORDER 12

#define PAGE_TAKEN (uint8_t)(1 << 0)
#define PAGE_LAST  (uint8_t)(1 << 1)

/*
 * Page Descriptor
 * flags:
 * - bit 0: flag if this page is taken(allocated)
 * - bit 1: flag if this page is the last page of the memory block allocated
 */
struct Page {
	uint8_t flags;
};
```

因此，首先，page 初始化：

```C
void page_init() {
    // 像高地址对齐到 4KB
	ptr_t _heap_start_aligned = _align_page(HEAP_START);
    // 存放 page 的信息的页面个数
	uint32_t num_reserved_pages = LENGTH_RAM / (PAGE_SIZE * PAGE_SIZE);

    // 真正的可分配的 page 的个数
	_num_pages = (HEAP_SIZE - (_heap_start_aligned - HEAP_START))/ PAGE_SIZE - num_reserved_pages;

    // 打印关键信息
	printf("HEAP_START = %p(aligned to %p), HEAP_SIZE = 0x%lx,\n"
	       "num of reserved pages = %d, num of pages to be allocated for heap = %d\n",
	       HEAP_START, _heap_start_aligned, HEAP_SIZE,
	       num_reserved_pages, _num_pages);

    // 这里直接在 HEAP_START 开始清除标记，因为这里操作的是 pages 之前的区域
	struct Page *page = (struct Page *)HEAP_START;
	for (int i = 0; i < _num_pages; i++) {
		_clear(page);
		page++;
	}

	_alloc_start = _heap_start_aligned + num_reserved_pages * PAGE_SIZE;
	_alloc_end = _alloc_start + (PAGE_SIZE * _num_pages);

	printf("TEXT:   %p -> %p\n", TEXT_START, TEXT_END);
	printf("RODATA: %p -> %p\n", RODATA_START, RODATA_END);
	printf("DATA:   %p -> %p\n", DATA_START, DATA_END);
	printf("BSS:    %p -> %p\n", BSS_START, BSS_END);
	printf("HEAP:   %p -> %p\n", _alloc_start, _alloc_end);
}
```

接着就是分配算法了，当遇到分配个数为 n 的时候，我们必须找出连续的 n 个 page：

```C
void *page_alloc(int npages){
	int found = 0;
	struct Page *page_i = (struct Page *)HEAP_START;
    // 从第一个开始扫描
	for (int i = 0; i <= (_num_pages - npages); i++) {
        // 如果找到了，那么就继续向后探索 n 个连续的区域
		if (_is_free(page_i)) {
			found = 1;

			struct Page *page_j = page_i + 1;
			for (int j = i + 1; j < (i + npages); j++) {
				if (!_is_free(page_j)) {
                    // 如果连续失败，直接退出，继续扫描
					found = 0;
					break;
				}
				page_j++;
			}
			/*
			 * get a memory block which is good enough for us,
			 * take housekeeping, then return the actual start
			 * address of the first page of this memory block
			 */
			if (found) {
            // 如果找到了，那么就打标记
				struct Page *page_k = page_i;
				for (int k = i; k < (i + npages); k++) {
					_set_flag(page_k, PAGE_TAKEN);
					page_k++;
				}
				page_k--;
				_set_flag(page_k, PAGE_LAST);
                // 返回 pages 中的找到的首地址
				return (void *)(_alloc_start + i * PAGE_SIZE);
			}
		}
		page_i++;
	}
	return NULL;
}
```

释放 page 也很简单，传入一个内存地址后，比如 p。计算出 p 的标记 page，然后开始撤销标记，一直到标记 page 的 last_tag 位：

```C
void page_free(void *p){
	if (!p || (ptr_t)p >= _alloc_end) {
		return;
	}
	struct Page *page = (struct Page *)HEAP_START;
	page += ((ptr_t)p - _alloc_start)/ PAGE_SIZE;
	while (!_is_free(page)) {
		if (_is_last(page)) {
			_clear(page);
			break;
		} else {
			_clear(page);
			page++;;
		}
	}
}

```
测试一下：
```C

void page_test(){
	void *p = page_alloc(2);
	printf("p = %p\n", p);
	page_free(p);

	void *p2 = page_alloc(3);
	printf("p2 = %p\n", p2);
	// page_free(p2);

	void *p3 = page_alloc(4);
	printf("p3 = %p\n", p3);
}
```

```bash
Hello, RVOS!
HEAP_START = 0x800033f4(aligned to 0x80004000), HEAP_SIZE = 0x07ffcc0c,
num of reserved pages = 8, num of pages to be allocated for heap = 32756
TEXT:   0x80000000 -> 0x80002d28
RODATA: 0x80002d28 -> 0x80002ee5
DATA:   0x80003000 -> 0x80003000
BSS:    0x80003000 -> 0x800033f4
HEAP:   0x8000c000 -> 0x88000000
p = 0x8000c000
p2 = 0x8000c000
p3 = 0x8000f000

```
可以看到，是符合预期的。












