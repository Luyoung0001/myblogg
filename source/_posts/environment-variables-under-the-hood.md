---
title: "环境变量探究：进程视角下的存储与访问"
date: 2023-01-10 20:42:18
tags:
  - "environment variables"
  - "process"
  - "Unix"
categories:
  - "Operating Systems"
---

<!--more-->

## 〇、环境变量是什么？
在使用 Windows 操作系统时我发现，在安装一个新的工具包后，是无法直接使用这个工具包的。计算机相关专业的学生在配置 VScode 时会感受颇深，他们往往得下载一个 MinGW，之后就得配置环境变量。配好之后，就可以在 cmd 窗口输入`gcc -v`来看是否配置成功了。

在不同的平台，比如 macOS下，装好工具之后，也得把工具的目录导入到一个`.zshrc`中，之后键入`source`命令，就可以是当前环境变量立即生效，然后就可以使用工具了。

所以，我对环境变量的理解，大致就形成了这样的印象：
- 下载好工具之后，终端得在当前目录中搜索这个工具，如果搜索不到，就回去 `PATH`中（环境变量）的目录中找，如果环境变量没有配置好或者找不到目录，那么就会弹出`zsh: command not found：gcc`。
- 所以一个工具想要执行，就一定得首先找到它，环境变量本质上就是在为操作系统指路。

如果我们希望执行某个程序时不用输入完整路径，通常就会在~/.bashrc中的PATH中加上相应的路径，这样以后就只用输入程序名了。

维基百科中给出的解释为：

> 环境变量是一个动态命名的值，可以影响计算机上进程的行为方式。例如一个正在运行的进程可以查询TEMP环境变量的值，以发现一个合适的位置来存储临时文件，或者查询HOME或USERPROFILE变量，以找到运行该进程的用户所拥有的目录结构。
>
这个解释就非常到位，和我的理解大差不差。

## 一、工具
- macOS
- VScode

## 二、探究
如果`.zshrc`中的路径就是环境变量，那么我们打开看看：

> export PATH="/opt/homebrew/opt/binutils/bin:$PATH"

可以发现，里面只有一点内容。那么其它的环境变量在哪儿？其实，存放环境变量的文件不止`.zshrc`，还有如下的：

```bash
# 系统级别
/etc/profile
/etc/paths

# 用户级别
~/.bash_profile
~/.bash_login
~/.profile

~/.bashrc（或者~/.zshrc）
```
可以打开看看：

```bash
******** /etc % cat paths
/usr/local/bin
/usr/bin
...
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME PATH CLASSPATH

```
可以发现，这里面确实放着环境变量。事实上，`**environ`就是C语言运行时环境提供的对进程环境变量的访问指针，指向一个个环境变量字符串。我们可以用一个简单的程序直接把所有的C语言运行时环境的环境变量打印出来：

```c
#include <stdio.h>
extern char** environ;
int main(int argc, const char* argv[]) {
    int i = 0;
    while (environ[i]) {
        printf("%s\n", environ[i]);
        i++;
    }
}
```
打印结果为：

```bash
USER=用户名
MallocNanoZone=0
__CFBundleIdentifier=com.microsoft.VSCode
COMMAND_MODE=unix2003
...
ZDOTDIR=/Users/用户名
USER_ZDOTDIR=/Users/用户名
...
_=/Users/用户名/CProjects/Test/./a.out
```
当然，`**environ`似乎没有多少说服力，我们可以修改一下程序：

```c
#include <stdio.h>
extern char** environ;
int main(int argc, const char* argv[]) {
    printf("environment variables:\n");
    int i = 0;
    while (environ[i]) {
        printf("%p\t%s\n", environ[i], environ[i]);
        i++;
    }

    printf("argv:\n");
    for (int i = 0; i < argc; i++) {
        printf("%p\t%s\n", argv[i], argv[i]);
    }
}
```
键入：

```bash
./a.out aa bbb ccc ddd eee fff
```

打印结果为：

