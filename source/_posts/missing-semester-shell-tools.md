---
title: "Missing Semester 学习笔记（二）：Shell 工具与脚本"
date: 2023-12-23 14:28:18
tags:
  - "Missing Semester"
  - "shell"
  - "tools"
categories:
  - "Tooling"
---

<!--more-->

## Shell tools
### shell 脚本

Bash中的字符串通过 `'` 和 `"` 分隔符来定义，但是它们的含义并不相同。以 `'` 定义的字符串为原义字符串，其中的变量不会被转义，而 `"` 定义的字符串会将变量值进行替换。

```bash
foo=bar
echo "$foo"
# 打印 bar
echo '$foo'
# 打印 $foo
```

**bash** 同样支持函数：

```bash
mcd () {
    mkdir -p "$1"
    cd "$1"
}
```

输入：

```bash
mcd hello
```
由于 **hello** 是函数的第一个参数，因此 `mcd()` 会创建一个名为 **hello** 的目录，然后就会进入：`~/hello`。

对于 bash 参数，有以下含义：

- $0 - 脚本名
- $1 到 $9 - 脚本的参数。 $1 是第一个参数，依此类推。
- $@ - 所有参数
- $# - 参数个数
- $? - 前一个命令的返回值
- $$ - 当前脚本的进程识别码
- !! - 完整的上一条命令，包括参数。常见应用：当你因为权限不足执行命令失败时，可以使用 sudo !!再尝试一次
- $_ - 上一条命令的最后一个参数。如果你正在使用的是交互式 shell，你可以通过按下 Esc 之后键入 . 来获取这个值

命令通常使用 **STDOUT**来返回输出值，使用**STDERR** 来返回错误及错误码，便于脚本以更加友好的方式报告错误。 返回码或退出状态是脚本/命令之间交流执行状态的方式。返回值0表示正常执行，其他所有非0的返回值都表示有错误发生。

退出码可以搭配 **&&**（与操作符）和 **||**（或操作符）使用，用来进行条件判断，决定是否执行其他程序。它们都属于短路运算符（short-circuiting） 同一行的多个命令可以用 `; `分隔。程序 true 的返回码永远是0，false 的返回码永远是1。让我们看几个例子：

```bash
false || echo "Oops, fail"
# Oops, fail

true || echo "Will not be printed"
#

true && echo "Things went well"
# Things went well

false && echo "Will not be printed"
#

false ; echo "This will always run"
# This will always run
```


另一个常见的模式是以变量的形式获取一个命令的输出，这可以通过 *命令替换（command substitution）* 实现。

当您通过 `$(CMD)` 这样的方式来执行 **CMD** 这个命令时，它的输出结果会替换掉 `$(CMD) `。例如，如果执行 `for file in $(ls)` ，**shell**首先将调用 **ls** ，然后遍历得到的这些返回值。再比如：

```bash
echo "date is $(date)"
date is Sat 23 Dec 2023 11:24:05 CST
```

还有一个冷门的类似特性是 **进程替换（process substitution）**， `<(CMD) `会执行 **CMD** 并将结果输出到一个临时文件中，并将 `<(CMD)` 替换成临时文件名。这在我们希望返回值通过文件而不是STDIN传递时很有用。例如， `diff <(ls foo) <(ls bar)` 会显示文件夹 **foo** 和 **bar** 中文件的区别。比如：

```bash
diff <(ls -a) <(ls)
< .
< ..
< .CFUserTextEncoding
< .DS_Store
< .IdentityService
< .ServiceHub
< .Trash
```
分析一下这个脚本：
```powershell
#!/bin/bash

echo "Starting program at $(date)" # date会被替换成日期和时间

echo "Running program $0 with $# arguments with pid $$"

for file in "$@"; do
    grep foobar "$file" > /dev/null 2> /dev/null
    # 如果模式没有找到，则grep退出状态为 1
    # 我们将标准输出流和标准错误流重定向到Null，因为我们并不关心这些信息
    if [[ $? -ne 0 ]]; then
        echo "File $file does not have any foobar, adding one"
        echo "# foobar" >> "$file"
    fi
done
```

