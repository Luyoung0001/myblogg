---
title: "毕业设计记录（5）：mylibc 与硬件平台接口"
date: 2026-01-15 10:03:13
tags:
  - "graduation thesis"
  - "libc"
  - "bare metal"
categories:
  - "Thesis Project"
---

## GCC 自带头文件

GCC 编译器自带了一些基础头文件，例如：

- `<stdint.h>` - 定义了标准整数类型（如 uint8_t、int32_t 等）
- `<stddef.h>` - 定义了 size_t、NULL 等基础类型和宏
- `<stdarg.h>` - 提供了可变参数列表支持（va_list、va_start、va_end 等）

这些库提供了编译器内置的宏定义和类型定义。在裸机系统中可以直接 include 使用，因为它们主要是类型定义和宏，不会引入额外的代码。如果需要，可以通过编译参数（如 `-fno-builtin`）禁用 GCC 的内置函数。

## mylibc

mylibc 是一个轻量级的 C 标准库实现，是 glibc 的精简子集，适用于裸机和嵌入式环境。目前实现了以下头文件的常用函数：

- **ctype.h** - 字符分类和转换函数
- **stdio.h** - 格式化输入输出函数（printf、sprintf、snprintf 等）
- **stdlib.h** - 通用工具函数（atoi、strtol、abs 等）
- **string.h** - 字符串和内存操作函数（strlen、strcpy、memcpy 等）
- **assert.h** - 断言宏

## 与硬件平台的对接

### putchar 函数

`putchar` 是 mylibc 与底层硬件交互的关键接口。通过弱符号（weak symbol）机制，允许用户根据具体硬件平台进行定制：

```C
__attribute__((weak)) int putchar(int c) {
    /* 声明外部 BSP 函数 */
    extern void bsp_uart_putc(uint8_t uart_id, char ch);

    /* 默认使用 UART0 输出 */
    #define DEFAULT_UART_ID 0

    bsp_uart_putc(DEFAULT_UART_ID, (char)c);
    return (unsigned char)c;
}
```

由于使用了 `__attribute__((weak))` 修饰，用户可以在自己的代码中重新定义 `putchar` 函数，以适配不同的输出设备：

- **UART 输出**：通过 `bsp_uart_putc` 实现串口输出（默认实现）
- **LCD 显示**：可以重定义为使用 `lcd_display` 等显示驱动
- **其他设备**：根据实际硬件需求自由定制

这种设计使得 mylibc 具有良好的可移植性，可以方便地适配到不同的硬件平台和操作系统中。


## mylibc 的测试

实现好了后，我需要将 mylibc 中所有的函数进行测试，一般使用断言来测试。比如：

```C
void test_stdio(void) {
    TEST_START("stdio.h");

    char buf[256];
    int ret;

    /* 测试 sprintf - 基础格式化 */
    ret = sprintf(buf, "Hello");
    assert(ret == 5);
    printf("buff:%s\n",buf);
    assert(strcmp(buf, "Hello") == 0);
}
```

`assert()` 是一个宏：
```C
#ifdef NDEBUG
    #define assert(expr) ((void)0)
#else
    #define assert(expr) \
        do { \
            if (!(expr)) { \
                printf("\n*** ASSERTION FAILED ***\n"); \
                printf("File: %s\n", __FILE__); \
                printf("Line: %d\n", __LINE__); \
                printf("Expression: %s\n", #expr); \
                printf("*** TEST FAILED ***\n\n"); \
                while(1); /* 停止程序 */ \
            } \
        } while(0)
#endif
```
宏中将所有的语句塞入了一行，并使用了 `__FILE__`、`__LINE__` 等编译器内置宏来确保打印的断言信息能精准指出是哪里出现了断言错误。而 `printf("Expression: %s\n", #expr);` 直接将表达式字符串化，直接打印出了断言失误的表达式：

```txt
=== Testing stdio.h ===

*** ASSERTION FAILED ***
File: hello.c
Line: 44
Expression: ret == 6
*** TEST FAILED ***
```

当所有的测试通过后，就可以（大概率没问题？）使用 mylibc 了：

```txt
========================================
   mylibc Comprehensive Test Suite
========================================


=== Testing stdio.h ===
buff:Hello
All sprintf/snprintf tests passed!
PASSED

=== Testing string.h - basic ===
All basic string tests passed!
PASSED

=== Testing string.h - search ===
All string search tests passed!
PASSED

=== Testing string.h - memory ===
All memory operation tests passed!
PASSED

=== Testing ctype.h ===
All ctype tests passed!
PASSED

=== Testing stdlib.h ===
All stdlib tests passed!
PASSED

=== Testing Edge cases & stress tests ===
All edge case tests passed!
PASSED

========================================
   Test Summary
========================================
Total tests: 7
Passed:      7
Failed:      0

*** ALL TESTS PASSED! ***
mylibc is working correctly!
========================================
```




