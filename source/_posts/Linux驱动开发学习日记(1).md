---
title: Linux 驱动开发学习日记：字符设备驱动
date: 2025-10-20 15:03:13
tags: Linux 驱动
categories: Linux
---

## 前言

为了更深层次了解计算机的工作原理，我要开始学习 Linux 驱动开发了，主要面向嵌入式 Linux。

## 驱动

驱动要保证内核安全，防止崩溃、数据泄露等，防御性编程是安全的基本思维。

驱动作为 Linux 内核的关建代码，要和设备直接沟通：在高权限模式下直接访问寄存器或者内存（地址重定向），因此必须保证安全，使用内核提供的库函数。这样，可以防止用户态程序提供的指针非法或者借用内核来执行一些危险操作。 

## 一个例子

这是一个很简单的字符设备驱动：

```C
#include <linux/fs.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/uaccess.h>

static int major = 0;
static int ker_val = 123;

static ssize_t hello_read(struct file* file,
                          char __user* buf,
                          size_t size,
                          loff_t* offset) {
    int err;
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    copy_to_user(buf, &ker_val, 4);
    return 4;
}

static ssize_t hello_write(struct file* file,
                           const char __user* buf,
                           size_t size,
                           loff_t* offset) {
    int err;
    printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
    copy_from_user(&ker_val, buf, 4);
    return 4;
}

static struct file_operations hello_fops = {
    .owner = THIS_MODULE,
    .read = hello_read,
    .write = hello_write,
};

int __init hello_init(void) {
    printk("hello drv init\n");
    major = register_chrdev(0, "hello_drv", &hello_fops);
    return 0;
}

void __exit hello_exit(void) {
    printk("hello drv exit\n");
    unregister_chrdev(major, "hello_drv");
}

MODULE_LICENSE("GPL");

module_init(hello_init);
module_exit(hello_exit);

```

### module_init module_exit 宏

这两个宏会给 init_module、cleanup_module 定义一个别名，为hello_init、hello_exit。

比如：
```C
#include <linux/module.h>

int init_module(void){}
void cleanup_module(void){}

MODULE_LICENSE("GPL");
```

这就是一个非常简单的驱动，它摒弃了 module_init，module_exit 宏。至于 `MODULE_LICENSE("GPL")`，Linux 内核采用 GPL (GNU General Public License) 许可证，内核模块被视为内核的衍生作品。使用 MODULE_LICENSE("GPL") 声明：
- 表明你的模块遵循 GPL 许可证
- 允许模块与 GPL 许可的内核代码链接
- 避免违反 GPL 许可证的法律风险

### struct file_operations

这是一个关建的结构体，里面定义了设备的一切操作。

### register_chrdev & unregister_chrdev

一个设备有三个信息：
- 操作集合；
- 设备号；
- 设备名。

因此可以在 init 函数中通过函数 `major = register_chrdev(0, "hello_drv", &hello_fops)` 来将设备导入内核设备数组，而且返回一个主设备号。`unregister_chrdev(major, "hello_drv")` 来将设备从内核设备数组中删掉。


### 数据传输：copy_from_user、copy_to_user

在驱动中，可以设置一些 buffer 甚至变量。而如果要将用户空间传来的指指针所指引的数据复制到内核空间，那么就不能直接使用`解引用操作`，这样相当危险，必须使用可以对指针检验的函数 `copy_from_user`、`copy_to_user`。因为切换到内核的时候，CPU 权限很高，可以执行任何访存操作，防御性编程意味着不能信任任何用户态传来的指针。


## 模块编译与加载

使用以下编译脚本：

```Makefile
ARCH = x86
CROSS_COMPILE =

KVERSION = $(shell uname -r)
KERN_DIR =  /lib/modules/$(KVERSION)/build

all:
	make -C $(KERN_DIR) M=`pwd` modules

clean:
	make -C $(KERN_DIR) M=`pwd` modules clean
	rm -rf modules.order

obj-m	+= hello_drv.o
```

插入模块后，可以看到日志：`[4081587.620624] hello drv init`，也可以看到新的设备：

```bash
cat /proc/devices
Character devices:
  1 mem
  4 /dev/vc/0
  4 tty
...
226 drm
237 hello_drv
...
```

但是此时我们的设备文件在模块初始化的时候并没有创建    `device_create(hello_class, NULL, MKDEV(MAJOR(devno), i),NULL, "hello%d", i)`，因此得用 mknod 来创建一个临时的设备节点，缺点是在重启之后就会消失。

## 创建临时设备节点

```bash
sudo mknod /dev/xxx c 237 0

```

我们使用 root 下的 echo、cat 也可以读取设备，它们调用的也是 open、read、write。当然也可以使用测试程序对字符设备进行测试：

```bash
$ gcc hello_drv_test.c -o test

$ sudo ./test -w A
open file /dev/xxx ok
write driver: 4

$ sudo ./test -r
open file /dev/xxx ok
read driver: 4
APP read : A
```