首先会打印出当前的日期时间，然后打印出当前进程的名字、参数个数和进程 ID。接着遍历所有的参数，这些参数就是脚本输入的文件名等，在这些文件中搜索 **foobar**，并将错误结果、结果全部重定向到 **null**。如果上一个命令的返回值 不为 0，这意味着出错，就在那个文件中追加 `# foobar` 。

接着了解通配符。

```bash
mkdir {hello1,hello2}
ls hello?
hello1:

hello2:

ls hello*
hello.txt

hello1:

hello2:
```

这就是通配符 `?` 和 `*` 的区别。

比如：

```bash
mkdir foo bar
touch {foo,bar}/{a..h}
touch foo/x bar/y

ls bar
a b c d e f g h y
ls foo
a b c d e f g h x

diff <(ls foo) <(ls bar)
< x
---
> y
```

**python** 是一个强大的脚本语言，因此用 **python** 写脚本，可以轻松实现很强大的功能，使得自动化变得轻而易举。以下是一个简单的 **python** 脚本：

```powershell
#!/usr/local/bin/python
import sys
for arg in reversed(sys.argv[1:]):
    print(arg)
```
这个脚本能执行的前提是：

```bash
ls /usr/local/bin | grep python
```
这个文件必须存在，不然就会：

```bash
./python_script
bash: ./python_script: /usr/local/bin/python: bad interpreter: No such file or directory
```
解决这个问题，要么修改脚本指定一个存在的解释器，要么可以让这个脚本更具通用性：

```powershell
#!/usr/bin/env python
import sys
for arg in reversed(sys.argv[1:]):
    print(arg)
```
在 **shebang** 行中使用 **env** 命令是一种好的实践，它会利用环境变量中的程序来解析该脚本，这样就提高脚本的可移植性。**env** 会利用我们第一节讲座中介绍过的 PATH 环境变量来进行定位。

当然前提是 **env** 命令存在：

```bash
ls /usr/bin | grep env
dbus-update-activation-environment
env
envsubst
grub-editenv
openvt
printenv
```

然后就可以运行了：

```bash
./python_script hello hello1 hello2
hello2
hello1
hello
```
### shell 工具

#### find

当使用`find`命令时，您可以使用不同的选项和参数来搜索文件或目录，并根据各种标准进行过滤。以下是`find`命令的一些常见用法和示例：

##### 基本用法：
1. **按文件名搜索：**
   ```bash
   find /path/to/search -name "filename.txt"
   ```
   在指定路径 `/path/to/search` 下按照文件名查找名为 `filename.txt` 的文件。

2. **按文件类型搜索：**
   ```bash
   find /path/to/search -type f
   ```
   在指定路径 `/path/to/search` 下查找普通文件。

3. **按目录名称搜索：**
   ```bash
   find /path/to/search -type d -name "dirname"
   ```
   在指定路径 `/path/to/search` 下查找名为 `dirname` 的目录。

##### 根据时间搜索：
1. **按最后修改时间搜索：**
   ```bash
   find /path/to/search -mtime -7
   ```
   在指定路径 `/path/to/search` 下搜索最近7天内修改过的文件。

2. **按最后访问时间搜索：**
   ```bash
   find /path/to/search -atime +30
   ```
   在指定路径 `/path/to/search` 下搜索超过30天未访问过的文件。

##### 结合条件搜索：
1. **结合AND条件：**
   ```bash
   find /path/to/search -type f -name "filename.txt"
   ```
   在指定路径 `/path/to/search` 下查找名为 `filename.txt` 的普通文件。

2. **结合OR条件：**
   ```bash
   find /path/to/search \( -name "*.txt" -o -name "*.pdf" \)
   ```
   在指定路径 `/path/to/search` 下查找扩展名为 `.txt` 或 `.pdf` 的文件。