```c
**** Test % ./a.out aaa bbb ccc ddd eee fff
environment variables:
0x16d7efa38     USER=luliang
0x16d7efa45     MallocNanoZone=0
0x16d7efa56     __CFBundleIdentifier=com.microsoft.VSCode
0x16d7efa80     COMMAND_MODE=unix2003
0x16d7efa96     LOGNAME=luliang
0x16d7efaa6     PATH=/usr/local/opt/binutils/bin:/opt/homebrew/opt/binutils/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/Applications/VMware Fusion.app/Contents/Public:/Library/Apple/usr/bin
0x16d7efb7f     SSH_AUTH_SOCK=/private/tmp/com.apple.launchd.2NSRW4JaeV/Listeners
...
0x16d7efe12     INFOPATH=/opt/homebrew/share/info:
0x16d7efe35     _=/Users/luliang/CProjects/Test/./a.out
argv:
0x16d7efa18     ./a.out
0x16d7efa20     aaa
0x16d7efa24     bbb
0x16d7efa28     ccc
0x16d7efa2c     ddd
0x16d7efa30     eee
0x16d7efa34     fff
```
先看熟悉的argv，argv是从bash传来的参数，在调用main()时作为实参压入栈中；栈向内存地址减小的方向生长，所以这些参数从右向左依次入栈。再来看看环境变量的内存地址，其值都比argv大，且按顺序依次增大；意味着环境变量在argv之前就入栈，并且在栈中环境变量和argv紧邻。

换句话说，就是下面这样：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/e450e4523abf47ca933041bbc3abbaf7_1720253884434.png?token=ANB4BCL3EX6FB7SSRES5LP3GRD67W)
也就是说将` int main (int argc, char *argv[])`可以理解为：`int main (int argc, char *argv[], char *envp[])`。即调用main()时实际还传递了环境变量数组。

### （一）环境变量怎么使用？
在C中通过函数`getenv()`就可以获得指定环境变量的值，比如：

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, const char* argv[]) {
    char* home = getenv("HOME");
    printf("Your home directory is %s.\n", home);
    return 0;
}
```
输出为：
> Your home directory is /Users/luliang.
>
不要求此函数为线程安全。到 getenv 的另一调用，还有到 POSIX 函数 setenv() 、 unsetenv() 及 putenv()的调用可能非法化先前调用所返回的指针，或修改从先前调用得到的字符串。(C++11 前)
只要无其他函数修改宿主环境，则此函数线程安全（从多个线程调用它不引入数据竞争）。尤其是若无同步地调用，则 POSIX 函数 setenv() 、 unsetenv() 及 putenv() 会引入数据竞争。(C++11 起)修改 getenv 所返回的字符串引起未定义行为。
### （二）环境变量从哪里来？
#### （1）继承
进程的环境变量继承自其父进程。父进程在创建子进程时，可以修改子进程的环境变量（修改、添加或删除），但一旦子进程创建完毕，子进程和父进程的环境变量便不再有任何联系，这也就意味着父进程失去了修改子进程相关信息的权利。

父进程：

```c
#include <stdio.h>
#include <unistd.h>
extern char** environ;
void show_env() {
    printf("environment variables:\n");
    int i = 0;
    while (environ[i]) {
        printf("%p\t%s\n", environ[i], environ[i]);
        i++;
    }
}
int main(int argc, const char* argv[]) {
    printf("parent process:\n");
    show_env();
    if (fork() == 0) {
        execl("./child", "child", NULL);
    }

    return 0;
}
```
子进程：

```c
#include <stdio.h>
extern char** environ;
void show_env() {
    printf("environment variables:\n");
    int i = 0;
    while (environ[i]) {
        printf("%p\t%s\n", environ[i], environ[i]);
        i++;
    }
}

int main(int argc, const char* argv[]) {
    printf("child process\n");
    show_env();
}
```
父进程首先打印出自己的环境变量，然后fork()一个克隆的新进程，再通过execl()执行新的程序child。

执行结果如下：

```c
parent process:
environment variables:
0x16f2efa39     USER=luliang
0x16f2efa46     MallocNanoZone=0
0x16f2efa57     __CFBundleIdentifier=com.microsoft.VSCode
0x16f2efa81     COMMAND_MODE=unix2003
0x16f2efa97     LOGNAME=luliang
 					...
