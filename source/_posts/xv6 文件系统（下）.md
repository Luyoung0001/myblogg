---
title: xv6 文件系统（下）
date: 2023-12-18 22:43:42
tags: 学习 笔记 xv6 操作系统 OS
categories: OS
---

<!--more-->

## 〇、前言
计算机崩溃后如何恢复，是一个很重要的话题。对于内存中的数据无关痛痒，开机后重新载入就能解决问题；但是对于持久化存储设备，当你尝试修改一个文件，突然断电当你重新打开文件后，这个文件的状态是否正确，是一个问题。

我们讨论文件的状态是否正确，是指文件系统对于这个文件是否运行正常，比如 **entry** 中的 **inode** 信息是否与用户期望的一致，比如 **size** 字段是否正确等。

因此，如何从崩溃中恢复文件，是着重研究的问题。事实上，logging 是解决这一类问题的普遍方法，不管是 **Redis**、**MySQL**，还是其它数据系统，都利用 **logging** 技术恢复、导入、复制数据等。

在文件系统中，**logging** 通过记录修改操作的顺序和内容来确保文件系统的数据一致性。通过在事务发生前将修改操作写入日志，然后再将这些修改应用到实际的文件系统结构中，即使在崩溃时，系统也可以根据日志来重新应用这些操作，从而恢复到崩溃之前的状态。

这种技术在许多数据系统中也被广泛应用，比如数据库系统（比如**MySQL**、**PostgreSQL**等）、缓存系统（比如**Redis**）等。它们使用日志记录来确保数据库或者缓存在崩溃或者异常情况下的数据完整性和一致性。通过记录操作和事务，系统可以在崩溃后重新应用这些记录，从而确保数据的恢复和一致性。

对于文件系统而言，**logging** 技术能够确保即使在发生意外情况时，文件系统也能够安全地进行崩溃恢复，防止数据损坏或丢失。

## 一、文件系统 logging

在 **super block** 之后就是 **log block**，我们今天主要介绍的就是 **log block**。**log block** 之后是 **inode block**，每个 **block** 可能包含了多个 **inode**。之后是 **bitmap block**，它记录了哪个 **data block** 是空闲的。最后是 **data block**，这里包含了文件系统的实际数据。

**logging**的基本思想就是将磁盘分成两部分：一个是用于记录变更的log区域；另一个是存储实际文件系统的数据区域。这种分区的方式确保了在写入文件系统之前，**先将变更记录到log中，从而提供了一种安全机制以应对崩溃或意外关机等情况**。

这种方式的优点在于，即使在写入文件系统的过程中发生了意外情况，比如断电或者系统崩溃，**log中的记录仍然存在**，记录了文件系统的变更操作。这样，在系统重启时，可以根据 **log** 中的内容重新应用这些变更，从而恢复到上一次的一致状态。

文件系统通常会选择定期地或者在关键操作之前将 **log** 中的变更同步到文件系统数据区域中，以确保文件系统的一致性和持久性。

这种设计对于确保数据的完整性和系统的稳定性非常有帮助，因为它提供了一种机制来处理异常情况下的数据恢复，并且在很大程度上减少了数据损失的风险。

它有一些好的属性：
- 首先，它可以确保文件系统的系统调用是原子性的。比如你调用 `create/write` 系统调用，这些系统调用的效果是要么完全出现，要么完全不出现，这样就避免了一个系统调用只有部分写磁盘操作出现在磁盘上。
- 其次，它支持快速恢复（**Fast Recovery**）。在重启之后，我们不需要做大量的工作来修复文件系统，只需要非常小的工作量。这里的快速是相比另一个解决方案来说，在另一个解决方案中，你可能需要读取文件系统的所有 **block**，读取 **inode**，**bitmap block**，并检查文件系统是否还在一个正确的状态，再来修复。**而logging可以有快速恢复的属性**。
- 最后，原则上来说，它可以非常的高效。

