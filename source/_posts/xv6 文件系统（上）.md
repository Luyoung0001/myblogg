---
title: xv6 文件系统（上）
date: 2023-12-18 17:00:59
tags: 笔记 学习 xv6 操作系统 OS
categories: OS
---

<!--more-->

## 〇、前言
本文将会结合 **xv6** 源码讨论文件系统的工作原理。
## 一、文件系统实现概述
 xv6 文件系统可以用下面的图来表示：
 ![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/30fe0488ebb94e3c923471bcec6e0b31_1720253687327.png?token=ANB4BCLBDV23KBET47CTY6TGRD6TO)
按照分层的方式进行理解：
- 在最底层是**磁盘**，也就是一些实际保存数据的存储设备，正是这些设备提供了持久化存储。
- 在这之上是**buffer cache**或者说 **block cache**，这些cache可以避免频繁的读写磁盘。这里我们将磁盘中的数据保存在了内存中。
- 为了保证持久性，再往上通常会有一个**logging层**。
- 在logging层之上，xv6有**inode cache**，这主要是为了同步（synchronization）。inode通常小于一个disk block，所以多个inode通常会打包存储在一个disk block中。为了向单个inode提供同步操作，xv6维护了**inode cache**。
- 再往上就是**inode本身**了。它实现了read/write。
- 再往上，就是**文件名**&**文件描述符**操作。

## 二、文件系统怎么使用硬盘
有些术语有点让人困惑，它们是 **sectors** 和 **blocks**：
- **sector** 通常是磁盘驱动可以读写的最小单元，它过去通常是**512**字节；
- **block** 通常是操作系统或者文件系统视角的数据。它由文件系统定义，在xv6中它是1024字节。**所以 xv6 中一个block对应两个sector**。通常来说一个block对应了一个或者多个sector。

有的时候，人们也将磁盘上的 **sector** 称为 **block**，所以这里的术语也不是很精确。

从文件系统的角度来看磁盘还是很直观的。因为对于磁盘就是读写 **block** 或者 **sector**，我们可以将磁盘看作是一个巨大的**block**的**数组**，数组从**0**开始，一直增长到磁盘的最后。

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/5ddd240a426a45d0923a595d5941f384_1720253687327.png?token=ANB4BCPT6SB7PRRYEKLKIKLGRD6TQ)


而文件系统的工作就是将所有的数据结构以一种能够在重启之后重新构建文件系统的方式，存放在磁盘上。虽然有不同的方式，但是 **xv6** 使用了一种非常简单，但是还挺常见的布局结构。通常来说：
- **block0** 要么没有用，要么被用作**boot sector**来启动操作系统。
- **block1** 通常被称为**super block**，它描述了文件系统。它可能包含磁盘上有多少个block共同构成了文件系统这样的信息。我们之后会看到XV6在里面会存更多的信息，你可以通过block1构造出大部分的文件系统信息。
- 在**xv6** 中，**log**从 **block2** 开始，到 **block32** 结束。实际上 **log** 的大小可能不同，**这里在super block中会定义log就是30个block**。
- 接下来在 **block32** 到 **block45** 之间，**xv6** 存储了 **inode**。
- 之后是 **bitmap block**，这是我们构建文件系统的默认方法，**它只占据一个block**。它记录了数据block是否空闲。
- 之后就全是**数据block**了，**数据block**存储了文件的内容和目录的内容（目录也是文件）。

通常来说，**bitmap block，inode blocks和log blocks被统称为metadata block**。它们虽然不存储实际的数据，但是它们存储了能**帮助文件系统完成工作**的元数据。

假设 **inode** 是 **64** 字节，如果你想要读取 **inode10**，那么你应该按照下面的公式去对应的 **block** 读取 **inode**：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/04e09f55418a4ec78b30579c533a2499_1720253687327.png?token=ANB4BCOBPNV633LQZDFNZM3GRD6TS)
所以 **inode0** 在 **block32**，**inode17** 会在 **block33**。只要有 **inode** 的编号，我们总是可以找到 **inode** 在磁盘上存储的位置。

## 三、inode 结构
**inode** 结构体在内存中的表示如下：
```c
#define NDIRECT 12

// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```
在硬盘中的表示如下：

```c
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};

```