0x16f2efd67     OLDPWD=/Users/luliang/CProjects/Test
0x16f2efd8c     HOMEBREW_PREFIX=/opt/homebrew
0x16f2efdaa     HOMEBREW_CELLAR=/opt/homebrew/Cellar
0x16f2efdcf     HOMEBREW_REPOSITORY=/opt/homebrew
0x16f2efdf1     MANPATH=/opt/homebrew/share/man::
0x16f2efe13     INFOPATH=/opt/homebrew/share/info:
0x16f2efe36     _=/Users/luliang/CProjects/Test/./father
child process
environment variables:
0x16d3bba36     USER=luliang
0x16d3bba43     MallocNanoZone=0
0x16d3bba54     __CFBundleIdentifier=com.microsoft.VSCode
0x16d3bba7e     COMMAND_MODE=unix2003
0x16d3bba94     LOGNAME=luliang
 					...
0x16d3bbd64     OLDPWD=/Users/luliang/CProjects/Test
0x16d3bbd89     HOMEBREW_PREFIX=/opt/homebrew
0x16d3bbda7     HOMEBREW_CELLAR=/opt/homebrew/Cellar
0x16d3bbdcc     HOMEBREW_REPOSITORY=/opt/homebrew
0x16d3bbdee     MANPATH=/opt/homebrew/share/man::
0x16d3bbe10     INFOPATH=/opt/homebrew/share/info:
0x16d3bbe33     _=/Users/luliang/CProjects/Test/./father
```
在看一个例子，修改父进程，在fork()出新的进程后，修改环境变量PATH，增加一个新的环境变量PENGUIN=BEAR。

```c
#include <stdio.h>
#include <unistd.h>

extern char** environ;

void show_env() {
    printf("environment variables:\n");
    int i = 0;
    while (environ[i]) {
        printf("%p\t%s\n", environ[i], environ[i]);
        i++;
    }
}
void setenv();
void putenv();
int main(int argc, const char* argv[]) {
    printf("parent process:\n");
    show_env();
    if (fork() == 0) {
        setenv("PATH", "wrong", 1);
        putenv("PENGUIN=BEAR");
        execl("./child", "child", NULL);
    }

    return 0;
}
```
输出为：

```c
parent process:
environment variables:
					...
0x16f4b7b80     SSH_AUTH_SOCK=/private/tmp/com.apple.launchd.2NSRW4JaeV/Listeners
0x16f4b7bc2     SHELL=/bin/zsh
0x16f4b7bd1     HOME=/Users/luliang
0x16f4b7be5     __CF_USER_TEXT_ENCODING=0x1F5:0x19:0x34
0x16f4b7c0d     TMPDIR=/var/folders/kh/2q4rgmw559s8p4zkm7df58340000gn/T/
0x16f4b7c46     XPC_SERVICE_NAME=0
					...
0x16f4b7e36     _=/Users/luliang/CProjects/Test/./father
child process
environment variables:
					...
0x16cfabb64     PATH=wrong
0x16cfabb6f     SSH_AUTH_SOCK=/private/tmp/com.apple.launchd.2NSRW4JaeV/Listeners
0x16cfabbb1     SHELL=/bin/zsh
0x16cfabbc0     HOME=/Users/luliang
0x16cfabbd4     __CF_USER_TEXT_ENCODING=0x1F5:0x19:0x34
0x16cfabbfc     TMPDIR=/var/folders/kh/2q4rgmw559s8p4zkm7df58340000gn/T/
0x16cfabc35     XPC_SERVICE_NAME=0
					...
