---
title: MIT 6.S081学习笔记（第八章）
date: 2023-12-21 19:22:05
tags: 学习 笔记 xv6 OS 操作系统 文件系统
categories: OS
---

<!--more-->

## 〇、前言

本文主要完成**MIT 6.S081** 实验八：**file system**
开始之前，切换分支：

```bash
  $ git fetch
  $ git checkout fs
  $ make clean
```

## Large files (moderate)

> The format of an on-disk `inode` is defined by `struct dinode` in `fs.h`. You're particularly interested in **NDIRECT**, **NINDIRECT**, **MAXFILE**, and the **addrs[]** element of `struct dinode`. The code that finds a file's data on disk is in `bmap()` in `fs.c`. Have a look at it and make sure you understand what it's doing. `bmap()` is called both when reading and writing a file. When writing, `bmap()` allocates new blocks as needed to hold file content, as well as allocating an indirect block if needed to hold block addresses.
`bmap()` deals with two kinds of block numbers. The **bn** argument is a "logical block number" -- **a block number** within the file, relative to the start of the file. The block numbers in `ip->addrs[]`, and the argument to `bread()`, are disk block numbers. You can view `bmap()` as mapping a file's logical block numbers into disk block numbers.

### Question requirements

> Modify `bmap()` so that it implements a doubly-indirect block, in addition to direct blocks and a singly-indirect block. You'll have to have only 11 direct blocks, rather than 12, to make room for your new doubly-indirect block; you're not allowed to change the size of an on-disk inode. The first 11 elements of `ip->addrs[]` should be direct blocks; the 12th should be a singly-indirect block (just like the current one); **the 13th should be your new doubly-indirect block.** 

### Some hints

> - Make sure you understand `bmap()`. Write out a diagram of the relationships between `ip->addrs[]`, the indirect block, the doubly-indirect block and the singly-indirect blocks it points to, and data blocks. Make sure you understand why adding a doubly-indirect block increases the maximum file size by `256*256` blocks (really -1, since you have to decrease the number of direct blocks by one).
> - Think about how you'll index the doubly-indirect block, and the indirect blocks it points to, with the logical block number.
> - If you change the definition of **NDIRECT**, you'll probably have to change the declaration of `addrs[]` in `struct inode` in `file.h`. Make sure that `struct inode` and `struct dinode` have the same number of elements in their `addrs[]` arrays.
> - If you change the definition of **NDIRECT**, make sure to create a new `fs.img`, since mkfs uses **NDIRECT** to build the file system.
> If your file system gets into a bad state, perhaps by crashing, delete fs.img (do this from Unix, not xv6). make will build a new clean file system image for you.
> - Don't forget to `brelse()` each block that you `bread()`.
> - You should allocate indirect blocks and doubly-indirect blocks only as needed, like the original `bmap()`.
> - Make sure itrunc frees all blocks of a file, including double-indirect blocks.



### Answer
上面说得很清楚，**xv6** 不支持太大的单文件，修改 `bmap()` 使其能用一个 `double-indirect block`，从而映射更多的块（256*256+256+11=65803）。

首先看看 `(d)inode` 的主要字段：

```c
#define NDIRECT 12
// in-memory copy of an inode
struct inode {
...
  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```
可以看到，`addrs[]` 字段有 **13** 个空间，我们要用前 **11** 个直接映射，第 **12** 个一级间接映射，第 **13** 个二级间接映射。要做这些，就得修改 `bmap()`：

```c
// Inode content
//
// The content (data) associated with each inode is stored
// in blocks on the disk. The first NDIRECT block numbers
// are listed in ip->addrs[].  The next NINDIRECT blocks are
// listed in block ip->addrs[NDIRECT].

// Return the disk block address of the nth block in inode ip.
// If there is no such block, bmap allocates one.
// returns 0 if out of disk space.
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[bn] = addr;
    }
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT] = addr;
    }
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

`bmap()` 函数根据 **inode** 以及 **logic block number** 返回物理磁盘块号。如果没有这个磁盘块号，那就给它分配一个，赋值之后返回。bmap()目前只能映射逻辑块号为 **0——11**（直接映射）、**12——267**（间接映射）的物理块号。

我们需要在里面设计二级间接映射，并且重新设计 逻辑块号**0——10**（直接映射）、**11——266**（一级间接映射）、**267——65802**（二级间接映射）的物理块号映射。

以下是注释以及部分代码：
```c
// 直接映射
// 一级间接映射
// 二级间接映射
	// 逻辑块号计算第一个块号（0~255）
	// 继续计算第二个 块号（0~255）