### log write
当需要更新文件系统时，我们并不是更新文件系统本身。假设我们在内存中缓存了 **bitmap block**，也就是 **block45**。当需要更新**bitmap** 时，我们并不是直接写 **block45**，而是将数据写入到 **log** 中，并记录这个更新应该写入到 **block45**。对于所有的**写 block** 都会有相同的操作，例如**更新inode**，也会记录一条写 **block33**的 **log**。

所以基本上，任何一次写操作都是先写入到 **log**，我们并不是直接写入到 **block** 所在的位置，而总是先将写操作写入到 **log** 中。

### commit op

之后在某个时间，当文件系统的操作结束了，并且都存在于 **log** 中，我们会 **commit** 文件系统的操作。这意味着我们需要在 **log** 的某个位置记录属于同一个文件系统的操作的个数，例如 **5**。

### install log

当我们在 **log** 中存储了所有写 **block** 的内容时，如果我们要真正执行这些操作，只需要将 **block** 从 **log分区** 移到 **文件系统分区**。我们知道第一个操作该写入到 **block45**，我们会直接将数据从 **log** 写到 **block45**，第二个操作该写入到 **block33**，我们会将它写入到 **block33**，依次类推。

### clean log

一旦完成了，就可以 **清除log**。**清除log** 实际上就是**将属于同一个文件系统的操作的个数设置为0**。

因此 logging 可以分为以下几步：
- 1）log write;
- 2) commit op;
- 3) install;
- 4) clean up.

我们考虑 **crash** 的几种可能情况：
- 在第1步和第2步之间崩溃，会导致数据丢失，因为 **log** 还未被写入磁盘。这样的情况下，系统重启时不会有任何数据变更，就好像这些系统调用从未发生过一样；
- 在第2步和第3步之间崩溃，由于 **log** 已经被成功写入磁盘并有**commit**记录，所有的文件系统操作已经完成。这种情况下，在重启后系统能够保持一致性；
- 在install（第3步）过程中和第4步之前崩溃，系统在下次重启时会重新执行 **log**，**redo log** 会再次将 **log block** 的数据应用到文件系统。因为 **install log** 是幂等操作，即使重复多次也不会产生问题，这确保了系统在恢复时能够达到一致状态。

有一个有意思的问题：当正在 **commit log** 的时候 **crash** 了会发生什么？比如说你想执行多个写操作，但是只 **commit** 了一半。

在上面的第2步，执行 **commit** 操作时，你只会在记录了**所有的 write操作**之后，**才会执行commit操作**。所以在执行 **commit** 时，**所有的write操作必然都在log中**。而commit操作本身也有个有趣的问题，它究竟会发生什么？如我在前面指出的，commit操作本身只会写一个block。

文件系统通常可以这么假设，单个block或者单个sector的write是原子操作。这里的意思是，如果你执行写操作，要么整个sector都会被写入，要么sector完全不会被修改。所以sector本身永远也不会被部分写入，并且commit的 **目标sector** 总是包含了有效的数据。

而commit操作本身只是写log的header，如果它成功了只是在commit header中写入log的长度，例如5，这样我们就知道log的长度为5。这时crash 并重启，我们就知道需要**重新install 5个block的log**。如果 **commit header** 没能成功写入磁盘，那这里的数值会是0。**我们会认为这一次事务并没有发生过。**这里本质上是 **write ahead rule**，**它表示 logging系统在所有的写操作都记录在log中之前，不能install log。**

在执行 **commit** 操作时，系统会确保在**将所有的write操作记录到log之后再执行commit**。这种写前日志记录（**write-ahead logging**）确保了事务的持久性。

当执行 **commit** 时，系统会记录一个 **commit header** 到 **log** 中。这个 **header** 包含了当前事务的信息，比如事务的长度等。这个写入commit header的操作本身也是原子的，并且遵循着原子性和持久性的规则。**如果这个 commit header 成功写入磁盘，系统就知道了这个事务已经完整地被记录下来了。如果在写commit header时系统崩溃，那么在重启时会认为这个事务并没有发生过，因为这个commit操作本身的信息并没有被成功地记录下来。**

这种机制确保了在系统崩溃或故障时能够维持数据的一致性，并且能够通过log中的信息来进行恢复，使得系统能够重新执行那些还未完全记录的操作，从而达到事务的完整性和原子性。

