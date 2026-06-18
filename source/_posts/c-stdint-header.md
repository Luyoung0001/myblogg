---
title: "C 标准库笔记：<stdint.h> 的类型设计与价值"
date: 2026-01-03 15:03:13
tags:
  - "C"
  - "standard library"
  - "integer types"
categories:
  - "Programming Languages"
mermaid: true
---

## 前言
在做课程设计的时候，有这样的一个代码片段：

```C
#ifndef MEMORY_HPP
#define MEMORY_HPP

#define MEM_SIZE        (64 * 1024 * 1024)  // 64MB memory
#define MEM_BASE        0x00000000  // CPU reset PC: 0xfffffffc + 4 = 0x00000000

#include <stdint.h>

void sram_init(void);
void sram_cleanup(void);

uint32_t sram_read(uint32_t addr, int size);
void sram_write(uint32_t addr, uint32_t data, uint8_t wen, int size);

#endif
```

可以看到我使用了 uint32_t 数据类型，因此我得使用一个库 `<stdint.h>`，这个库中包含了 uint32_t 的定义。

但是我突发奇想，能不能自己定义：

```C
typedef unsigned int uint32_t;
// or
typedef unsigned long uint32_t;
```
其实是不行的，因为这样写，在 64 位目标平台上编译出的代码可能是 32 位，在 32 位目标平台上编译出的代码可能就是 16 位，这样严重错误了（尽管我很明确我的机器是 64 位的，它不会出错，但不符合设计通用性）。

## \<stdint.h\> 的做法

stdin.h 是这样做的：

```C
#ifdef __INT32_TYPE__

# ifndef __int8_t_defined /* glibc sys/types.h also defines int32_t*/
typedef __INT32_TYPE__ int32_t;
# endif /* __int8_t_defined */

# ifndef __uint32_t_defined  /* more glibc compatibility */
# define __uint32_t_defined
typedef __UINT32_TYPE__ uint32_t;
# endif /* __uint32_t_defined */

# undef __int_least32_t
# define __int_least32_t int32_t
# undef __uint_least32_t
# define __uint_least32_t uint32_t
# undef __int_least16_t
# define __int_least16_t int32_t
# undef __uint_least16_t
# define __uint_least16_t uint32_t
# undef __int_least8_t
# define __int_least8_t int32_t
# undef __uint_least8_t
# define __uint_least8_t uint32_t
#endif /* __INT32_TYPE__ */
```

首先，判断编译器有没有定义`__INT32_TYPE__`，如果定义了，那么继续确认 glibc 是否定义 `__int32_t_defined`，如果没有定义的话，就定义 int32_t，否则就不定义了。虽然这里有点奇怪：它是用 `__int8_t_defined` 来判断是否定义 `__int32_t_defined` 的，因为一旦 glibc 将 `__int8_t_defined` 定义了，那么 `__int32_t_defined` 一定也是定义了的。

如果编译器定义了 `__INT32_TYPE__`，那么在判断 glibc 没有被包含后就自己定义；如果 glibc 被包含了，那么就没必要定义 int32_t 了，因为 glibc 已经被定义过了。后面的 `uint32_t` 定义过程也是如此。

这段代码的设计就是为了同时兼容有 glibc 和无 glibc 的环境，这样就可以保证安全地定义 `uint32_t`、`int32_t`了。

至于后面的宏定义，比如将 __int_least32_t 重新定义了一次，确保它是 32 位的。
