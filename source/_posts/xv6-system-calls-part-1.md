---
title: "xv6 系统调用笔记（上）：exit、wait 与进程控制"
date: 2023-12-10 23:02:24
tags:
  - "xv6"
  - "system call"
  - "process"
categories:
  - "Operating Systems"
---

<!--more-->

## 〇、前言
本文将会结合源代码谈论 **exit**、**wait**、**kill** 这三个系统调用。

## 一、exit 系统调用
以下是 `exit()`的源码：

```c
// Exit the current process.  Does not return.
// An exited process remains in the zombie state
// until its parent calls wait().
void
exit(int status)
{
  struct proc *p = myproc();

  if(p == initproc)
    panic("init exiting");

  // Close all open files.
  for(int fd = 0; fd < NOFILE; fd++){
    if(p->ofile[fd]){
      struct file *f = p->ofile[fd];
      fileclose(f);
      p->ofile[fd] = 0;
    }
  }

  begin_op();
  iput(p->cwd);
  end_op();
  p->cwd = 0;

  acquire(&wait_lock);

  // Give any children to init.
  reparent(p);

  // Parent might be sleeping in wait().
  wakeup(p->parent);

  acquire(&p->lock);

  p->xstate = status;
  p->state = ZOMBIE;

  release(&wait_lock);

  // Jump into the scheduler, never to return.
  sched();
  panic("zombie exit");
}

// Wait for a child process to exit and return its pid.
// Return -1 if this process has no children.
```

这个函数做的第一个检查就是判断这个被退出的函数是不是 `initproc`，众所周知，这个线程是 `pid=1` 的线程，是不能退出的。

然后就是关闭这个进程的文件了。这里面涉及到引用计数，`fileclose()`就减少这个文件的引用计数。

如果一个进程要退出，但是它又有自己的子进程，接下来需要设置这些子进程的父进程为`initproc`：

```c
// Pass p's abandoned children to init.
// Caller must hold wait_lock.
void
reparent(struct proc *p)
{
  struct proc *pp;

  for(pp = proc; pp < &proc[NPROC]; pp++){
    if(pp->parent == p){
      pp->parent = initproc;
      wakeup(initproc);
    }
  }
}
```
如果父进程退出了，那么子进程就不再有父进程，当它们要退出时就没有对应的父进程的`wait()`。这些失去父进程的子进程，在后面的时间段里一旦调用了 `exit()`，而没有父进程在执行 `wait()`，后面我们会看到，这些子进程将会变成僵尸进程。所以最好在`exit()`函数中，为即将**exit**进程的子进程重新指定父进程为 `initproc` 是很明智的。正常来说，每一个 `exit()`的进程，父进程都在调用 `wait()`（这也不一定，这取决于程序员是否有好的编程习惯）。

问题是，当一个父进程结束时，没有调用 `exit()`，这自然地就导致了它的子进程没有过继给 **initproc**，这时候，就产生了孤儿进程。当孤儿进程调用 `exit()` 时，由于没有父进程，自然就成了僵尸进程。

我没有在 xv6 中看到它是如何解决这一问题的，但是在 **LINUX** 中，它是通过将没有父进程的孤儿进程自然地过继给 **initproc**，即 **initproc** 不停地在找孤儿进程，找到后就完成过继。


之后唤醒父进程，父进程从 `wait()` 处醒来，继续执行。

进程的状态被设置为**ZOMBIE**。

现在进程还没有完全释放它的资源，所以它还不能被重用。所谓的进程重用是指，我们期望在最后，进程的所有状态都可以被一些其他无关的fork系统调用复用，但是目前我们还没有到那一步。
现在我们还没有结束，因为我们还没有释放进程资源。我们在还没有完全释放所有资源的时候，通过调用 `sched()` 函数进入到调度器线程。

到目前位置，进程的状态是**ZOMBIE**，并且进程不会再运行，因为调度器只会运行**RUNNABLE**进程。同时进程资源也并没有完全释放，如果释放了进程的状态应该是**UNUSED**。但是可以肯定的是进程不会再运行了，因为它的状态是**ZOMBIE**。所以调度器线程会决定运行其他的进程。

