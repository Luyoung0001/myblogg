---
title: "Linux 驱动开发日记（二）：总线设备驱动模型"
date: 2025-10-22 15:03:13
tags:
  - "Linux"
  - "driver"
  - "bus device model"
categories:
  - "Operating Systems"
---

## 总线驱动模型

这种架构将设备（硬件描述）和驱动（操作逻辑）分离，实现了解耦和灵活性。内核中的平台总线（platform bus）负责将它们连接起来。

这种分离将驱动与驱动对硬件操作分成了两部分：驱动和设备。这就引来了一些问题，比如：

- 驱动怎么匹配设备
- 设备怎么匹配驱动
- 匹配上了后，probe 要做什么
- 删除设备后，remove 要做什么
- 一个驱动带多个设备
- 等等

以下通过例子来阐述这种分离，以及分离的两部分的相互配合。

## 示例

驱动代码示例 hello_drv.c:

```C

static int hello_probe(struct platform_device* pdev) {
    printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
    return 0;
}

static int hello_remove(struct platform_device* pdev) {
    printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
    return 0;
}

static struct platform_driver hello_driver = {
    .probe = hello_probe,
    .remove = hello_remove,
    .driver =
        {
            .name = "hello_bus",
        },
};

static int __init hello_drv_init(void) {
    int err;

    printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
    err = platform_driver_register(&hello_driver);

    return err;
}

static void __exit lhello_drv_exit(void) {
    printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
    platform_driver_unregister(&hello_driver);
}

module_init(hello_drv_init);
module_exit(lhello_drv_exit);

MODULE_LICENSE("GPL");

```

设备节点示例 hello_dev.c:

```C
static void hello_dev_release(struct device* dev) {
    printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
}

static struct platform_device hello_dev = {
    .name = "hello_bus",
    .dev =
        {
            .release = hello_dev_release,
        },
};

static int __init hello_dev_init(void) {
    int err;

    printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
    err = platform_device_register(&hello_dev);

    return err;
}

static void __exit hello_dev_exit(void) {
    printk("%s %s %d\n", __FILE__, __FUNCTION__, __LINE__);
    platform_device_unregister(&hello_dev);
}

module_init(hello_dev_init);
module_exit(hello_dev_exit);

MODULE_LICENSE("GPL");
```

## 现象

首先编译内核模块，不管是 dev 还是 drv 都是内核模块；然后将 drv 加载到内核中：

```bash
make all
make install_drv
sudo dmesg
```

查看内核日志，可以看到会调用 hello_drv_init 函数。

```bash
[357667.755952] /home/luyoung/Linux_based/linux_driv/examples/hello_bus/hello_drv.c hello_drv_init 40
```

```bash
make install_dev
sudo dmesg
```
当 dev 被注册的时候，会进行 「驱动：设备」匹配，一旦匹配成功，drv 就会调用 probe 函数。

```bash
[357700.704809] /home/luyoung/Linux_based/linux_driv/examples/hello_bus/hello_dev.c hello_dev_init 33
[357700.704863] /home/luyoung/Linux_based/linux_driv/examples/hello_bus/hello_drv.c hello_probe 19
```

```bash
make uninstall
```

当卸载 dev 的时候，会先调用 hello_dev_exit，然后 platform_device_unregister，从而调用 drv 中的 hello_remove，最后调用 hello_dev_release：

```bash
[357974.894469] /home/luyoung/Linux_based/linux_driv/examples/hello_bus/hello_dev.c hello_dev_exit 40
[357974.894486] /home/luyoung/Linux_based/linux_driv/examples/hello_bus/hello_drv.c hello_remove 24
[357974.894505] /home/luyoung/Linux_based/linux_driv/examples/hello_bus/hello_dev.c hello_dev_release 19
```
写在 drv 的时候，直接调用 hello_drv_exit：
```bash
[358280.089620] /home/luyoung/Linux_based/linux_driv/examples/hello_bus/hello_drv.c hello_drv_exit 47
```

示例图大致如下：

```txt
┌─────────────┐      ┌─────────────┐
│  加载驱动    │      │  加载设备    │
│ (install_drv)│      │ (install_dev)│
└──────┬──────┘      └──────┬──────┘
       │                    │
       │ 注册平台驱动       │ 注册平台设备
       │ hello_drv_init     │ hello_dev_init
       │                    │
       └───────────┬────────┘
                   │
                   │ 总线匹配
                   │ (name: "hello_bus")
                   │
                   ▼
            调用驱动 probe 函数
               hello_probe
                   │
                   │
┌─────────────┐     │      ┌─────────────┐
│  卸载驱动    │◀────┘      │  卸载设备    │
│             │            │ (uninstall) │
└──────┬──────┘            └──────┬──────┘
       │                          │
       │ 注销平台驱动             │ 注销平台设备
       │ hello_drv_exit          │ hello_dev_exit
       │                          │
       │                          │ 触发驱动 remove
       │                          │ hello_remove
       │                          │
       │                          │ 释放设备资源
       │                          │ hello_dev_release
       │                          │
       └───────────┬──────────────┘
                   │
                   ▼
                 结束

```