##### 进行操作：
1. **删除匹配的文件：**
   ```bash
   find /path/to/search -type f -name "filename.txt" -delete
   ```
   删除指定路径 `/path/to/search` 下名为 `filename.txt` 的文件。

2. **执行命令操作：**
   ```bash
   find /path/to/search -type f -exec chmod 644 {} \;
   ```
   更改指定路径 `/path/to/search` 下所有普通文件的权限为 `644`。

这些示例展示了`find`命令的一些常见用法，可以根据需要进行修改和组合。`find`命令非常灵活，可以根据文件名、类型、时间等多种条件进行搜索和操作。

#### grep

`grep`命令是一个强大的文本搜索工具，可以在文件中查找特定模式的文本行。以下是`grep`命令的一些常见用法和示例：

##### 基本用法：
1. **查找特定字符串：**
   ```bash
   grep "pattern" filename.txt
   ```
   在 `filename.txt` 文件中查找包含指定模式 `pattern` 的文本行。

2. **忽略大小写查找：**
   ```bash
   grep -i "pattern" filename.txt
   ```
   在查找时忽略大小写。

3. **显示匹配行数：**
   ```bash
   grep -c "pattern" filename.txt
   ```
   显示匹配 `pattern` 的行数。

##### 使用正则表达式：
1. **查找以特定单词开头的行：**
   ```bash
   grep "^word" filename.txt
   ```
   在 `filename.txt` 文件中查找以 `word` 开头的文本行。

2. **查找以特定单词结尾的行：**
   ```bash
   grep "word$" filename.txt
   ```
   在 `filename.txt` 文件中查找以 `word` 结尾的文本行。

3. **查找匹配特定模式的行：**
   ```bash
   grep "[0-9]\{3\}" filename.txt
   ```
   在 `filename.txt` 文件中查找包含三个连续数字的文本行。

##### 结合其他命令：
1. **在某个目录下的所有文件中查找：**
   ```bash
   grep -r "pattern" /path/to/directory
   ```
   在指定目录 `/path/to/directory` 及其所有子目录中查找包含 `pattern` 的文本行。

2. **通过管道结合其他命令：**
   ```bash
   cat filename.txt | grep "pattern"
   ```
   使用 `cat` 命令读取文件内容，然后使用 `grep` 过滤包含 `pattern` 的行。

##### 显示上下文信息：
1. **显示匹配行上下文：**
   ```bash
   grep -C 2 "pattern" filename.txt
   ```
   在匹配的行上下各显示2行文本。

2. **仅显示匹配行号：**
   ```bash
   grep -n "pattern" filename.txt
   ```
   在每行输出匹配行的行号。

这些示例展示了`grep`命令的一些常见用法，可以根据需要进行修改和组合。`grep`命令支持多种搜索模式和选项，使其成为处理文本数据时的强大工具。


## 课后练习

### 1、阅读 man ls ，然后使用ls 命令进行如下操作：
- 所有文件（包括隐藏文件）：

```bash
ls -a
.
..
...
```

- 文件打印以人类可以理解的格式输出 (例如，使用454M 而不是 454279954)：

```bash
ls -l --block-size=human-readable
total 119M
drwxr-xr-x 24 root root 4.0K Sep  8 21:52 gdb-13.2
-rw-r--r--  1 root root  23M May 27  2023 gdb-13.2.tar.xz
...
```
- 文件以最近访问顺序排序：

```bash
ls -lth
total 119M
-rwxr-xr-x  1 root root   83 Dec 23 11:51 python_script
-rwxr-xr-x  1 root root   32 Dec 22 20:36 lab
...
```

- 以彩色文本显示输出结果：