xv6的 **log结构**如往常一样也是极其的简单。我们在最开始有一个 **header block**，也就是我们的 **commit record**，里面包含了：
- 数字**n**代表有效的 **log block** 的数量；
- 每个 **log block** 的实际对应的 **block** 编号。

之后就是 **log的数据**，也就是**每个block的数据**，依次为 **bn0** 对应的 **block** 的数据，**bn1** 对应的 **block** 的数据以此类推。这就是 **log** 中的内容，并且 **log** 也不包含其他内容。


![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/599ef52b6c164eb89805125337551fdd_1720253687327.png?token=ANB4BCOBAKVCKZKLUUAOVS3GRD6TK)

当文件系统在运行时，**在内存中也有header block的一份拷贝**，拷贝中也包含了 **n** 和 **block编号** 的数组。这里的 **block编号** 数组就是**log**数据对应的实际**block**编号，并且相应的 **block** 也会缓存在 **block cache** 中。

## 二、`log_write()` 函数

我们 **不应该在所有的写操作完成之前写入commit record**，这意味着文件系统操作必须表明事务的开始和结束。在 **xv6** 中，以创建文件的 **sys_open** 为例（在 `sysfile.c` 文件中）每个文件系统操作，都有**begin_op**和**end_op**分别表示事务的开始和结束。

```c
uint64
sys_open(void)
{
  ...

  begin_op(); // 事务开始

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op(); // 事务结束
      return -1;
    }
  }
  ...
  return fd;
}
```

事务中的所有写block操作具备原子性，这意味着这些写block操作要么全写入，要么全不写入。xv6 中的文件系统调用都有这样的结构，最开始是**begin_op**，**之后是实现系统调用的代码**，最后是**end_op**。在**end_op**中会实现**commit**操作。

在 **begin_op** 和 **end_op** 之间，磁盘上或者内存中的数据结构会更新，但是磁盘中的数据不会有实际的改变。即在 **end_op** 之前，是不会写入到实际的 **block**中。在 **end_op** 时，**我们会将数据写入到log中，之后再写入commit record或者log header**。

可以看到，这个事务看了一件事，就是**创建了inode**。看看 `create()`：

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

这段代码是一个函数 `create`，其作用是在文件系统中创建一个新的文件或目录，并返回对应的 inode 结构体指针。

1. **参数**：
   - `path`: 要创建文件或目录的路径。
   - `type`: 文件类型，可以是文件 (`T_FILE`) 或目录 (`T_DIR`)。
   - `major`: 设备文件的主设备号。
   - `minor`: 设备文件的次设备号。

2. **主要步骤**：
   - 函数首先通过 `nameiparent` 函数获取路径的父目录的 `inode` 结构体指针 `dp`，并且获取了要创建的文件或目录的名称 `name`。
   - 然后对父目录 `dp` 进行加锁 `ilock(dp)`，确保不会在并发情况下出现问题。
   - 接着，通过 `dirlookup` 函数在父目录中查找要创建的文件或目录。如果找到了同名的文件或目录，且其类型与要创建的类型相同（即均为文件或目录），则直接返回该 `inode` 指针。
   - 如果没有找到同名文件或目录，则通过 `ialloc` 函数分配一个新的 `inode`，表示新文件或目录。如果分配失败，则解锁父目录并返回 0。
   - 为新分配的 `inode` 设置一些基本属性，如 `major`、`minor`、`nlink`，然后通过 `iupdate` 函数将这些属性更新到磁盘上。
   - 如果要创建的是目录 (`T_DIR`)，则创建 `.` 和 `..` 两个目录项，分别表示当前目录和父目录。
   - 将新创建的文件或目录项写入父目录中，如果写入失败，则跳转到 `fail` 标签进行错误处理。
   - 如果创建的是目录 (`T_DIR`)，则在操作成功后更新父目录的 `nlink`，表示父目录包含了一个子目录。
   - 最后解锁父目录 `dp` 并返回新创建的 `inode` 结构体指针。