我们先研究在硬盘的 **inode** 结构：
- 通常来说它有一个 **type** 字段，表明 **inode** 是文件还是目录。
- **nlink**字段，也就是 **link计数器**，用来跟踪究竟有多少文件名指向了当前的**inode**。
- **size**字段，表明了文件数据有多少个字节。
- 不同文件系统中的表达方式可能不一样，不过在 **xv6** 中接下来是一些 **block** 的编号，例如编号0，编号1，等等。**xv6**的 **inode** 中总共有**12**个 **block编号**。这些被称为**direct block number**。这 **12** 个 **block** 编号指向了构成文件的前 **12** 个 **block**。举个例子，如果文件只有**2**个字节，那么只会有一个**block**编号**0**，它包含的数字是磁盘上文件前2个字节的**block**的位置。
- 之后还有一个 **indirect block number**，它对应了磁盘上一个 **block**，这个 **block** 包含了**256**个 **block number**，这 **256** 个 **block number** 包含了文件的数据。所以 **inode** 中 **block number0** 到 **block number11** 都是**direct block number**，而 **block number12**保存的 **indirect block number** 指向了另一个**block**。所以数组 `addrs[NDIRECT+1]` 的长度为 **13**。

如果一个 **block号** 占 **4** 个字节，那么一个**block** 就可以存储 `1024/4 = 256` 个 **block号**。所以我们很轻易地计算出 **xv6** 中一个文件的大小上限：`(256+12)*1KB`。因为一个块号占 4 个字节，那么 xv6 一共可以寻址 `2^32`个 block，一个 block 大小为 1KB，那么 xv6 一共可以安装的硬盘大小为  `2^42`B，也就是 **4TB**。

## 四、目录
文件系统的酷炫特性就是**层次化的命名空间**（**hierarchical namespace**），你可以在文件系统中保存对用户友好的文件名。大部分 **Unix文件系统** 有趣的点在于，一个目录本质上是一个文件加上一些文件系统能够理解的结构。在 **xv6** 中，这里的结构极其简单。每一个目录包含了**directory entries**，每一条 **entry** 都有固定的格式：
- 前 **2** 个字节包含了目录中文件或者子目录的 **inode** 编号;
- 接下来的 **14** 个字节包含了文件或者子目录名。

所以每个 **entry** 总共是**16**个字节。


对于实现路径名查找，这里的信息就足够了。假设我们要查找路径名 **“/y/x”**，我们该怎么做呢？

从路径名我们知道，应该从 **root inode** 开始查找。通常 **root inode** 会有固定的 **inode** 编号，在 **xv6** 中，这个编号是**1**。我们该如何根据编号找到 **root inode** 呢？

从前一节我们可以知道，**inode** 从 **block32** 开始，如果是 **inode1**，那么必然在**block32** 中的 **64** 到 **128** 字节的位置。所以文件系统可以直接读到 **root inode** 的内容。

由于 **inode**（**in-memory copy of an inode**） 中包含对于路径名查找程序，接下来就是扫描 **root inode** 包含的所有 **block**，就是为了找到 **“y”** 。该怎么找到 **root inode** 所有对应的 **block** 呢？根据前一节的内容就是读取所有的 **direct block number**和**indirect block number**。

如果找到了，那么 **目录y** 也会有一个 **inode** 编号，假设是 **251**。我们可以继续从**inode251**查找，先用公式计算出 **block号**，然后再根据偏移量读取 **inode251** 的内容，之后再扫描 **inode** 所有对应的**block**，以找到 **“x”**并得到**文件x**对应的**inode编号**，最后将其作为**路径名查找的结果**返回。

这么简单的一件事，上面的过程一共查询了三次 **inode**（两次**目录node**，一次**文件x** 的 **inode** ），两次 **扫描block** 操作。

很明现，这里的结构不是很有效。**为了找到一个目录名，需要线性扫描**。实际的文件系统会使用更复杂的数据结构来使得查找更快，当然这又是设计数据结构的问题，而不是设计操作系统的问题。出于简单和更容易解释的目的，**xv6** 使用了这里这种非常简单的数据结构。

## 五、文件系统工作示例

首先启动 **xv6**，启动 **xv6** 的过程中，调用了**makefs**指令，来创建一个文件系统。

所以**makefs**创建了一个全新的磁盘镜像，在这个磁盘镜像中包含了我们在指令中传入的一些文件。**makefs**为你创建了一个**包含这些文件的新的文件系统**。

xv6 总是会打印文件系统的一些信息，所以从指令的下方可以看出有**46**个**meta block**，其中包括了：
- boot block
- super block
- 30个log block
- 13个inode block
- 1个bitmap block

之后是**954**个**data block**。所以这是一个袖珍级的文件系统，总共就包含了**1000**个**block**。

稍微修改了一下 **xv6**之后，使得任何时候写入 **block** 都会打印出 **block** 的编号：

```c
void
log_write(struct buf *b)
{
  ...
  printf("write: %d\n",b->blockno);     // 打印使用的 block 号
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // Add new block to log?
    bpin(b);
    log.lh.n++;
  }
  release(&log.lock);
}
```


接下来我们运行一个命令 `echo hi > x`，来看一下特定的命令对哪些 **block** 做了写操作，并理解为什么要对这些 **block** 写入数据。