## 二、wait 系统调用
以下是 `wait()`的源码：
```c
// Wait for a child process to exit and return its pid.
// Return -1 if this process has no children.
int
wait(uint64 addr)
{
  struct proc *pp;
  int havekids, pid;
  struct proc *p = myproc();

  acquire(&wait_lock);

  for(;;){
    // Scan through table looking for exited children.
    havekids = 0;
    for(pp = proc; pp < &proc[NPROC]; pp++){
      if(pp->parent == p){
        // make sure the child isn't still in exit() or swtch().
        acquire(&pp->lock);

        havekids = 1;
        if(pp->state == ZOMBIE){
          // Found one.
          pid = pp->pid;
          if(addr != 0 && copyout(p->pagetable, addr, (char *)&pp->xstate,
                                  sizeof(pp->xstate)) < 0) {
            release(&pp->lock);
            release(&wait_lock);
            return -1;
          }
          freeproc(pp);
          release(&pp->lock);
          release(&wait_lock);
          return pid;
        }
        release(&pp->lock);
      }
    }

    // No point waiting if we don't have any children.
    if(!havekids || killed(p)){
      release(&wait_lock);
      return -1;
    }

    // Wait for a child to exit.
    sleep(p, &wait_lock);  //DOC: wait-sleep
  }
}
```

首先进入一个无限循环里，最开始标记它的孩子数目为 **0**。然后遍历，如果某个进程的父进程是它，那么标记它的还子进程数目为 **1**。之后继续检查它的状态是否是 **ZOMBIE**，如果是，那么就继续释放资源（上一步 `exit()` 没有完全是释放完全的）：

```c
// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```
直到子进程 `exit()` 的最后，它都没有释放所有的资源，因为它还在运行的过程中，所以不能释放这些资源。相应的其他的进程，也就是父进程，释放了运行子进程代码所需要的资源。这样的设计可以让我们极大的精简 `exit()` 的实现。

## 三、kill 系统调用
以下是`kill()`的源码：

```c
// Kill the process with the given pid.
// The victim won't exit until it tries to return
// to user space (see usertrap() in trap.c).
int
kill(int pid)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++){
    acquire(&p->lock);
    if(p->pid == pid){
      p->killed = 1;
      if(p->state == SLEEPING){
        // Wake process from sleep().
        p->state = RUNNABLE;
      }
      release(&p->lock);
      return 0;
    }
    release(&p->lock);
  }
  return -1;
}
```

可以看到，`kill()`仅仅做了很简单的事情。它拿到某个 **pid** 之后，将 **kill** 标志位置为 **1**，如果它处于 **SLEEPING** 状态，就将其置于 **RUNNABLE**，然后就结束了。

而目标进程运行到内核代码中能安全停止运行的位置时，会检查自己的 **killed** 标志位，如果设置为**1**，目标进程会自愿的执行**exit**系统调用。

所以当你 **kill** 一个僵尸进程，僵尸进程是无法被 **kill** 掉的，因为它的状态不是 **SLEEPING**，无法转为 **RUNNABLE**。

## 四、总结
这篇文章主要介绍了在 **xv6** 操作系统中的三个系统调用：`exit()`、`wait() `和 `kill()`。下面是这些系统调用的主要功能：

### 1、exit() 系统调用
`exit()` 用于终止当前进程的执行，并进行一系列清理工作。
它关闭当前进程打开的文件，释放当前进程的工作目录资源，设置进程状态为 **ZOMBIE**，并且如果有子进程，将这些子进程的父进程设置为 **initproc**。
最终，进程进入 **ZOMBIE** 状态并放入等待队列，等待父进程调用 `wait()` 进行回收。
### 2、wait() 系统调用
`wait()` 用于等待子进程退出，并返回子进程的 **PID**。
它首先遍历当前进程的子进程，检查是否有子进程处于 **ZOMBIE** 状态，如果有，释放资源并返回子进程的 **PID**。
如果当前进程没有子进程或者子进程尚未退出，则进入睡眠状态等待子进程退出。
### 3、kill() 系统调用
`kill()` 用于向指定 **PID** 的进程发送信号，设置被杀进程的 **killed** 标志为 **1**。
如果目标进程处于 **SLEEPING** 状态，它会被唤醒，状态转换为 **RUNNABLE**，然后等待检查 **killed** 标志位进行 **exit**。
被杀的进程会在下次进入内核代码时检查自己的 **killed** 标志，如果设置为 **1**，则自愿执行 **exit** 系统调用。
这些系统调用是操作系统中非常基础和重要的部分，它们负责管理进程的生命周期和资源释放，确保系统中的资源能够被充分利用和重用。

*嗯，大模型总结地真好，全文完，感谢阅读。*







