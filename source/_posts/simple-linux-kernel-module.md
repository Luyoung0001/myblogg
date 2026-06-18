---
title: "Linux 内核模块入门：编写第一个 Hello 模块"
date: 2022-12-28 17:44:35
tags:
  - "Linux"
  - "kernel module"
  - "C"
categories:
  - "Operating Systems"
---

<!--more-->

# Start to write a simple Linux kernel Module

### Step 1 Create a folder
Create a suitable folder, this folder will contain the source program and other compiled files.
```c
➜  mkdir Mooc && cd Mooc
```
### Step 2 Create the source file
This source file is used for code.
```c
➜  touch hello.c
```
### Step 3 Programming

```c
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/module.h>
static int __init hello_init(void) {
    printk("hello, world%n");
    return 0;
}
static void __exit hello_exit(void) {
    printk("goodbye!%n");
}

MODULE_LICENSE("GPL");

module_init(hello_init);
module_exit(hello_exit);
```
You may feel that these codes are very strange. They are obviously programs written in C language, but the style and functions used are different. This is because this is a Linux kernel program, of course it is different. Get used to it ~

### Step 4 Write the Makefile

```c
➜  touch Makefile
```
Note that the letter `M` must be capitalized so that the `make` command will find our `Makefile` file.

```c
obj-m:=hello.o

CURRENT_PATH:=$(shell pwd)
LINUX_KERNEL:=$(shell uname -r)
LINUX_KERNEL_PATH:=/usr/src/linux-headers-$(LINUX_KERNEL)
all:
	make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) modules
clean:
	make -C $(LINUX_KERNEL_PATH) M=$(CURRENT_PATH) clean
```

For how to write `Makefile`, please search Google for relevant content.
### Step 5 Compile
We have written the source code and the `Makefile` file, and then we can use the `make` command to compile.
```c
➜ make
```
If such a message pops up, it means that the compilation is successful:

```c
make -C /usr/src/linux-headers-5.4.0-48-generic M=/home/ubuntu/Mooc modules
make[1]: Entering directory '/usr/src/linux-headers-5.4.0-48-generic'
  CC [M]  /home/ubuntu/Mooc/hello.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC [M]  /home/ubuntu/Mooc/hello.mod.o
  LD [M]  /home/ubuntu/Mooc/hello.ko
make[1]: Leaving directory '/usr/src/linux-headers-5.4.0-48-generic'
```

At this time, we can look at the various files generated after compilation. We need to pay special attention to the `.ko` file, which is the target of our compilation.

```c
➜ ls -a
```
You can see that many files are generated:

```c
.   hello.c   .hello.ko.cmd  hello.mod.c     hello.mod.o       hello.o       Makefile       Module.symvers
..  hello.ko  hello.mod      .hello.mod.cmd  .hello.mod.o.cmd  .hello.o.cmd  modules.order
```

### Step 6 Insert the module
It can only be run after the super permission module is inserted.
```c
➜ sudo insmod hello.ko
```
Let's take a look at whether the insertion is successful. The `lsmod` command can view the modules running in the system.

```powershell
Module                  Size  Used by
hello             	   16384  0
binfmt_misc            24576  1
dm_multipath           32768  0
scsi_dh_rdac           16384  0
scsi_dh_emc            16384  0
scsi_dh_alua           20480  0
intel_rapl_msr         20480  0
intel_rapl_common      24576  1 intel_rapl_msr
isst_if_common         16384  0
nfit                   65536  0
rapl                   20480  0
input_leds             16384  0
joydev                 24576  0
serio_raw              20480  0
mac_hid                16384  0
qemu_fw_cfg            20480  0
			...
```

Clearly, our `hello` module was inserted successfully!
### Step 7 Print system log information
We use the `dmesg` command to print syslog messages.

```c
➜ dmesg
```
If you see a message, it means that our module is inserted into the kernel and runs successfully!

