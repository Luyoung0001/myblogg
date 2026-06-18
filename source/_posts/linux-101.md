---
title: "Linux 101：适合入门的学习资料与实践路径"
date: 2024-08-07 17:22:33
tags:
  - "Linux"
  - "learning roadmap"
  - "shell"
categories:
  - "Tooling"
---

<!--more-->

## 前言
本文是对 Linux101 这本及其优秀的 Linux 入门的一些心得体会，链接:[Linux101](https://101.ustclug.org/)

## 软件安装和文件操作
### apt
建议用 apt 安装，apt 作为前端，它会自动处理软件包之间的依赖关系、升级软件包以至升级发行版，自动处理升级发行版所需的依赖关系等等。但是需要明白，真正执行安装软件的是 dpkg。

另外，dpkg -i 也可以安装软件，但是它不会处理依赖，如果安装报错，应该使用 apt -f install 来处理依赖。因此为了方便，建议 apt install 软件。

### tar
tar 是打包工具，常见的操作如：
```bash
$ tar -c -f target.tar file1 file2 file3 # 将 file1、file2、file3 打包为 target.tar
$ tar -cz -f target.tar.gz file1 file2 file3 # 将 file1、file2、file3 打包，并使用 gzip 算法压缩，得到压缩文件 target.tar.gz
$ tar -xz -f target.tar.gz -C test/ # 将压缩文件 target.tar.gz 解压到 test 目录中
```
### man & tldr(too long didn't read)

这两个命令很好用，都是很好的命令解释工具。比如：
```bash
tldr tldr
tldr
Display simple help pages for command-line tools from the tldr-pages project.Note: the --language and --list options are not required by the client specification, but most clients implement them.More information: https://github.com/tldr-pages/tldr/blob/main/CLIENT-SPECIFICATION.md#command-line-interface.

 - Print the tldr page for a specific command (hint: this is how you got here!):
   tldr {{command}}

 - Print the tldr page for a specific subcommand:
   tldr {{command}} {{subcommand}}

 - Print the tldr page for a command in the given [L]anguage (if available, otherwise fall back to English):
   tldr --language {{language_code}} {{command}}

 - Print the tldr page for a command from a specific [p]latform:
   tldr --platform {{android|common|freebsd|linux|osx|netbsd|openbsd|sunos|windows}} {{command}}

 - [u]pdate the local cache of tldr pages:
   tldr --update

 - [l]ist all pages for the current platform and common:
   tldr --list

 - [l]ist all available subcommand pages for a command:
   tldr --list | grep {{command}} | column
```
细节：为什么 apt 需要root 权限，因为在安装的时候，需要向系统目录写入二进制文件、用户手册等等，这里必须使用到 root 权限。同理 make install 一样，必须使用sudo:
```bash
$ make
$ sudo make install
```
在这个过程中，make 命令会将编译好的二进制文件拷贝到相对应的安装目录，拷贝用户手册等。

### vim
这个只需要会用i、wq、q!、Esc就行了。

## 进程、前后台、服务与例行性任务

### 常见的信号

- SIGINT: 由 Ctrl c 产生
- SIGTERM: 由 kill <pid> or pkill <process name> 产生
- SIGKILL: 很强，kill -9
- SIGSTOP: Ctrl z 产生，停止进程
- SIGCOTN: fg、bg
- SIGHUP: 脱离会话的时候，会产生这个信号，要想继续执行，就必须得另开启会话

另外需要注意的是，kill -9 并不能杀死僵尸进程，因为僵尸进程已经不是传统意义上的进程了，它只是一种数据存档，造成的影响就是占用 pid。

### 进程组

假设有一个 shell zsh，在 zsh 中输入 ls、cat等等命令，都会形成一个以 zsh 为组长的进程组。发给一个进程组的信号将被所有属于该组的进程接收，意义就是停止整个任务整体，比如终止掉 zsh，所有的进程都会收到 SIGHUP信号而终止，除非某些进程用 nohup 执行。

### 会话

会话，是以用户登录而出现的概念。当用户进入shell，比如 zsh 中，就会以本 zsh 为会话首进程展开本次会话。事实上，zsh、进程组、会话组都是一个 id，这三个 id 相同。
前台 (foreground) 与后台 (background)，本质上决定了是否需要与用户交互，对于单独的一个 shell，只能有一个前台进程（组），其余进程只能在后台默默运行，上述中若干进程组，正是前台进程组和后台进程组的概称。这点很好理解，不能让两个进程同时占用前台，导致打印或者输入异常。

### 守护进程
创建会话必须调用 setsid()。当登录到 zsh 中，zsh 本身就是一个 leader of this session，然而 zsh 中 fork 出来的其它进程是不能调用setsid()展开新的会话的：

- DESCRIPTION
       setsid() creates a new session if the calling process is not a process group leader.  The calling
       process is the leader of the new session (i.e., its session ID is made the same  as  its  process
       ID).   The  calling  process  also becomes the process group leader of a new process group in the
       session (i.e., its process group ID is made the same as its process ID).

       The calling process will be the only process in the new process group and in the new session.

       Initially, the new session has no controlling terminal.  For details of how a session acquires  a
       controlling terminal, see credentials(7).

所以想要创建一个独立的进程，方法就是在 shell 中创建一个进程 A，A 再创建一个 进程 B，然后 B 调用 setsid()开启新的会话，并成为绘画组组长，A 再退出。当然不打算退出 shell 的时候有更加直接的方法，比如：bash -c "ping localhost &" &。

当 zsh fork 一个子进程 bash 的时候，bash 执行 ping localhost & 之后退出，ping 由于 ping localhost & 是 bash 的子进程，bash 退出后，按理说 ping localhost & 也会结束，但是它是后台进程，后台进程并不会收到 bash 父进程退出的影响（在这种情况下，bash 并不是绘会话组组长，因此bash 退出，并不会影响它的后台子进程）。因此 ping 进程继续会执行，除非 zsh 结束。

当在 Bash 或其他类 Unix 的 shell 中使用 `&` 将命令置于后台运行时，这个后台进程具有一些特定的特点和行为模式。根据你的问题，我们主要关注后台进程的生命周期、它与父进程（如 Bash）的关系，以及父进程结束时会发生什么。这里是详细的解释：

### 后台进程的特点

1. **独立性**：
   - 后台进程能够独立于终端运行，用户可以继续在同一个 shell 中执行其他命令，而后台进程则在不干扰前台（用户交互式）工作的情况下执行。

2. **与终端的关系**：
   - 即使一个进程被放在后台执行，它通常仍然与启动它的 shell 绑定，接收同一个终端的输入和输出。这意味着它们依旧可以向终端输出数据（除非输出被重定向）。

3. **会话和进程组**：
   - 后台进程通常属于启动它的 shell 所在的会话和进程组。这个会话管理着终端及其所有前台和后台进程，可以向它们发送信号。

### 进程结束对后台进程的影响

- **父进程结束**：
  - 当启动后台进程的 Bash shell 正常结束时，通常对后台进程没有直接影响，因为后台进程已经在独立的子进程中运行。
  - 但是，如果终端的会话结束（例如关闭终端窗口），会话领导者（通常是最初启动的 shell）会收到 `SIGHUP`（挂断信号）。默认情况下，这会导致所有该会话下的子进程，包括后台进程，也接收到 `SIGHUP`，除非这些进程已经通过某种方式（如调用 `nohup` 或重写 `SIGHUP` 信号处理）明确地脱离了这种行为。

### 使用 `nohup` 避免后台进程退出

- **`nohup` 命令**：
  - 如果你希望确保即使 Bash 结束或用户注销，后台进程如 `ping localhost` 也不会退出，可以使用 `nohup` 命令。`nohup` 可以防止进程接收到 `SIGHUP` 信号。
  - 示例命令：`nohup ping localhost &`
  - 这会启动 `ping localhost`，并确保即使终端关闭或用户注销，它也会继续运行。


当 Bash 执行 `ping localhost &` 后，即使 Bash 结束，`ping localhost` 也不会自动退出，除非 Bash 所在的会话被关闭（例如关闭终端窗口）。在这种情况下，除非对 `SIGHUP` 信号有特别的处理，否则 `ping` 会因为接收到 `SIGHUP` 而退出。使用 `nohup` 是一种常见的防止这种情况发生的方法。

### 终端与控制台

早期的终端：物理设备，控制台也是早期的概念，它是一种特殊的终端，它可以干其它终端不能干的事情，比如维修系统。

现在的终端是指图形界面，用户要输入命令和接受信息必须使用终端模拟程序，比如 gnome_terminal。终端模拟器与 pty 建立联系，用户输入到终端模拟器中的字符转发给 pty的前端 、pty后端就是我们说的 tty，tty 拿到字符后，交给 shell 进行处理。具体看下：
进一步澄清这个过程中涉及的各个组件之间的关系和交互，以便更好地理解终端模拟器、pty（伪终端）、tty 和 shell 如何协同工作。

### 终端模拟器、pty、tty 和 shell 的关系

1. **终端模拟器（如 gnome-terminal）**：
   - 终端模拟器是一个在图形用户界面中运行的应用程序，它模拟了传统的物理终端的功能。终端模拟器提供了用户与系统交互的界面，允许用户输入命令并显示输出。

2. **伪终端（pty）**：
   - 伪终端是一种软件驱动的终端设备，它模拟了物理终端（tty）的行为。pty 通常分为主设备（pty master）和从设备（pty slave）。
   - 主设备由终端模拟器控制，而从设备提供一个接口，表现得像是一个真实的终端（tty）。

3. **连接 tty**：
   - 在这里，tty 实际上是指由 pty 的从设备所模拟的那部分。在现代系统中，当我们通常谈论到 tty 时，往往实际上是指 pty 的从设备。
   - 终端模拟器通过 pty 与 shell 进行交互。用户在终端模拟器中的输入首先传递到 pty 的主设备，然后由主设备转发到从设备。

4. **Shell**：
   - Shell（如 bash、zsh 等）通常启动并运行在 pty 的从设备上。Shell 接收从 pty 从设备来的输入（用户的命令），执行这些命令，并将输出返回到同一个 pty 从设备。
   - 这些输出随后由 pty 的主设备接收，并通过终端模拟器显示给用户。

### 过程概述

- 当用户在终端模拟器（如 gnome-terminal）中键入命令时，这些输入数据被发送到伪终端的主设备。
- 主设备将这些输入传递到从设备，从设备再将输入交给运行在其上的 shell。
- Shell 解析和执行命令，然后将结果输出回 pty 的从设备。
- 输出数据由主设备接收，并通过终端模拟器展示给用户。

需要明确，现代的 Linux/Unix 系统中，通常说的 tty 实际上在很多情况下是指 pty 的从设备，特别是在涉及到图形界面中的终端模拟器时。通过这种方式，终端模拟器可以模拟出一个看似真实的终端（tty）环境，与用户的 shell 交互，而用户实际上是在使用由软件实现的伪终端。这个机制使得终端模拟器非常灵活和强大。

## 用户与用户组、文件权限、文件系统层次结构

### 用户
可以在/etc/passwd 中修改用户登录的默认 shell，当然也可以用其它命令，比如 chsh：

```txt
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
（中间内容省略）
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
ustc:x:1000:1000:ustc:/home/ustc:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
mysql:x:111:116:MySQL Server,,,:/nonexistent:/bin/false
```

我当时以 sudo 用户安装 zsh 的时候，由于粗心，chsh 输入错了 zsh 的路径，导致退出后没办法ssh 连接，后来才发现是默认 shell zsh在指定路径找不到，还好我还有一个 root 用户，不然只能重置云服务器了。

当 sudo 组的用户利用 sudo 提权操作时，默认指定的是 root 用户。sudo 用户可以通过 su 来将 shell 提权到 root 权限，因此可以限制某用户禁止使用 su 命令。

事实上，你可以添加很多拥有root 权限的组，比如 admin、AAA，只要你在 sudoers 中定义：
```bash
# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

# Allow members of group admin to execute any command
%admin   ALL=(ALL:ALL) ALL

# Allow members of group AAA to execute any command
%AAA   ALL=(ALL:ALL) ALL

```

当使用 sudo 提权时，其实使用的是root 用户的身份来执行命令的：
```bash
$ ~
$ sudo whoami
root
$ ~
$ whoami
luyoung

```

### 文件系统的特殊权限位
```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 59976 Feb  6  2024 /usr/bin/passwd

```
可以看到，该命令对于 root 用户有s 权限，事实上，还有一些命令也拥有 s 权限：
```bash
luyoung at luyoungUbt in /etc
$ ls -l /usr/bin/su
-rwsr-xr-x 1 root root 55680 Apr  9 23:32 /usr/bin/su

luyoung at luyoungUbt in /etc
$ ls -l /usr/bin/sudo
-rwsr-xr-x 1 root root 232416 Apr  4  2023 /usr/bin/sudo
```

这代表此文件有 setuid 特殊权限位。在你执行 passwd 的时候，它的实际权限和 root 一样，只是它知道，执行它的人是你（而非 root），所以只提供修改你自己的密码的功能。

### cd
cd 是 shell 中的一个 build_in 命令，因此在 sudo cd 的时候，会提示找不到命令。cd 之所以会失败，是因为当前用户对该目录没有执行权限，想要 cd 进去，必须以 root 身份，ls 不是 build_in 命令，因此可以执行成功：
```bash
luyoung at luyoungUbt in /
$ ls -l /root
ls: cannot open directory '/root': Permission denied

luyoung at luyoungUbt in /
$ sudo ls -l /root
total 4
drwx------ 5 root root 4096 May 18  2022 snap
```

另外，就算将 cd 设计成一个非 build in 命令也没用，shell 执行cd 后，改变的仅仅是子进程的目录，执行完成后返回 shell，依然在原目录。