3. **错误处理**：
   - 在出现错误时，函数会跳转到 `fail` 标签处，将 `inode` 的 `nlink` 设为 0，通过 `iupdate` 将这个修改写入磁盘，然后释放这个 `inode` 的锁并解锁父目录，最后返回 0，表示创建过程出现了问题。

再看看它是怎么获取 **inode** 的：

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

可以看到，在拿到 **inode** 之后，并没有直接在磁盘中 **block**修改这个 **inode** 中的字段，而是初始化 **buffer**（内存中） 中的 **inode**、修改这个 **inode** 的 **type**。它并没有将这些改变写回到磁盘中，而是直接调用 `log_write()`，写到日志中。

来看看 `log_write()`：

```c
// Caller has modified b->data and is done with the buffer.
// Record the block number and pin in the cache by increasing refcnt.
// commit()/write_log() will do the disk write.
//
// log_write() replaces bwrite(); a typical use is:
//   bp = bread(...)
//   modify bp->data[]
//   log_write(bp)
//   brelse(bp)
void
log_write(struct buf *b)
{
  int i;
  acquire(&log.lock);
  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if (log.outstanding < 1)
    panic("log_write outside of trans");

  for (i = 0; i < log.lh.n; i++) {
    if (log.lh.block[i] == b->blockno)   // log absorption
      break;
  }
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // Add new block to log?
    bpin(b);
    log.lh.n++;
  }
  release(&log.lock);
}

```

这段代码是一个日志系统中的 `log_write` 函数。它的作用是记录被修改的数据块，并在日志中进行相应的标记，以便稍后将这些数据块的更改写入磁盘。

这个函数的主要步骤：

1. **函数作用**：
   - 记录被修改的数据块的块号，并将其标记在日志中，以便在后续的事务提交 (`commit()`) 或日志写入 (`write_log()`) 过程中执行磁盘写入。

2. **主要步骤**：
   - 首先，通过 `acquire(&log.lock)` 获取了日志的锁，确保在并发情况下不会出现问题。
   - 然后检查当前日志中记录的块号数量是否超出了规定的阈值 (`LOGSIZE`) 或者超出了日志的最大容量 (`log.size - 1`)，如果超出，则会产生  **panic**。
   - 接着检查 `log_write` 是否在一个事务 (`transaction`) 中进行调用，如果不是，则会产生 panic。
   - 然后，通过循环遍历当前日志中已经记录的块号，检查是否存在与要记录的块号相同的块号，如果存在则跳出循环，否则将当前要记录的块号加入到日志中。
   - 如果要记录的块号是新的，即不存在于当前日志中，则增加日志的块号数量 `log.lh.n`，并通过 `bpin(b)` 增加对应数据块的引用计数，表示这个数据块被日志所引用。

3. **释放资源**：
   - 最后通过 `release(&log.lock)` 释放了日志的锁，表示记录过程结束。

这个函数是日志系统中的一部分，用于记录被修改的数据块的块号，并在日志中进行标记，以备后续的磁盘写入操作。

可以看到它做的事情不多，就是简单地记录下需要写入的块号，如果是 **新block**，就把 **log.lh.n** 字段自增1。**log** 字段如下：


```c
// Contents of the header block, used for both the on-disk header block
// and to keep track in memory of logged block# before commit.
struct logheader {
  int n;
  int block[LOGSIZE];
};

struct log {
  struct spinlock lock;
  int start;
  int size;
  int outstanding; // how many FS sys calls are executing.
  int committing;  // in commit(), please wait.
  int dev;
  struct logheader lh;
};
```
以上就是 **log_write** 的全部工作了。**任何文件系统调用，如果需要更新block或者说更新block cache中的block，都会将block编号加在这个内存数据中（注，也就是log header在内存中的cache），除非编号已经存在。**

## 三、`end_op()` 函数

事务在完成之后，立即调用 `end_op()`：

```c
// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    do_commit = 1;
    log.committing = 1;
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```

在简单情况下，没有其他的文件系统操作正在处理中。这部分代码非常简单直观，首先调用了 `commit()`  函数。让我们看一下`commit()` 函数的实现：

