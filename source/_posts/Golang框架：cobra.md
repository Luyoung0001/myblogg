---
title: Golang框架：cobra
date: 2023-06-06 23:27:59
tags:
	- golang
	- 后端
	- 前端
	- 网络
categories: Golang Golang_框架
---

<!--more-->

## 〇、前言
[cobra](https://github.com/spf13/cobra) 库是 golang 的一个开源第三方库，能够快速便捷的建立命令行应用程序。
比如：

>  ls -a
>  git add .
>  gcc a.c -o app
>  ...

这些都是命令，二前面的ls、git、gcc等都是我们写的程序。如果不使用 cobra 框架进行编写这些命令，那么程序写起来是相当费劲的。
## 一、基本概念
cobra由三部分构成：commands，arguments 和 flags：
- commands：表示要执行的动作。每一个 command 表示应用程序的一个动作。每个命令可以包含子命令。

- arguments：给动作传入的参数。

- flags：表示动作的行为。可以设置执行动作的行为。flags 包括两种：对某个命令生效和对所有命令生效。

比如：`./app echo time helloword  -t 4`中，`app` 就是程序，`echo` 就是子命令，`time` 是 `echo` 的子命令，`helloword` 是传入的参数，`-t` 是则是 `flags`。

## 二、一个例子
这个例子的目录结构如下：

> ▾ pro_test/
    ___▾ cmd/
    ______echo.go
    ______print.go
    ______times.go
    ______root.go
    main.go

### （一）root.go

```go
package cmd

import "github.com/spf13/cobra"

var rootCmd = &cobra.Command{
	Use: "app [command]",
}

func Execute() {
	rootCmd.Execute()
}

```
### （二）print.go

```go
package cmd

import (
	"fmt"
	"github.com/spf13/cobra"
	"strings"
)

var cmdPrint = &cobra.Command{
	Use:   "Print [string to print]",
	Short: "Print anything to the screen",
	Long: `print is for printing anything back to the screen.
For many years people have printed back to the screen.`,
	Args: cobra.MinimumNArgs(1),
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("Print: " + strings.Join(args, " "))
	},
}

func init() {
	rootCmd.AddCommand(cmdPrint)
}

```
### （三）echo.go

```go
package cmd

import (
	"fmt"
	"github.com/spf13/cobra"
	"strings"
)

var cmdEcho = &cobra.Command{
	Use:   "echo [string to echo]",
	Short: "Echo anything to the screen",
	Long: `echo is for echoing anything back.
Echo works a lot like print, except it has a child command.`,
	Args: cobra.MinimumNArgs(1),
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("Print: " + strings.Join(args, " "))
	},
}

func init() {
	rootCmd.AddCommand(cmdEcho)
}

```
### （四）times.go

```go
package cmd

import (
	"fmt"
	"github.com/spf13/cobra"
	"strings"
)

var echoTimes int
var cmdTimes = &cobra.Command{
	Use:   "time [# times] [string to echo]",
	Short: "Echo anything to the screen more times",
	Long: `echo things multiple times back to the user y providing
		a count and a string.`,
	Args: cobra.MinimumNArgs(1),
	Run: func(cmd *cobra.Command, args []string) {
		for i := 0; i < echoTimes; i++ {
			fmt.Println("Echo: " + strings.Join(args, " "))
		}
	},
}

func init() {
	cmdTimes.Flags().IntVarP(&echoTimes, "times", "t", 1, "times to echo the input")
	cmdEcho.AddCommand(cmdTimes)
}

```

### （五）main.go

main.go的作用就是初始化：
```go
package main

import "projects_20/pro_1_example/cmd"

func main() {
	cmd.Execute()
}

```
### （六）编译以及使用
键入：`go build -o app  `，就可以得到一个 名为app 的命令行应用程序。

cobra帮我们生成了很棒的 help 信息：

```bash
(base) ***@shenjian pro_1_example % ./app -h
Usage:
  app [command]

Available Commands:
  Print       Print anything to the screen
  completion  Generate the autocompletion script for the specified shell
  echo        Echo anything to the screen
  help        Help about any command

Flags:
  -h, --help   help for app

Use "app [command] --help" for more information about a command.

```

可以看到Available Commands有四个命令，试试 echo 这个命令：

```bash
(base) ***@shenjian pro_1_example % ./app echo -h
echo is for echoing anything back.
Echo works a lot like print, except it has a child command.

Usage:
  app echo [string to echo] [flags]
  app echo [command]

Available Commands:
  time        Echo anything to the screen more times

Flags:
  -h, --help   help for echo

Use "app echo [command] --help" for more information about a command.

```
可以看到Usage 选项有两个方法，而且里面还有一个子命令 time。看看 time 是怎么使用的：

```bash
(base) luliang@shenjian pro_1_example % ./app echo time -h
echo things multiple times back to the user y providing
                a count and a string.

Usage:
  app echo time [# times] [string to echo] [flags]

Flags:
  -h, --help        help for time
  -t, --times int   times to echo the input (default 1)

```
这个 Usage 就是用法，试试：

```bash
(base) ***@shenjian pro_1_example % ./app echo time helloworld -t 6
Echo: helloworld
Echo: helloworld
Echo: helloworld
Echo: helloworld
Echo: helloworld
Echo: helloworld

```
可以看到命令正常使用，太棒了！

## 三、使用标志
标志提供修饰符以控制命令的操作方式。由于标志是在不同位置定义和使用的，我们需要在外部定义一个具有正确作用域的变量，以分配要使用的标志。

全局标志：

```go
rootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "verbose output")
```
局部标志：

```go
localCmd.Flags().StringVarP(&Source, "source", "s", "", "Source directory to read from")
```

*全文完，感谢阅读。*