```bash
ls -lG
total 120980
drwxr-xr-x 24 root      4096 Sep  8 21:52 gdb-13.2
-rw-r--r--  1 root  23664644 May 27  2023 gdb-13.2.tar.xz
...
```
### 2、编写两个bash函数 marco 和 polo 执行下面的操作。 每当你执行 marco 时，当前的工作目录应当以某种形式保存，当执行 polo 时，无论现在处在什么目录下，都应当 cd 回到当时执行 marco 的目录。 为了方便debug，你可以把代码写在单独的文件 marco.sh 中，并通过 source marco.sh命令，（重新）加载函数


将以下代码保存在名为 `marco.sh` 的文件中：

```bash
#!/bin/bash

marco() {
    export MARCO_DIR=$(pwd)
}

polo() {
    cd "$MARCO_DIR" || return
}
```

这个脚本定义了两个函数：`marco` 和 `polo`。

- `marco` 函数将当前工作目录保存到名为 `MARCO_DIR` 的环境变量中。
- `polo` 函数将切换到 `MARCO_DIR` 所保存的目录。

要使用这些函数，必须通过运行 `source marco.sh` 来加载它们。这将使得函数在当前的 **shell** 环境中可用。**也就是说，这里不能通过 子 shell 去运行这个脚本，因为运行完了 子 shell 就结束了，当前 shell 不会存在这两个函数。**

然后，可以在任何目录中执行 `marco` 命令来保存当前目录，并在任何位置执行 `polo` 命令来返回到之前保存的目录。需要注意的是，`marco` 和 `polo` 函数的效果只在同一个 **shell** 会话中有效。如果打开了一个新的终端窗口或启动了一个新的 **shell**，需要再次运行 `source marco.sh` 来加载函数。

### 3、假设您有一个命令，它很少出错。因此为了在出错时能够对其进行调试，需要花费大量的时间重现错误并捕获输出。 编写一段bash脚本，运行如下的脚本直到它出错，将它的标准输出和标准错误流记录到文件，并在最后输出所有内容。 加分项：报告脚本在失败前共运行了多少次

```bash
 #!/usr/bin/env bash

 n=$(( RANDOM % 100 ))

 if [[ n -eq 42 ]]; then
    echo "Something went wrong"
    >&2 echo "The error was using magic numbers"
    exit 1
 fi

 echo "Everything went according to plan"
```

这个脚本可以实现要求：

```bash
#!/usr/bin/env bash

count=0
while true; do
    output=$(./hardly_wrong.sh)
    exit_code=$?

    echo "$output" >> output.log

    if [[ $exit_code -ne 0 ]]; then
        echo "Script failed after $count runs wright." >> output.log
        break
    fi
    count=$((count+1))
done

cat output.log
~
```
运行结果如下：
```bash
./test
The error was using magic numbers
Everything went according to plan
...
Something went wrong
Script failed after 11 runs wright.
```

这里为什么会打印 `The error was using magic numbers`？这实际上是 `./hardly_wrong.sh` 打印出来的，它出错的时候，我们并没有将它重定向到标准输出，而 **output** 接收的是标准输出。

总这里可以看出，`output=$(./hardly_wrong.sh)`只会将输出赋值给 **output**，至于标准错误 `&2`，它会直接打印到控制台上，尽管它会打印，这这不意味着它会赋值给 **output**，这点需要格外注意：

```bash
cat output.log
Everything went according to plan
...
Something went wrong
Script failed after 11 runs wright.
```

### 4、本节课我们讲解的 find 命令中的 -exec 参数非常强大，它可以对我们查找的文件进行操作