```c
static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(0); // Now install writes to home locations
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log
  }
}
```


这个 `commit()` 函数是文件系统中用于提交事务的部分，它负责实际执行事务的提交操作：

1. **检查日志中是否有待提交的内容**：
   - 首先，它会检查日志头中的记录数 `log.lh.n` 是否大于 0，这表示是否存在需要提交到磁盘的修改。

2. **执行提交的具体步骤**：
   - 如果 `log.lh.n > 0`，表示有待提交的内容：
     - `write_log()`: 将缓存中被修改的块（blocks）写入到日志（log）中。这一步是将对磁盘上数据的修改记录在日志中，以便在需要时恢复数据。
     - `write_head()`: 将日志头（log header）写入磁盘。**这个动作是真正的提交（commit），表示事务的变更已经被记录到了日志中。**
     - `install_trans(0)`: 将修改写入到各自的位置，也就是将对文件系统的修改从日志应用到实际的位置上。这一步确保了数据的一致性。
     - `log.lh.n = 0`: 清零记录在日志头中的日志计数，表示事务已经成功提交并完成。
     - `write_head()`: 最后一次调用，此次是为了擦除日志中的事务记录，确保之前的事务已经完全提交，日志可以重新开始记录新的事务。

来看看  `write_log()` 干了什么：

```c
// Copy modified blocks from cache to log.
static void
write_log(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *to = bread(log.dev, log.start+tail+1); // log block
    struct buf *from = bread(log.dev, log.lh.block[tail]); // cache block
    memmove(to->data, from->data, BSIZE);
    bwrite(to);  // write the log
    brelse(from);
    brelse(to);
  }
}
```

这个 `write_log()` 函数是用来将缓存中被修改的块（blocks）复制到日志（log）中的过程：

1. **循环处理需要写入日志的块**：
   - `for` 循环遍历日志头中记录的需要写入日志的块数 `log.lh.n`。

2. **从缓存中读取和写入到日志中**：
   - `struct buf *to = bread(log.dev, log.start+tail+1);`: 从日志中读取一个指定位置的日志块（log block）。
   - `struct buf *from = bread(log.dev, log.lh.block[tail]);`: 从缓存中读取一个需要被写入到日志的块（cache block）。**这里需要注意的是，这个块是缓存块，它已经被修改好了，现在准备要将它复制到日志块中，而且还要写到硬盘中。**
   - `memmove(to->data, from->data, BSIZE);`: 将缓存块中的数据复制到日志块中。这一步实际上是将被修改的数据从缓存中拷贝到了日志中，以记录这些修改。
   - `bwrite(to);`: 将这个日志块写入磁盘，这一步是将缓存块的内容写入到日志中，记录了被修改的数据。
   - `brelse(from); brelse(to);`: 释放对缓存块和日志块的引用，表示不再需要这些块的数据。

我们来看看 `bwrite()`：

```c
// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  virtio_disk_rw(b, 1);
}
```

之前有提到，`bwrite()`是真正向硬盘中写东西的函数，所以 `write_log()` 的作用，就是将缓存中的数据，或者说需要修改的缓存块，写到日志中，而且是硬盘中的 **log block** 中！

`write_head()`：

```c
// Write in-memory log header to disk.
// This is the true point at which the
// current transaction commits.
static void
write_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *hb = (struct logheader *) (buf->data);
  int i;
  hb->n = log.lh.n;
  for (i = 0; i < log.lh.n; i++) {
    hb->block[i] = log.lh.block[i];
  }
  bwrite(buf);
  brelse(buf);
}
```
这个函数 `write_head()` 的功能是将内存中的日志头信息写入硬盘中的日志块中，**实现当前事务的真正提交**。

具体步骤如下：

1. **读取日志块**：
   - `struct buf *buf = bread(log.dev, log.start);`: 从磁盘中读取日志块到内存中的缓冲区 `buf` 中。