```
首先修改宏定义：

```c
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT + NINDIRECT * NINDIRECT)
```

修改后的代码如下：

```c
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;
  // 0~10(11个) 直接映射
  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[bn] = addr;
    }
    return addr;
  }
  bn = bn - NDIRECT;
  // 11~256+11(267 个) 一级间接映射
  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT] = addr;
    }
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }
  bn = bn - NINDIRECT;
  // 267~256*256+267(65803 个) 二级间接映射
  if(bn < NINDIRECT * NINDIRECT){
    // Load double-indirect block, allocating if necessary.

    // 拿到这个块号
    uint bn1 = bn/256;
    uint bn2 = bn%256;

    if((addr = ip->addrs[NDIRECT+1]) == 0){
      addr = balloc(ip->dev);
      if(addr == 0)
        return 0;
      ip->addrs[NDIRECT+1] = addr;
    }

    // 拿到一级块
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;

    if((addr = a[bn1]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn1] = addr;
        log_write(bp);
      }
    }
    brelse(bp);

    // 拿到二级块
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn2]) == 0){
      addr = balloc(ip->dev);
      if(addr){
        a[bn2] = addr;
        log_write(bp);
      }
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

当然，修改好了以后，还要考虑释放块：

```c
// Truncate inode (discard contents).
// Caller must hold ip->lock.
void
itrunc(struct inode *ip)
{
  ...
  // 释放二级间接
  if(ip->addrs[NDIRECT+1]){
    bp = bread(ip->dev, ip->addrs[NDIRECT+1]);
    a = (uint*)bp->data;

    struct buf *bp2;
    uint *a1;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j]){
        bp2 = bread(ip->dev, a[j]);
         a1 = (uint*)bp2->data;
        for(int k = 0; k < NINDIRECT; k++){
          if(a1[k]){
            bfree(ip->dev, a1[k]);
          }
        }
        brelse(bp2);
        // 释放二级间接块
        bfree(ip->dev, a[j]);
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT+1]);
    ip->addrs[NDIRECT+1] = 0;
  }
  ip->size = 0;
  iupdate(ip);
}
```

这样，这个 **lab** 就完成了！（没有参考任何代码，完全自己写，一次跑过）：

```bash
xv6 kernel is booting

init: starting sh
$ bigfile
.....................
...
wrote 65803 blocks
bigfile done; ok
```
## Symbolic links (moderate)

> In this exercise you will add symbolic links to xv6. Symbolic links (or soft links) refer to a linked file by pathname; when a symbolic link is opened, the kernel follows the link to the referred file. Symbolic links resembles hard links, but hard links are restricted to pointing to file on the same disk, while symbolic links can cross disk devices. Although xv6 doesn't support multiple devices, implementing this system call is a good exercise to understand how pathname lookup works.
### Question requirements

> You will implement the `symlink(char *target, char *path)` system call, which creates a new symbolic link at path that refers to file named by **target**. For further information, see the man page symlink. To test, add **symlinktest** to the **Makefile** and run it. 

### Some hints
> - First, create a new system call number for symlink, add an entry to `user/usys.pl`, `user/user.h`, and implement an empty **sys_symlink** in `kernel/sysfile.c`.
> - Add a new file type (**T_SYMLINK**) to `kernel/stat.h` to represent a symbolic link.
> Add a new flag to `kernel/fcntl.h`, (**O_NOFOLLOW**), that can be used with the open system call. Note that flags passed to open are combined using a bitwise OR operator, so your new flag should not overlap with any existing flags. This will let you compile `user/symlinktest.c` once you add it to the **Makefile**.
> - Implement the `symlink(target, path)` system call to create a new symbolic link at path that refers to target. Note that target does not need to exist for the system call to succeed. You will need to choose somewhere to store the target path of a symbolic link, for example, in the **inode's data blocks**. symlink should return **an integer representing success (0) or failure (-1) similar to link and unlink**.
> - Modify the open system call to handle the case where the path refers to a symbolic link. If the file does not exist, open must fail. When a process specifies **O_NOFOLLOW** in the flags to open, open should open the symlink (and not follow the symbolic link).
> - If the linked file is also a symbolic link, you must recursively follow it until a non-link file is reached. **If the links form a cycle, you must return an error code**. You may approximate this by returning an error code if the depth of links reaches some threshold (e.g., 10).
> - Other system calls (e.g., link and unlink) must not follow symbolic links; these system calls operate on the symbolic link itself. You do not have to handle symbolic links to directories for this lab.

### Answer
要简单理解这个 lab，可以用 **Windows OS** 中创建快捷方式的行为来类比，链接（软连接和硬链接）是躺在父文件夹中的一个文件。我们的目标就是创建一个文件，然后当用户打开这个文件的时候，返回 **fd**。

因此，我们要做的事情是两件：
- 创建链接文件；
- 打开链接文件。

在 xv6 中，创建文件用的是：`static struct inode* create(char *path, short type, short major, short minor)`：

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

 fail:
  // something went wrong. de-allocate ip.
  ip->nlink = 0;
  iupdate(ip);
  iunlockput(ip);
  iunlockput(dp);
  return 0;
}
```

这个 `create` 函数是用于在文件系统中创建文件或目录的。它的主要功能包括（大模型解析）：

1. **获取父目录 `dp` 和要创建的文件或目录名 `name`：**
   - 通过 `nameiparent` 函数获取给定路径 `path` 的父目录 inode `dp` 和要创建的文件或目录名 `name`。

2. **获取父目录的锁 `ilock`：**
   - 对父目录 `dp` 进行加锁操作，确保其他进程不会同时修改该目录。

3. **检查是否已存在同名文件或目录：**
   - 通过 `dirlookup` 函数检查在父目录 `dp` 中是否已存在同名的文件或目录。
   - 如果已存在同名的文件或目录：
     - 如果要创建的类型是文件，并且已存在的节点类型是文件或设备，则返回已存在的节点 `ip`。
     - 否则释放对已存在节点 `ip` 和父目录 `dp` 的锁，并返回失败（返回 `0`）。

4. **分配新的 inode：**
   - 如果不存在同名的文件或目录，则通过 `ialloc` 函数分配一个新的 inode `ip`。
   - 对新分配的 inode `ip` 进行加锁操作。

5. **设置新节点的属性：**
   - 设置新节点 `ip` 的类型、主设备号、次设备号、链接计数等属性，并将这些信息写入磁盘（`iupdate(ip)`）。

6. **如果是目录类型 `T_DIR`：**
   - 如果要创建的类型是目录：
     - 创建目录中的 `.` 和 `..` 条目。
     - 更新父目录 `dp` 的链接计数。
     - 解锁并释放父目录 `dp`。

7. **连接新节点到父目录：**
   - 将新节点 `ip` 与父目录 `dp` 进行连接，创建文件或目录的实际条目。
   - 如果连接失败，则执行失败处理，释放新节点 `ip`。

8. **成功创建文件或目录：**
   - 如果创建成功且是目录类型，则更新父目录 `dp` 的链接计数。
   - 解锁并释放父目录 `dp`。
   
9. **失败处理：**
   - 如果出现失败，对于已分配但创建失败的 inode `ip`，将其链接计数设为 `0`，并更新其信息到磁盘。
   - 解锁并释放新节点 `ip` 和父目录 `dp`。

举个例子，要创建一个文件，必须传入一个如`/a/b/c`的 path，`nameiparent(path, name))`会返回文件 `b` 的 **inode**  **dp**。之后，会在 **dp** 中检查要被创建的文件 `c` 是否存在。如果存在直接返回这个文件 `c` 的 **inode**，如果不存在，就在 **inode block** 中分配一个 **inode**，然后对 **inode** 做一些设置，之后和父目录关联起来`dirlink(dp, name, ip->inum)`。这个文件创建完成了，至于 `c` 是文件夹还是文件，这对于父目录完全透明。也可以看到，创建文件事实上就是在创建 **inode**。由于目录也是一个文件，链接到父目录的本质是：

```c
// Write a new directory entry (name, inum) into the directory dp.
// Returns 0 on success, -1 on failure (e.g. out of disk blocks).
int
dirlink(struct inode *dp, char *name, uint inum)
{
  int off;
  struct dirent de;
  struct inode *ip;

  // Check that name is not present.
  if((ip = dirlookup(dp, name, 0)) != 0){
    iput(ip);
    return -1;
  }

  // Look for an empty dirent.
  for(off = 0; off < dp->size; off += sizeof(de)){
    if(readi(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
      panic("dirlink read");
    if(de.inum == 0)
      break;
  }

  strncpy(de.name, name, DIRSIZ);
  de.inum = inum;
  if(writei(dp, 0, (uint64)&de, off, sizeof(de)) != sizeof(de))
    return -1;

  return 0;
}
```

本质就是往  **inode** 绑定的的文件中写入一个 **entry**，也就是往父目录 `b` 中写一个 **entry**。而一个 **entry** 就是：

```c
struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```

它由 **inode** 编号和**文件名**构成。



接下来看看如何往一个文件中写内容，事实上，关于如何向一个文件中写内容，主要是指往这个文件的 **inode** 中的 **addrs** 字段指的 **block** 中写内容。由 `writei()` 函数实现：

```c

// Write data to inode.
// Caller must hold ip->lock.
// If user_src==1, then src is a user virtual address;
// otherwise, src is a kernel address.
// Returns the number of bytes successfully written.
// If the return value is less than the requested n,
// there was an error of some kind.
int
writei(struct inode *ip, int user_src, uint64 src, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  if(off > ip->size || off + n < off)
    return -1;
  if(off + n > MAXFILE*BSIZE)
    return -1;

  for(tot=0; tot<n; tot+=m, off+=m, src+=m){
    uint addr = bmap(ip, off/BSIZE);
    if(addr == 0)
      break;
    bp = bread(ip->dev, addr);
    m = min(n - tot, BSIZE - off%BSIZE);
    if(either_copyin(bp->data + (off % BSIZE), user_src, src, m) == -1) {
      brelse(bp);
      break;
    }
    log_write(bp);
    brelse(bp);
  }

  if(off > ip->size)
    ip->size = off;

  // write the i-node back to disk even if the size didn't change
  // because the loop above might have called bmap() and added a new
  // block to ip->addrs[].
  iupdate(ip);

  return tot;
}
```
这个函数 `writei` 用于向一个 inode 写入数据。它的功能包括（大模型解析）：

1. **参数解释：**
   - `struct inode *ip` 是要写入数据的目标 inode。
   - `int user_src` 表示源地址 `src` 是用户空间的虚拟地址（user virtual address）还是内核地址。
   - `uint64 src` 是数据的源地址。
   - `uint off` 是写入数据的偏移量。
   - `uint n` 是要写入的字节数。

2. **参数检查：**
   - 检查写入偏移量 `off` 是否超出了文件大小或范围。如果超出了文件大小或范围，返回错误 `-1`。
   - 检查写入的结束位置是否超过了最大文件大小限制，如果超出，则返回错误 `-1`。

3. **循环写入数据：**
   - 进入循环，`tot` 表示已经成功写入的字节数。
   - 对于每个循环迭代：
     - 使用 `bmap` 函数获取要写入数据的逻辑块的磁盘块地址。
     - 如果获取的地址为 `0`，表示无法分配新的磁盘块，退出循环。
     - 使用 `bread` 函数读取磁盘块到缓冲区 `bp` 中。
     - 计算本次写入的字节数 `m`，考虑到当前磁盘块的可用空间和写入偏移量。
     - 使用 `either_copyin` 函数将数据从源地址 `src` 复制到缓冲区 `bp` 中。如果复制失败，释放缓冲区并退出循环。
     - 将写入的缓冲区 `bp` 记录到日志（`log_write`）并释放缓冲区。

4. **更新文件大小：**
   - 如果写入的结束位置超出了当前文件的大小，则更新文件大小为写入结束位置 `off`。

5. **更新 inode 到磁盘：**
   - 即使文件大小没有变化，也将 inode 更新到磁盘，因为上面的循环可能调用了 `bmap()` 并添加了新的块到 `ip->addrs[]`。

6. **返回写入的总字节数：**
   - 返回成功写入的总字节数 `tot`。

这个函数其实就是往 **inode** 编号为 **ip** 的文件中写数据。

在创建一个文件后，如果这个文件是一个链接文件（比如快捷方式），那么拿到这个文件的 **inode** 之后，就会往这个文件中写一个很重要的信息，那就是这个链接文件要连接到的 **path**。比如文件 `c` 要指向的文件是`/a/b/c/d`，那么就需要将 `/a/b/c/d` 写入到这个文件``c``中。


以上两个函数相当重要，它是创建链接文件的重要函数，必须理解透彻。


#### 1、创建链接文件
根据 **hints**，我们要实现的 `sys_symlink()`，就是在创建链接，我们可以在这里面实现创建链接（文件）。代码和注释如下：

```c
// 获取目标链接和链接 path（包含文件名）
// 创建一个文件
// 在文件中写入目标链接
// 文件创建完成
```
先在 `kernel/stat.h` 定义必要的文件类型：

```c
#define T_DIR     1   // Directory
#define T_FILE    2   // File
#define T_DEVICE  3   // Device
#defien T_SYMLINK 4   // Symlink
```

在 `kernel/fcntl.h` 中定义文件控制宏：

```c
#define O_RDONLY  0x000
#define O_WRONLY  0x001
#define O_RDWR    0x002
#define O_CREATE  0x200
#define O_TRUNC   0x400
#define O_NOFOLLOW 0x800
```


以下是完整的代码：

```c

uint64
sys_symlink(void){

  struct inode *ip;

  // 获取目标链接和链接 path（包含文件名）
  char target[MAXPATH], path[MAXPATH];
  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
    return -1;

  begin_op();

  // 创建一个文件
  ip = create(path, T_SYMLINK, 0, 0);
  if(ip == 0){
    end_op();
    return -1;
  }

  // 在文件中写入目标链接
  if(writei(ip, 0, (uint64)target, 0, strlen(target)) < 0) {
    end_op();
    return -1;
  }
  iunlockput(ip);

  end_op();
  return 0;
}
```

#### 2、打开链接文件

创建好链接文件后，打开这个文件的时候，需要做出一些判断。当打开的是一个链接文件时，还要判断这个链接文件是否也同样也链接到了一个文件。我们设置一些标识，跟随符号链接，直到跟随到非符号链接的 inode 为止。

现对 `sys_open()` 函数做出一些修改：

```c
uint64
sys_open(void)
{
  ...
  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    // 如果不是创建文件,那么就在这里进行检查
    int symlink_depth = 0;
    while(1) {
      if((ip = namei(path)) == 0){
        end_op();
        return -1;
      }
      ilock(ip);
      // 如果是链接类型且指明要跟随,意味着要打开它，取出它的链接，进行迭代
      if(ip->type == T_SYMLINK && (omode & O_NOFOLLOW) == 0) {
        if(++symlink_depth > 10) {
          // too many layer of symlinks, might be a loop
          iunlockput(ip);
          end_op();
          return -1;
        }
        // 读到 path 中
        if(readi(ip, 0, (uint64)path, 0, MAXPATH) < 0) {
          iunlockput(ip);
          end_op();
          return -1;
        }
        iunlockput(ip);
      } else {
        // 说明这是一个普通文件,退出之后，准备返回
        break;
      }
    }
...

  iunlock(ip);
  end_op();

  return fd;
}
```

这样，我们就完成了整个实验：

```bash
== Test running bigfile == 
$ make qemu-gdb
running bigfile: OK (171.3s) 
== Test running symlinktest == 
$ make qemu-gdb
(1.0s) 
== Test   symlinktest: symlinks == 
  symlinktest: symlinks: OK 
== Test   symlinktest: concurrent symlinks == 
  symlinktest: concurrent symlinks: OK 
```



*全文完，感谢阅读。*
