> 但是，如果我们要对所有文件进行操作呢？例如创建一个zip压缩文件？我们已经知道，命令行可以从参数或标准输入接受输入。在用管道连接命令时，我们将标准输出和标准输入连接起来，但是有些命令，例如tar 则需要从参数接受输入。这里我们可以使用xargs 命令，它可以使用标准输入中的内容作为参数。 例如 ls | xargs rm 会删除当前目录中的所有文件。
>
> 您的任务是编写一个命令，它可以递归地查找文件夹中所有的HTML文件，并将它们压缩成zip文件。注意，即使文件名中包含空格，您的命令也应该能够正确执行（提示：查看xargs的参数-d，译注：MacOS 上的 xargs没有-d，查看这个issue）
>
> 如果您使用的是 MacOS，请注意默认的 BSD find 与 GNU coreutils中的是不一样的。你可以为find添加-print0选项，并为xargs添加-0选项。作为 Mac 用户，您需要注意 mac系统自带的命令行工具和 GNU 中对应的工具是有区别的；如果你想使用 GNU 版本的工具，也可以使用 brew 来安装。

当需要递归地查找文件夹中的HTML文件并将它们压缩成zip文件时，可以使用以下命令：

```bash
find . -type f -name "*.html" -exec zip -j output.zip {} +
```

这条命令中：

- `.` 是要搜索的目标文件夹的路径。
- `-type f` 表示查找文件而不是目录。
- `-name "*.html"` 指定了要查找的文件扩展名为HTML。
- `-exec zip -j output.zip {} +` 使用 `zip` 命令将找到的HTML文件压缩到名为 `output.zip` 的文件中。`-j` 选项告诉 `zip` 命令将所有文件放在同一层级，而不是创建子文件夹。

这个命令会在指定的目标文件夹中递归地查找所有的**HTML**文件，并将它们压缩成一个名为 `output.zip` 的**zip**文件。

### 5、（进阶）编写一个命令或脚本递归的查找文件夹中最近使用的文件。更通用的做法，你可以按照最近的使用时间列出文件吗？

可以使用 `find` 命令结合 `-printf` 来列出最近使用的文件。下面是一个示例：

```bash
find /path/to/directory -type f -printf '%T@ %p\n' | sort -n | tail -n 1
```

这个命令的步骤是：

- `find /path/to/directory -type f -printf '%T@ %p\n'`：使用 `find` 命令搜索目标文件夹中的所有文件，`-type f` 表示只查找文件。`-printf '%T@ %p\n'` 会以特定格式打印文件的修改时间（秒为单位自UTC 1970年1月1日00:00:00以来的时间）和文件路径。
- `sort -n`：使用 `sort` 命令对时间戳进行数字排序。
- `tail -n 1`：获取排序后的结果中的最后一行，即最近使用的文件。

这个命令将列出目标文件夹中最近使用的文件。如果要列出最近修改的文件，可以将 `%T@` 替换为 `%C@`。

```bash
find ~ -type f -printf '%T@ %p\n' | sort -n | tail -n 1
1703312042.6682969990 /root/pacman/nohup.out
```


## 总结（大模型）
`bash`中的字符串使用`'`和`"`定义，每种方式有不同的含义。`'`定义的字符串为原义字符串，不进行变量替换，而`"`定义的字符串会替换变量值。此外，`bash`中支持函数，例如`mcd()`函数可以创建目录并进入其中。命令行参数包括`$0`到`$9`，`$@`表示所有参数，`$#`表示参数个数，`$$`表示当前脚本的进程识别码，`$?`表示前一个命令的返回值等。退出状态码是脚本和命令间交流执行状态的方式，非零值表示有错误发生。`&&`和`||`是短路运算符，`true`返回0，`false`返回1。命令替换使用`$(CMD)`，进程替换使用`<(CMD)`。

在课后练习中，`find`命令和`grep`命令展示了各种用法。针对问题，`find`可以用于文件搜索和操作，而`grep`用于文本搜索。此外，提到了`ls`命令和`xargs`命令的使用。

示例脚本涉及不同场景，包括错误处理和记录日志，以及利用`marco`和`polo`函数保存和返回目录。同时，针对最近使用的文件和文件操作，给出了`find`和`sort`命令的使用方法。

这些内容覆盖了`bash`中的常见命令和技术，可以帮助进行文件操作、搜索和脚本编写。




