2. **更新日志头信息**：
   - `struct logheader *hb = (struct logheader *) (buf->data);`: 将缓冲区中的**数据强制转换为日志头结构体** （C语言的强转机制，就是这么暴力！），以便更新其中的信息。
   - `hb->n = log.lh.n;`: 将当前事务中记录的块数更新到日志头中。
   - `for (i = 0; i < log.lh.n; i++) { hb->block[i] = log.lh.block[i]; }`: 将当前事务中记录的每个块的块号（block number）更新到日志头中。

3. **将更新后的日志头信息写入硬盘**：
   - `bwrite(buf);`: 使用 `bwrite()` **函数将更新后的日志头数据写入硬盘中的日志块，这个过程实际上是将当前事务的信息提交到磁盘中的日志。**

4. **释放缓冲区**：
   - `brelse(buf);`: 释放对日志块的引用，表示不再需要这个块的数据。

这个函数的作用就是将当前事务的日志头信息持久化到磁盘中，标志着事务的正式提交。

`install_trans(int recovering)`:

```c
// Copy committed blocks from log to their home location
static void
install_trans(int recovering)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block
    struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
    memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    if(recovering == 0)
      bunpin(dbuf);
    brelse(lbuf);
    brelse(dbuf);
  }
}
```

这个函数 `install_trans()` 的主要功能是**将已经提交的块（blocks）从日志（log）中复制到它们原本的位置（home location）。**

具体步骤如下：

1. **循环处理需要安装的块**：
   - `for (tail = 0; tail < log.lh.n; tail++) {`: 使用循环遍历日志头中记录的需要安装的块数 `log.lh.n`。

2. **从日志和目标位置读取块**：
   - `struct buf *lbuf = bread(log.dev, log.start+tail+1);`: 从日志中读取一个指定位置的日志块（log block）。
   - `struct buf *dbuf = bread(log.dev, log.lh.block[tail]);`: 从磁盘中读取一个目标位置的块（destination block），这个块是在日志中记录的需要被修改的块。

3. **复制块数据到目标位置**：
   - `memmove(dbuf->data, lbuf->data, BSIZE);`: 将日志块中的数据复制到目标位置的块中，实际上是将日志中记录的修改写入到了原本的位置。

4. **将更新后的块数据写回磁盘**：
   - `bwrite(dbuf);`: 将被修改后的块数据写回到磁盘中，这一步是将日志中记录的修改应用到了磁盘的相应位置。

5. **解除引用和释放资源**：
   - `bunpin(dbuf);`: 如果正在恢复过程中，则解除对目标位置块的引用。这可能会是一个引用计数的操作，确保释放相关资源。
   - `brelse(lbuf); brelse(dbuf);`: 释放对日志块和目标位置块的引用，表示不再需要这些块的数据。

这个函数实现了事务的安装。

回到 commit()函数：

```c
static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(0); // Now install writes to home locations
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log
  }
}
```

在**commit**函数中，**install**结束之后，**会将log header中的n设置为0，再将log header写回到磁盘中。将n设置为0的效果就是清除log**。这里，才算是 `commit()` 真正结束！

## 四、文件恢复
我们看一下发生在xv6的启动过程中的文件系统的恢复流程。当系统crash并重启了，在xv6启动过程中做的一件事情就是调用`initlog()` 函数：

```c
void
initlog(int dev, struct superblock *sb)
{
  if (sizeof(struct logheader) >= BSIZE)
    panic("initlog: too big logheader");

  initlock(&log.lock, "log");
  log.start = sb->logstart;
  log.size = sb->nlog;
  log.dev = dev;
  recover_from_log();
}
```
从 **superblock** 中拿到 **log** 初始化所需的数据后，开始恢复：
```c
static void
recover_from_log(void)
{
  read_head();
  install_trans(1); // if committed, copy from log to disk
  log.lh.n = 0;
  write_head(); // clear the log
}
```
首先从硬盘中的 **log block** 中的 **log header** 数据恢复到内存中的 **log header**，还恢复了一点 **log** 信息：

```c
// Read the log header from disk into the in-memory log header
static void
read_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *lh = (struct logheader *) (buf->data);
  int i;
  log.lh.n = lh->n;
  for (i = 0; i < log.lh.n; i++) {
    log.lh.block[i] = lh->block[i];
  }
  brelse(buf);
}
```