0x16cfabe25     _=/Users/luliang/CProjects/Test/./father
0x16cfabe4e     PENGUIN=BEAR
```
可以看到子进程的环境变量PATH被修改为了wrong，并且多了一个环境变量`PENGUIN=BEAR`，这些环境变量都一起最先压入栈中。即父进程在创建子进程时，可以修改子进程的环境变量，子进程创建完毕调用main()时修改后的环境变量会首先压入栈中。

#### （2）从整体看
从上面的实验可以看出，一个进程创建另一个进程的同时，子进程会继承父进程的环境变量，同时父进程也会对子进程的环境变量做一些修改（增加、删除、更改）。同时，操作系统中所有的进程的父进程都是 init 进程的祖宗进程，这就自然地形成了一个环境变量树形结构。

在Linux中，进程继承得到的环境变量保存在/proc/<pid>/environ中（不包括进程运行中修改的环境变量），借着这个我们来简单分析下上述例子的环境变量继承关系。

首先重新罗列下父进程的环境变量：

```c
environment variables:
0x16f4b7a39     USER=luliang
0x16f4b7a46     MallocNanoZone=0
0x16f4b7a57     __CFBundleIdentifier=com.microsoft.VSCode
0x16f4b7a81     COMMAND_MODE=unix2003
0x16f4b7a97     LOGNAME=luliang
0x16f4b7aa7     PATH=/usr/local/opt/binutils/bin:/opt/homebrew/opt/binutils/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/Applications/VMware Fusion.app/Contents/Public:/Library/Apple/usr/bin
0x16f4b7b80     SSH_AUTH_SOCK=/private/tmp/com.apple.launchd.2NSRW4JaeV/Listeners
0x16f4b7bc2     SHELL=/bin/zsh
0x16f4b7bd1     HOME=/Users/luliang
0x16f4b7be5     __CF_USER_TEXT_ENCODING=0x1F5:0x19:0x34
0x16f4b7c0d     TMPDIR=/var/folders/kh/2q4rgmw559s8p4zkm7df58340000gn/T/
0x16f4b7c46     XPC_SERVICE_NAME=0
0x16f4b7c59     XPC_FLAGS=0x0
0x16f4b7c67     ORIGINAL_XDG_CURRENT_DESKTOP=undefined
0x16f4b7c8e     TERM_PROGRAM=vscode
0x16f4b7ca2     TERM_PROGRAM_VERSION=1.74.2
0x16f4b7cbe     LANG=zh_CN.UTF-8
0x16f4b7ccf     COLORTERM=truecolor
0x16f4b7ce3     VSCODE_INJECTION=1
0x16f4b7cf6     ZDOTDIR=/Users/luliang
0x16f4b7d0d     USER_ZDOTDIR=/Users/luliang
0x16f4b7d29     PWD=/Users/luliang/CProjects/Test
0x16f4b7d4b     TERM=xterm-256color
0x16f4b7d5f     SHLVL=1
0x16f4b7d67     OLDPWD=/Users/luliang/CProjects/Test
0x16f4b7d8c     HOMEBREW_PREFIX=/opt/homebrew
0x16f4b7daa     HOMEBREW_CELLAR=/opt/homebrew/Cellar
0x16f4b7dcf     HOMEBREW_REPOSITORY=/opt/homebrew
0x16f4b7df1     MANPATH=/opt/homebrew/share/man::
0x16f4b7e13     INFOPATH=/opt/homebrew/share/info:
0x16f4b7e36     _=/Users/luliang/CProjects/Test/./father
```
我们知道`father`进程是由`cpptools`创建的，我们看看它的环境变量。
首先找到它的pid，键入`ps -wwE -p 2519 `,输出为：

```c
/Users/luliang/.vscode/extensions/ms-vscode.cpptools-1.13.9-darwin-arm64/bin/cpptools ELECTRON_RUN_AS_NODE=1 USER=luliang MallocNanoZone=0
__CFBundleIdentifier=com.microsoft.VSCode COMMAND_MODE=unix2003 LOGNAME=luliang
PATH=/usr/local/opt/binutils/bin:/opt/homebrew/opt/binutils/bin:/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/Applications/VMware
Fusion.app/Contents/Public:/Library/Apple/usr/bin
SSH_AUTH_SOCK=/private/tmp/com.apple.launchd.2NSRW4JaeV/Listeners SHELL=/bin/zsh HOME=/Users/luliang
__CF_USER_TEXT_ENCODING=0x1F5:0x19:0x34
TMPDIR=/var/folders/kh/2q4rgmw559s8p4zkm7df58340000gn/T/ XPC_SERVICE_NAME=application.com.microsoft.VSCode.59852660.59852666
XPC_FLAGS=0x0 ORIGINAL_XDG_CURRENT_DESKTOP=undefined VSCODE_CWD=/ VSCODE_NLS_CONFIG={"locale":"zh-cn","availableLanguages":{"*":"zh-cn"},"_languagePackId":"f45d28db2b892ef5d3a7efebc519f640.zh-cn","_translationsConfigFile":"/Users/luliang/Library/Application Support/Code/clp/f45d28db2b892ef5d3a7efebc519f640.zh-cn/tcf.json","_cacheRoot":"/Users/luliang/Library/Application Support/Code/clp/f45d28db2b892ef5d3a7efebc519f640.zh-cn","_resolvedLanguagePackCoreLocation":"/Users/luliang/Library/Application Support/Code/clp/f45d28db2b892ef5d3a7efebc519f640.zh-cn/e8a3071ea4344d9d48ef8a4df2c097372b0c5161","_corruptedFile":"/Users/luliang/Library/Application Support/Code/clp/f45d28db2b892ef5d3a7efebc519f640.zh-cn/corrupted.info","_languagePackSupport":true}
VSCODE_CODE_CACHE_PATH=/Users/luliang/Library/Application
Support/Code/CachedData/e8a3071ea4344d9d48ef8a4df2c097372b0c5161
VSCODE_IPC_HOOK=/Users/luliang/Library/Application Support/Code/1.74.2-main.sock
VSCODE_PID=2461 SHLVL=0 PWD=/ OLDPWD=/ HOMEBREW_PREFIX=/opt/homebrew
HOMEBREW_CELLAR=/opt/homebrew/Cellar HOMEBREW_REPOSITORY=/opt/homebrew
MANPATH=/opt/homebrew/share/man:: INFOPATH=/opt/homebrew/share/info:
_=/Applications/Visual Studio Code.app/Contents/MacOS/Electron
VSCODE_AMD_ENTRYPOINT=vs/workbench/api/node/extensionHostProcess
VSCODE_HANDLES_UNCAUGHT_ERRORS=true APPLICATION_INSIGHTS_NO_DIAGNOSTIC_CHANNEL=1


