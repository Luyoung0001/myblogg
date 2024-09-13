---
title: 调用 glibc 的技巧
date: 2024-09-12 12:57:40
tags:
    - PA2
    - glibc
    - 编译技巧
categories:
    - ysyx
    - PA2
---

<!--more-->

## 一、一个问题

```
native的IOE是基于SDL库实现的, 这些库很有可能会调用glibc的库函数, 例如malloc()和free(). 但我们自己实现的klib通常不能完美地符合glibc的标准, 因此直觉上看, 如果定义了__NATIVE_USE_KLIB__, 很可能会导致SDL库产生不正确的行为.

不过你会发现, 即使定义了__NATIVE_USE_KLIB__, 也可以正确地在native上执行IOE相关的功能. 实际上, 我们使用了一个小技巧, 使得在定义了__NATIVE_USE_KLIB__的情况下, 避免SDL库调用klib中的函数, 而是调用glibc中的相应函数. 如果屏蔽这个小技巧, 在定义__NATIVE_USE_KLIB__的情况下, native将无法正确运行依赖IOE的程序. 你知道这个小技巧是如何做到的吗?

```

对于这件事情，我刚开始一直在 Makefile 中找答案，还有.ld 链接脚本中看，后面把 Makefile 搞清楚后，我觉得问题应该在源码中，果然被我找到了`ysyx-workbench/abstract-machine/am/src/native/platform.c`：

```C
// save the address of memcpy() in glibc, since it may be linked with klib
  memcpy_libc = dlsym(RTLD_NEXT, "memcpy");
  assert(memcpy_libc != NULL);
```

这里说得很清楚，`memcpy()`可能会被连接到 klib，但并不是我们想要的，我们希望在链接 SDL 的时候，SDL 中的符号能链接到 glibc 而不是 klib，因此在这里做了控制。

那么这是如何实现的呢？

## 二、第一个例子

这个例子设置了一个宏，如果注释了那个宏，那么就会使用默认的库，如果没有注释，那就会使用我自己的库。

`mylib.h`:

```C
//#define __NATIVE_USE_MYLIB__

int printf(const char* s1);

```

`mylib.c`:

```C

#include "mylib.h"

#if defined(__NATIVE_USE_MYLIB__)

int printf(const char* s1) {
    return 0;
}

#endif

```

`main.c`:

```C

#include "mylib.h"

#if defined(__NATIVE_USE_MYLIB__)

int printf(const char* s1) {
    return 0;
}

#endif

```

可以看到，我自己重定义了printf，它什么都不干。因此我们可以通过 main 函数运行结果来判断调用的是glibc还是 mylib。

首先注释宏，然后编译、链接：

```bash
$ gcc -c -o mylib mylib.c
In file included from mylib.c:2:
mylib.h:3:5: warning: conflicting types for built-in function ‘printf’; expected ‘int(const char *, ...)’ [-Wbuiltin-declaration-mismatch]
    3 | int printf(const char* s1);
      |     ^~~~~~
mylib.h:1:1: note: ‘printf’ is declared in header ‘<stdio.h>’
  +++ |+#include <stdio.h>
    1 | // #define __NATIVE_USE_MYLIB__

luyoung at luyoung-desktop in ~/Test/test6
$ gcc -o main main.c mylib
In file included from main.c:1:
mylib.h:3:5: warning: conflicting types for built-in function ‘printf’; expected ‘int(const char *, ...)’ [-Wbuiltin-declaration-mismatch]
    3 | int printf(const char* s1);
      |     ^~~~~~
mylib.h:1:1: note: ‘printf’ is declared in header ‘<stdio.h>’
  +++ |+#include <stdio.h>
    1 | // #define __NATIVE_USE_MYLIB__

luyoung at luyoung-desktop in ~/Test/test6
$ ./main
hello.
```

虽然弹出了一些警告，但是不用管，可以看到，它打印出了东西，说明它链接的是 glibc。

取消注释：

```bash
$ gcc -c -o mylib mylib.c
In file included from mylib.c:2:
mylib.h:3:5: warning: conflicting types for built-in function ‘printf’; expected ‘int(const char *, ...)’ [-Wbuiltin-declaration-mismatch]
    3 | int printf(const char* s1);
      |     ^~~~~~
mylib.h:1:1: note: ‘printf’ is declared in header ‘<stdio.h>’
  +++ |+#include <stdio.h>
    1 | #define __NATIVE_USE_MYLIB__

luyoung at luyoung-desktop in ~/Test/test6
$ gcc -o main main.c mylib
In file included from main.c:1:
mylib.h:3:5: warning: conflicting types for built-in function ‘printf’; expected ‘int(const char *, ...)’ [-Wbuiltin-declaration-mismatch]
    3 | int printf(const char* s1);
      |     ^~~~~~
mylib.h:1:1: note: ‘printf’ is declared in header ‘<stdio.h>’
  +++ |+#include <stdio.h>
    1 | #define __NATIVE_USE_MYLIB__

luyoung at luyoung-desktop in ~/Test/test6
$ ./main

```

可以看到，它链接了 `mylib`，因为它什么都没打印出来。

这里将我们的库打包成静态库 `mylib.a`:

```bash
$ gcc -c -o mylib.o mylib.c
In file included from mylib.c:2:
mylib.h:2:5: warning: conflicting types for built-in function ‘printf’; expected ‘int(const char *, ...)’ [-Wbuiltin-declaration-mismatch]
    2 | int printf(const char* s1);
      |     ^~~~~~
mylib.h:1:1: note: ‘printf’ is declared in header ‘<stdio.h>’
  +++ |+#include <stdio.h>
    1 | #define __NATIVE_USE_MYLIB__

luyoung at luyoung-desktop in ~/Test/test6
$ ar rcs libmylib.a mylib.o

luyoung at luyoung-desktop in ~/Test/test6
$ gcc -c main.o main.c
In file included from main.c:1:
mylib.h:2:5: warning: conflicting types for built-in function ‘printf’; expected ‘int(const char *, ...)’ [-Wbuiltin-declaration-mismatch]
    2 | int printf(const char* s1);
      |     ^~~~~~
mylib.h:1:1: note: ‘printf’ is declared in header ‘<stdio.h>’
  +++ |+#include <stdio.h>
    1 | #define __NATIVE_USE_MYLIB__
gcc: warning: main.o: linker input file unused because linking not done

luyoung at luyoung-desktop in ~/Test/test6
$ gcc -o main main.o -L. -lmylib


luyoung at luyoung-desktop in ~/Test/test6
$ ./main

```

我们通过 `ar` 命令将目标文件归档成静态库。


## 三、升级后的例子

在这个例子中，我要使用 `dlsym` 来主动抛弃我们自己做的库，然后加载 glibc 中的 `printf`。

这里需要对文件进行修改，首先创建 aaa.h:
```C
#include <dlfcn.h>
#include <stdarg.h>
#include <stdio.h>
#ifndef RTLD_NEXT
#define RTLD_NEXT ((void *)-1l)
#endif

#ifndef RTLD_DEFAULT
#define RTLD_DEFAULT ((void *)0)
#endif

// 自定义的 printf 函数
int aaa_printf(const char *format, ...);
void aaa();
```

接着定义函数实现 aaa.c:
```C
#include "aaa.h"

// 自定义的 printf 函数
int aaa_printf(const char *format, ...) {
    static int (*real_printf)(const char *format, ...) = NULL;
    if (!real_printf) {
        real_printf = (int (*)(const char *, ...))dlsym(RTLD_DEFAULT, "printf");
        // real_printf = (int (*)(const char *, ...))dlsym(RTLD_NEXT, "printf");
        if (!real_printf) {
            fprintf(stderr, "Error loading real printf\n");
            return -1;
        }
    }

    va_list args;
    va_start(args, format);
    real_printf("Intercepted: ");
    real_printf(format);
    // int ret = vfprintf(stdout, format, args); // 使用正确的文件流指针
    va_end(args);

    return 0;
}
void aaa(){
    aaa_printf("hello....");
}
```

这里的思路是，先将 `mylib`中的符号通过 `-Wl,--whole-archive` 全部链接到目标文件中，然后再通过动态加载的方式来加载另一个编译单元 `aaa.c`:
```bash
gcc -o main -Wl,--whole-archive -L. -lmylib aaa.o main.o  -Wl,-no-whole-archive -ldl
```
这样就能在链接 `aaa.c` 的时候，选择已经连接好的 `mylib` 中的符号 `printf` 了。因此运行结果为：
```bash
$ ./main

```

可以看到它什么都没输出，这符合预期。如果我换成：
```C
int aaa_printf(const char *format, ...) {
   ...
        real_printf = (int (*)(const char *, ...))dlsym(RTLD_NEXT, "printf");
        if (!real_printf) {
            fprintf(stderr, "Error loading real printf\n");
            return -1;
        }
    }

    ...

    return 0;
}
```

编译、连接、运行之后：
```bash
$ gcc -c -o aaa.o aaa.c

luyoung at luyoung-desktop in ~/Test/test6
$ gcc -o main aaa.o main.o -Wl,--whole-archive -L. -lmylib  -Wl,-no-whole-archive -ldl

luyoung at luyoung-desktop in ~/Test/test6
$ ./main
Intercepted: hello....%
```
很符合预期，它在查找第二个 `printf` 符号，它自然在 `glibc` 中（因此它找不到第二个符号，就会自动链接 `glibc`）。


## 四、platform.c 中的例子
看看 `platform.c` 是怎么组织这一关系的。首先在编译 `ARCH=native`的时候，它要使用到 ioe，而 ioe 依赖于一些静态库，比如 `klib`。坏消息是，`klib` 中可能（是否启用 `__NATIVE_USE_KILB__`）会包含一些 `glibc` 中的已经实现的库函数，比如 `printf`、`memset`、`malloc` 等等。如果 `ioe` 使用了`klib`，可能会出现一些问题导致 `ioe` 不正常工作，因为 `klib` 中的自己实现的函数可能不健壮。

为了让 `ioe` 健壮执行，必须让它去选择链接 `glibc` 中的函数而不是我们的 `klib`，那么 `am` 是怎么解决这个问题的呢？它使用了`动态加载库dl` 中的函数来控制链接行为，就是上文描述的那样。
```C
static void init_platform() {
  ...
  // use dynamic linking to avoid linking to the same function in RT-Thread
  int (*ftruncate_libc)(int, off_t) = dlsym(RTLD_NEXT, "ftruncate");
  ...
}

```
当把 `platform.c` 编译好后，它要组成静态库 `am.a` 的一部分。当使用如下命令进行连接的时候：
```C
gcc -o main -Wl,--whole-archive -L. -lmylib aaa.o main.o -Wl,-no-whole-archive -ldl
```

这个将 `aaa.o main.o` 链接到一起形成 `main`，并且将 `mylib` 中的符号全部包含到 `main`。当 `main` 调用 `printf` 时，可以选择动态加载这个符号（因为我们使用了`dlsym`）。


以上就是文章的所有内容了。