```c
							...
[    5.545811] audit: type=1400 audit(1672065866.284:8): apparmor="STATUS" operation="profile_load" profile="unconfined" name="/usr/lib/snapd/snap-confine" pi
d=504 comm="apparmor_parser"
[    5.545819] audit: type=1400 audit(1672065866.284:9): apparmor="STATUS" operation="profile_load" profile="unconfined" name="/usr/lib/snapd/snap-confine//mo
unt-namespace-capture-helper" pid=504 comm="apparmor_parser"
[    5.557997] audit: type=1400 audit(1672065866.296:10): apparmor="STATUS" operation="profile_load" profile="unconfined" name="/usr/sbin/tcpdump" pid=506 com
m="apparmor_parser"
[    5.560334] audit: type=1400 audit(1672065866.300:11): apparmor="STATUS" operation="profile_load" profile="unconfined" name="nvidia_modprobe" pid=507 comm=
"apparmor_parser"
[   18.146737] rfkill: input handler disabled
[152396.285398] hello: loading out-of-tree module taints kernel.
[152396.285504] hello: module verification failed: signature and/or required key missing - tainting kernel
[152396.285705] hello, world
```

### Step 8 Uninstall the module
After finishing the experiment, we can uninstall and clear this module, and also use the super authority command:

```c
➜ sudo rumod hello
```
Just write the module name. After uninstalling, let's see if our `hello` module is still there:

```c
➜  Mooc lsmod
Module                  Size  Used by
binfmt_misc            24576  1
dm_multipath           32768  0
scsi_dh_rdac           16384  0
scsi_dh_emc            16384  0
scsi_dh_alua           20480  0
intel_rapl_msr         20480  0
intel_rapl_common      24576  1 intel_rapl_msr
isst_if_common         16384  0
nfit                   65536  0
rapl                   20480  0
input_leds             16384  0
joydev                 24576  0
serio_raw              20480  0
mac_hid                16384  0
qemu_fw_cfg            20480  0
sch_fq_codel           20480  2
ip_tables              32768  0
x_tables               40960  1 ip_tables
autofs4                45056  2
```
Obviously, the `hello` module was successfully uninstalled.
Let's look at the system log information again:

```c
[    5.542072] audit: type=1400 audit(1672065866.280:7): apparmor="STATUS" operation="profile_load" profile="unconfined" name="/{,usr/}sbin/dhclient" pid=503
comm="apparmor_parser"
[    5.545811] audit: type=1400 audit(1672065866.284:8): apparmor="STATUS" operation="profile_load" profile="unconfined" name="/usr/lib/snapd/snap-confine" pi
d=504 comm="apparmor_parser"
[    5.545819] audit: type=1400 audit(1672065866.284:9): apparmor="STATUS" operation="profile_load" profile="unconfined" name="/usr/lib/snapd/snap-confine//mo
unt-namespace-capture-helper" pid=504 comm="apparmor_parser"
[    5.557997] audit: type=1400 audit(1672065866.296:10): apparmor="STATUS" operation="profile_load" profile="unconfined" name="/usr/sbin/tcpdump" pid=506 com
m="apparmor_parser"
[    5.560334] audit: type=1400 audit(1672065866.300:11): apparmor="STATUS" operation="profile_load" profile="unconfined" name="nvidia_modprobe" pid=507 comm=
"apparmor_parser"
[   18.146737] rfkill: input handler disabled
[152396.285398] hello: loading out-of-tree module taints kernel.
[152396.285504] hello: module verification failed: signature and/or required key missing - tainting kernel
[152396.285705] hello, world
[153086.332570] goodbye!
```

As you can see, the system correctly printed `goodbye!`.

### Step 9 Clear the compiled files

```c
➜  Mooc make clean
make -C /usr/src/linux-headers-5.4.0-48-generic M=/home/ubuntu/Mooc clean
make[1]: Entering directory '/usr/src/linux-headers-5.4.0-48-generic'
  CLEAN   /home/ubuntu/Mooc/Module.symvers
make[1]: Leaving directory '/usr/src/linux-headers-5.4.0-48-generic'
```
We can see that the files generated in our folder have been cleared:

```c
➜  Mooc ls -a
.  ..  hello.c  Makefile
```

*This is one small step in Linux kernel programming, and this article is over, thank you for reading.*