```
可以看到`cpptools`在创建我们的程序的进程时，又向环境变量中添加了新的环境变量。

> 通过上述例子或许还能解开一个疑惑：Linux中众多的设置环境变量的文件（如`/etc/environment`，`~/.bashrc`），我到底该修改哪个呢？
可以发现，这些设置环境变量的文件实际是不同的进程在创建时固定读取的，如systemd会读取`/etc/default/locale`，sshd会读取`/etc/environment`，bash会读取`~/.bashrc`；而进程间的父子关系又决定了这些环境变量的加载时机和作用范围，如systemd作为所有进程的父进程，修改`/etc/default/locale`会作用到所有进程中，bash仅为当前用户提供交互，所以修改`/.bashrc`只会对当前的bash有效。当将一个个配置文件对应到相应的进程，理顺父子关系后，环境变量的配置问题便很容易解决了。

全文完，感谢你的阅读。

**参考：**
http://freewind.in/posts/2781-how-to-see-the-env-vars-system-passed-to-a-process/
https://blog.csdn.net/cnwyt/article/details/105073749
Linux---[fork函数和exec函数](https://blog.csdn.net/weixin_45676049/article/details/120351219?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-1-120351219-blog-126602077.pc_relevant_3mothn_strategy_recovery&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-1-120351219-blog-126602077.pc_relevant_3mothn_strategy_recovery&utm_relevant_index=2)
https://blog.csdn.net/nihaoma95278/article/details/126602077
https://blog.csdn.net/sinat_38604998/article/details/101078479
https://www.cnblogs.com/qingergege/p/6495475.html
https://blog.csdn.net/Mint6/article/details/124156340
https://zh.m.wikipedia.org/zh-hans/MinGW
https://www.polarxiong.com/