然后就再一次 `install_trans()`，将日志中的数据复制到 **buffer** 中，然后写回硬盘。

这就是恢复的全部流程。如果我们在 `install_trans`函数中又crash了，也不会有问题。

## 五、Log 写磁盘流程
为了追踪写磁盘，修改 `bwrite()` 函数：
```c
// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");

  printf("bwrite: %d\n",b->blockno);
  virtio_disk_rw(b, 1);
}
```
然后运行以下命令（ **文件x** 之前存在，因此开始将它删了）：

```bash
xv6 kernel is booting

hart 1 starting
hart 2 starting
bwrite: 2
init: starting sh
$ rm x
bwrite: 3
bwrite: 4
bwrite: 5
bwrite: 6
bwrite: 2
bwrite: 46
bwrite: 32
bwrite: 33
bwrite: 45
bwrite: 2
$ echo hi > x
1	bwrite: 3
2	bwrite: 4
3	bwrite: 5
4	bwrite: 2
5	bwrite: 33
6	bwrite: 46
7	bwrite: 32
8	bwrite: 2

9	bwrite: 3
10	bwrite: 4
11	bwrite: 5
12	bwrite: 2
13	bwrite: 45
14	bwrite: 908
15	bwrite: 33
16	bwrite: 2

17	bwrite: 3
18	bwrite: 4
19	bwrite: 2
20	bwrite: 908
21	bwrite: 33
22	bwrite: 2
```

可以看到，系统刚启动，就往磁盘中 **log block2**中写东西了。根据这些输出信息，我们随意畅谈，只要合理就好。

比如，对于 `rm x`，很明显，需要修改多个 log block，然后修改根目录的 block，然后还要修改 inode block、bitmap block。这都能与 shell 打印的信息吻合。

我们再看看 `echo hi > x`。

```bash
1	bwrite: 3
2	bwrite: 4
3	bwrite: 5
```
这 3 行都是 log block，记录的 3 个操作，写入到了这里。

```bash
4	bwrite: 2
```

这个是 **log header**，这条是 **commit** 记录，因为只有 `commit()` 中才会执行 `write_head()` 函数。

```bash
5	bwrite: 33
6	bwrite: 46
7	bwrite: 32
```

这里实际就是将前3行的 **log data** 写入到实际的文件系统的**block**位置，这里 **install log**，还写了**entry**（创建了**文件x**），顺便修改了根目录的 **inode** 字段的 **size**。

```bash
4	bwrite: 2
5	bwrite: 33
6	bwrite: 46
7	bwrite: 32
8	bwrite: 2
```
连起来看，就很清楚，完美对应`commit()`：

```c
static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(0); // Now install writes to home locations
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log
  }
}
```
因此，第 8 行的 bwrite 2 就是在清除log。

```bash
9	bwrite: 3
10	bwrite: 4
11	bwrite: 5
12	bwrite: 2
13	bwrite: 45
14	bwrite: 908
15	bwrite: 33
16	bwrite: 2
```
这里也是，记录 3 组操作之后，然后 `commit()`。这里修改了位图，之后写了 `hi`,并修改了 **文件x** 的 **inode** 中的 **size** 等字段（**block33**），

```bash
17	bwrite: 3
18	bwrite: 4
19	bwrite: 2
20	bwrite: 908
21	bwrite: 33
22	bwrite: 2
```

这里记录了两组操作，也就 `commit()`了，最后清除 **log**。这里应该在写最后的换行符，同时修改文件大小。

所以以上就是 **xv6** 中文件系统的 **logging** 介绍，即使是这么一个简单的 **logging** 系统也有一定的复杂度。这里立刻可以想到的一个问题是，通过观察这些记录，这是一个很有效的实现吗？很明显不是的，因为数据被写了两次。如果我写一个大文件，我需要在磁盘中将这个大文件写两次。所以这必然不是一个高性能的实现，**为了实现 Crash safety 我们将原本的性能降低了一倍。**



*全文完，感谢阅读。*












