我们通过`echo hi > x`，来创建一个**文件x**，并写入字符 `hi`，这会打印以下信息：

```bash
$ echo hi > x
1	write: 33
2	write: 33
3	write: 46
4	write: 32
5	write: 33
6	write: 45
7	write: 908
8	write: 908
9	write: 33
10	write: 908
11	write: 33
```

这里会有几个阶段：
- 第一阶段是创建**文件x**；
- 第二阶段将 `hi`写入文件；
- 第三阶段将 `\n`换行符写入到文件；

看一下`user/echo.c`的代码实现，基本就是这3个阶段：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int i;

  for(i = 1; i < argc; i++){
    write(1, argv[i], strlen(argv[i]));
    if(i + 1 < argc){
      write(1, " ", 1);
    } else {
      write(1, "\n", 1);
    }
  }
  exit(0);
}

```

首先它将 `hi`写入到标准输出（我们在此处已经重定向了，对于 `echo()`，`argc==2` ），然后继续写入 `\n`。

### 1、创建文件x
我们已经了解到，**x文件**是创建在根目录下的。既然是创建文件，第一步肯定是分配一个 **inode**。因此要在空闲 inode 列表中找一个空闲的 inode，然后打上标记。

然后我们要在此 **inode** 中写入一些信息，比如 **inode** 的引用计数、**addrs**等内容。

以上两步对应于：

```bash
1	write: 33
2	write: 33
```

由于我们在根目录下创建了一个**文件x**，因此根目录的 **inode**、**inode** 映射的**addrs**中的内容都要被修改。首先修改的是根目录中的第一个 **block**，**文件x** 是根目录下的一个 **entry**，自然要修改这个 **block**。然后根目录的 **inode** 也要修改，因为根目录大小变了。之后，又修改了一次**文件x** 的 **inode**。


以上两步对应于：

```bash
3	write: 46
4	write: 32
5	write: 33
```

### 2、写入文件x `hi`
因为要引用某个块，所以要更新 **bitmap**。之后要往某个 **direct block** 中写两个字符。最后还需要更改**文件x** **inode**中的**size**字段，而且还会将 **inode** 与 **block** 关联起来，即修改 `addrs` 字段。这对应于：

```bash
6	write: 45
7	write: 908
8	write: 908
9	write: 33
```

这意味着文件系统挑选了 **block 908**。

### 3、写入`\n`
这就很简单了，写文件，修改 **inode**。
```bash
10	write: 908
11	write: 33
```


###  4、再执行一条命令
通过再执行一条命令 `echo hii > x`：这条命令会将 **文件x** 中内容覆盖，并且重新写 `hii`、`\n`（注意这是分开写的）。因为**文件x** 已经创建过了，因此这个写入**block** 的记录和上面的那个命令是不同的：
```bash
$ echo hii > x
1	write: 45
2	write: 33
3	write: 45
4	write: 908
5	write: 908
6	write: 33
7	write: 908
8	write: 33
```
首先修改**bitmap**，然后修改 **inode**。这是为了将 **x文件**中的内容全部清空，那么这个**block** 自然就不用了，需要修改 **inode** 中 的 **size**、**addrs**字段等。这对应于：

```bash
1	write: 45
2	write: 33
```

之后又要继续写 `hii`。这意味着要重新启用这个 **block**，因此要修改 **bitmap**。之后写入 `hii`，写完后，继续更新 **inode**，这对应于：

```bash
3	write: 45
4	write: 908
5	write: 908
6	write: 33
```

因为写入了 3 个字符，但是这里的 **block** 被写了两次，*目前还找不到原因*。事实上，我尝试写了几十个字符，也只有两次写入 **block**：

```bash
$ echo asdfsdfasdf > x
write: 45
write: 33
write: 45
write: 908
write: 908
write: 33
write: 908
write: 33
```

然后就是写入`\n`了：

```bash
7	write: 908
8	write: 33
```
以上就是文件系统的工作示例了。

## 五、代码

`sysfile.c`中包含了所有与文件系统相关的函数，分配**inode**发生在`sys_open()`函数中，这个函数会负责创建文件：

```c
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  argint(1, &omode);
  if((n = argstr(0, path, MAXPATH)) < 0)
    return -1;

  begin_op();

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } 
  ...
}
```

**superblock** 中存储着各种各样的信息，包括 **inode** 的数量 **ninodes**：

```c
// Disk layout:
// [ boot block | super block | log | inode blocks |
//                                          free bit map | data blocks]
//
// mkfs computes the super block and builds an initial file system. The
// super block describes the disk layout:
struct superblock {
  uint magic;        // Must be FSMAGIC
  uint size;         // Size of file system image (blocks)
  uint nblocks;      // Number of data blocks
  uint ninodes;      // Number of inodes.
  uint nlog;         // Number of log blocks
  uint logstart;     // Block number of first log block
  uint inodestart;   // Block number of first inode block
  uint bmapstart;    // Block number of first free map block
};
```

创建的过程中，它会调用 `create()`函数：

```c
static struct inode*
create(char *path, short type, short major, short minor)
{
  struct inode *ip, *dp;
  char name[DIRSIZ];

  if((dp = nameiparent(path, name)) == 0)
    return 0;

  ilock(dp);

  if((ip = dirlookup(dp, name, 0)) != 0){
    iunlockput(dp);
    ilock(ip);
    if(type == T_FILE && (ip->type == T_FILE || ip->type == T_DEVICE))
      return ip;
    iunlockput(ip);
    return 0;
  }

  if((ip = ialloc(dp->dev, type)) == 0){
    iunlockput(dp);
    return 0;
  }

  ilock(ip);
  ip->major = major;
  ip->minor = minor;
  ip->nlink = 1;
  iupdate(ip);

  if(type == T_DIR){  // Create . and .. entries.
    // No ip->nlink++ for ".": avoid cyclic ref count.
    if(dirlink(ip, ".", ip->inum) < 0 || dirlink(ip, "..", dp->inum) < 0)
      goto fail;
  }

  if(dirlink(dp, name, ip->inum) < 0)
    goto fail;

  if(type == T_DIR){
    // now that success is guaranteed:
    dp->nlink++;  // for ".."
    iupdate(dp);
  }

  iunlockput(dp);

  return ip;
}
```
`create()`函数中首先会解析路径名并找到最后一个目录，之后会查看文件是否存在，如果存在的话会返回错误。之后就会调用`ialloc()`，这个函数会为 **文件x** 分配**inode**。**ialloc**函数位于`fs.c`文件中：

```c
// Allocate an inode on device dev.
// Mark it as allocated by  giving it type type.
// Returns an unlocked but allocated and referenced inode,
// or NULL if there is no free inode.
struct inode*
ialloc(uint dev, short type)
{
  int inum;
  struct buf *bp;
  struct dinode *dip;

  for(inum = 1; inum < sb.ninodes; inum++){
    bp = bread(dev, IBLOCK(inum, sb));
    dip = (struct dinode*)bp->data + inum%IPB;
    if(dip->type == 0){  // a free inode
      memset(dip, 0, sizeof(*dip));
      dip->type = type;
      log_write(bp);   // mark it allocated on the disk
      brelse(bp);
      return iget(dev, inum);
    }
    brelse(bp);
  }
  printf("ialloc: no inodes\n");
  return 0;
}
```

它会遍历所有可能的 **inode编号**，`IBLOCK(inum, sb)`可以将通过**inum**找到 **superblock** 对应的 **block number**。`bread()`可以通过**设备名和 blockno** 可以得到硬盘中的 **block** 在内存中的**缓存**，记为 **bp**。由于 **bp** 是 **dblock** 中的一个拷贝，因此接下来就直接在 **bp** 中操作了。

找到**inode**所在的**block**，再看位于**block**中的**inode**数据的**type字段**。如果这是一个空闲的 **inode**，那么将其**type**字段设置为文件，**这会将inode标记为已被分配**。函数中的`log_write()` 就是我们之前看到在**console**中有关写**block**的输出。这里的 `log_write()` 是我们看到的**整个输出的第一个**。

以上就是第一次写磁盘涉及到的函数调用。

这里有个有趣的问题，**如果有多个进程同时调用create函数会发生什么？** 对于一个多核的计算机，进程可能并行运行，两个进程可能同时会调用到 **ialloc函数**，然后进而调用 `bread()` 函数。所以必须要有一些机制确保这两个进程不会互相影响。

这个已经在上一个实验中讨论了，就不再赘述了。

## 六、总结（大模型总结）

这篇文章深入探讨了文件系统在 **xv6** 环境下的工作原理，分层次地介绍了文件系统的实现概述、硬盘使用、**inode** 结构、目录、以及文件系统的工作示例。文章详细解释了**inode**和**block**在磁盘上的分布方式，以及文件的创建、写入和元数据修改的过程。它提供了清晰的代码示例和相关函数的解释，为理解文件系统的内部机制提供了深入而全面的认识。

文章非常详尽地描述了 **xv6** 文件系统的内部机制和数据结构，从底层的磁盘操作到高层的文件操作都有涉及。它还着重强调了在文件系统设计中的一些限制和简化，尤其是对于路径查找等操作的效率问题。这样的详细解释对于想要深入了解文件系统工作原理的读者来说是非常有帮助的。

全文完，感谢阅读。














