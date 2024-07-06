---
title: Go Web下gin框架的模板渲染
date: 2023-05-23 22:16:27
tags: 前端 golang gin
categories: Golang 前端
---

<!--more-->

## 〇、前言
Gin框架是一个用于构建Web应用程序的轻量级Web框架，使用Go语言开发。它具有高性能、低内存占用和快速路由匹配的特点，旨在提供简单、快速的方式来开发可扩展的Web应用程序。
Gin框架的设计目标是保持简单和易于使用，同时提供足够的灵活性和扩展性，使开发人员能够根据项目的需求进行定制。它提供了许多有用的功能，如中间件支持、路由组、请求参数解析、模板渲染等，使开发人员能够快速构建高效的Web应用程序。
以下是Gin框架的一些主要特性：

- 快速的性能：Gin框架采用了高性能的路由引擎，使请求路由匹配变得非常快速。

- 中间件支持：Gin框架支持中间件机制，允许你在请求处理过程中添加自定义的中间件，用于处理认证、日志记录、错误处理等功能。

- 路由组：Gin框架允许将路由按照逻辑组织成路由组，使代码结构更清晰，并且可以为不同的路由组添加不同的中间件。

- 请求参数解析：Gin框架提供了方便的方法来解析请求中的参数，包括查询字符串参数、表单参数、JSON参数等。

- 模板渲染：虽然Gin框架本身不提供模板引擎，但它与多种模板引擎库（如html/template、pongo2等）集成，使你能够方便地进行模板渲染。

- 错误处理：Gin框架提供了内置的错误处理机制，可以捕获和处理应用程序中的错误，并返回适当的错误响应。

总体而言，Gin框架是一个简单、轻量级但功能强大的Web框架，非常适合构建高性能、可扩展的Web应用程序。它在Go语言社区中得到广泛的认可，并被许多开发人员用于构建各种类型的Web应用程序。

## 一、html/template
`html/template`是Go语言标准库中的一个包，用于生成和渲染HTML模板。它提供了一种安全且灵活的方式来生成HTML输出，支持模板继承、变量替换、条件语句、循环结构等功能。

`html/template`包的主要目标是防止常见的Web安全漏洞，如跨站脚本攻击（XSS）。它通过自动进行HTML转义和编码来确保生成的HTML是安全的，并防止恶意用户注入恶意代码。

使用`html/template`包可以将动态数据与静态HTML模板分离，使代码更易于维护和重用。你可以定义模板文件，然后将数据传递给模板进行渲染，最后生成最终的HTML输出。

 ### （一）初次渲染
 
先创建一个名为 `hello.tmpl`的文件：
```go
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <title>Hello</title>
</head>
<body>
<p>hello, {{.}}</p>
</body>
</html>
```
这里的`{{.}}`中的`.`就代表了我们要填充的东东，接着创建我们的`main.go`函数：

```go
package main

import (
	"fmt"
	"html/template"
	"net/http"
)

func sayHello(w http.ResponseWriter, r *http.Request) {
	// 请勿刻舟求剑,用绝对地址
	t, err := template.ParseFiles("/Users/luliang/GoLand/gin_practice/chap1/hello.tmpl")
	if err != nil {
		fmt.Println("http server failed:%V", err)
		return
	}
	// 渲染模板
	err = t.Execute(w, "小王子!")
	if err != nil {
		fmt.Println("http server failed:%V", err)
		return
	}
}
func main() {
	http.HandleFunc("/hello", sayHello)
	err := http.ListenAndServe(":9000", nil)
	if err != nil {
		fmt.Println("http server failed:%V", err)
		return
	}
}

```

使用`html/template`进行HTML模板渲染的一般步骤如下：
- 定义模板：
首先，你需要定义HTML模板。可以在代码中直接定义模板字符串，也可以将模板保存在独立的文件中。比如我们创建了模板：`hello.tmpl`；

- 创建模板对象：
使用template.New()函数创建一个模板对象。你可以选择为模板对象指定一个名称，以便在渲染过程中引用它。

- 解析模板：
使用模板对象的Parse()或ParseFiles()方法解析模板内容。如果模板内容保存在单独的文件中，可以使用ParseFiles()方法解析文件内容并关联到模板对象。

- 渲染模板：
创建一个用于存储模板数据的数据结构，并将数据传递给模板对象的Execute()方法。该方法将渲染模板并将结果写入指定的输出位置（如os.Stdout或http.ResponseWriter）。

在浏览器中输入：`127.0.0.0:9000/hello`就可以看到结果：**hello, 小王子!**

### （二）传入其它数据进行渲染
上次渲染，我们用的是这一句：`err = t.Execute(w, "小王子!")`，可以看到我们传入了一个字符串而已。这次我们打算传点稍微不同的其它数据。继续编写模板`test.tmpl`:

```go
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <title>Hello</title>
</head>
<body>
<p>姓名: {{.Name}}</p>
<p>年龄: {{.Age}}</p>
<p>性别: {{.Gander}}</p>
</body>
</html>
```
可以看到，我们的模板稍微复杂起来了！继续编写 `main.go`：

```go
package main

import (
	"fmt"
	"html/template"
	"net/http"
)

type User struct {
	Name   string
	Gander string // 首字母是否大小写,作为是否对外暴露的标识
	Age    int
}

func sayHello(w http.ResponseWriter, r *http.Request) {
	// 定义模板
	u1 := User{
		Name:   "小王子",
		Gander: "男",
		Age:    19,
	}
	// 解析模板
	t, err := template.ParseFiles("/Users/luliang/GoLand/gin_practice/chap2/test.tmpl")
	if err != nil {
		fmt.Println("ParseFiles failed:%V", err)
		return
	}
	err = t.Execute(w, u1)
	if err != nil {
		return
	}

}
func main() {
	http.HandleFunc("/hello", sayHello)
	err := http.ListenAndServe(":9000", nil)
	if err != nil {
		fmt.Println("http server failed:%V", err)
		return
	}

}

```
可以看到我们在里面定义了一个结构类型 User，传入了一个 User 对象 u1：`err = t.Execute(w, u1)`

```go
type User struct {
	Name   string
	Gander string // 首字母是否大小写,作为是否对外暴露的标识
	Age    int
}
```
赶紧看看运行结果，可以看到结果符合预期：

> 姓名: 小王子
年龄: 19
性别: 男


### （三）定义函数参与渲染
定义我们的模板：`f.tmpl`:

```go
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <title>hello</title>
</head>
<body>
{{ kua . }}
</body>
</html>
```
可以看到，我们的模板里面有一个函数名字叫做 `kua`，没错，这就是我们要的函数。

编写`main.go`:

```go
package main
import (
	"fmt"
	"html/template"
	"net/http"
)
func f(w http.ResponseWriter, r *http.Request) {
	// 定义模板
	k := func(name string) (string, error) {
		return name + "太棒了!", nil
	}
	t := template.New("f.tmpl")
	t.Funcs(template.FuncMap{
		"kua": k,
	})
	// 解析模板
	_, err := t.ParseFiles("/Users/luliang/GoLand/gin_practice/chap3/f.tmpl")
	if err != nil {
		return
	}
	// 渲染模板
	err = t.Execute(w, "小王子")
	if err != nil {
		return
	}
}
func main() {
	http.HandleFunc("/hello", f)
	err := http.ListenAndServe(":9002", nil)
	if err != nil {
		fmt.Println("http server failed:%V", err)
		return
	}

}

```
可以看到，我用了一个关键语句将函数kua 关联到了模板：

```go
	t.Funcs(template.FuncMap{
		"kua": k,
	})
```
点击运行，可以看到结果：**小王子太棒了!**

### （四）c.HTML() 渲染

在Gin框架中，可以使用c.HTML()方法来进行HTML模板渲染。该方法接收HTTP状态码、模板名称和渲染所需的数据作为参数。


编写模板`index.tmpl`:

```go
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>posts/index</title>
</head>
<body>
{{.title}}
</body>
</html>
```
可以看到我们只需渲染传入对象的 title 值，编写 `main.go`:

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	r := gin.Default()
	r.LoadHTMLFiles("/Users/luliang/GoLand/gin_practice/chap4/index.tmpl")
	r.GET("/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.tmpl", gin.H{
			"title": "你好,前端真是太有意思了！",
		})
	})
	r.Run(":9091")
}

```
首先，创建一个 gin.default()路由对象，然后给该对象载入已经写好的模板文件，之后就可以用 GET 函数进行请求了。
先看看 GET 是什么东西：

```go
// GET is a shortcut for router.Handle("GET", path, handlers).
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodGet, relativePath, handlers)
}
```
它说，GET 是`router.Handle("GET", path, handlers)`的一个捷径。再看看`router.Handle("GET", path, handlers)`是什么东西：

```go
func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
	absolutePath := group.calculateAbsolutePath(relativePath)
	handlers = group.combineHandlers(handlers)
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}

// Handle registers a new request handle and middleware with the given path and method.
// The last handler should be the real handler, the other ones should be middleware that can and should be shared among different routes.
// See the example code in GitHub.
//
// For GET, POST, PUT, PATCH and DELETE requests the respective shortcut
// functions can be used.
//
// This function is intended for bulk loading and to allow the usage of less
// frequently used, non-standardized or custom methods (e.g. for internal
// communication with a proxy).
```
原来就是做了一下中间处理，注册了一个新的请求句柄。它还说GET, POST, PUT, PATCH、DELETE 都有类似的捷径。之后在 handlers中放一个匿名函数：

```go
func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.tmpl", gin.H{
			"title": "你好,前端真是太有意思了！",
		})
	}
```
这个函数就是用于渲染的函数。猜测`c.HTML()`依然是一个 shortcut:

```go
// HTML renders the HTTP template specified by its file name.
// It also updates the HTTP code and sets the Content-Type as "text/html".
// See http://golang.org/doc/articles/wiki/
func (c *Context) HTML(code int, name string, obj any) {
	instance := c.engine.HTMLRender.Instance(name, obj)
	c.Render(code, instance)
}
```
可以看到真正干活的还是 `Render()`。

点击运行，结果为：**你好,前端真是太有意思了!**

### （五）从多个模板中选择一个进行渲染
如果要渲染多个文件，该如何操作？比如我们通过输入不同的网站，服务器这时只需要选择不同的模板进行渲染就好。在这里创建了两个模板文件，`users/index.tmpl`:

```go
{{define "users/index.tmpl"}}
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <meta http-equiv="X-UA-Compatible" content="ie=edge">
        <title>users/index</title>
    </head>
    <body>
    {{.title}}
    </body>
    </html>
{{end}}
```
`posts/index.tmpl`:

```go
{{define "posts/index.tmpl"}}
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <meta http-equiv="X-UA-Compatible" content="ie=edge">
        <title>posts/index</title>
    </head>
    <body>
    {{.title}}
    </body>
    </html>
{{end}}
```
这里使用了`{{define "posts/index.tmpl"}}...{{end}}`结构，对模板文件进行了命名。
编写`main.go`:

```go
package main

import (
	"github.com/gin-gonic/gin"
	"html/template"
	"net/http"
)

func main() {
	r := gin.Default()
	
	r.LoadHTMLGlob("/Users/luliang/GoLand/gin_practice/chap5/templates/**/*")
	r.GET("/posts/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "posts/index.tmpl", gin.H{
			"title": "posts/index.tmpl",
		})
	})
	r.GET("/users/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "users/index.tmpl", gin.H{
			"title": "送你到百度！",
		})
	})
	r.Run(":9091")
}

```
我们在在浏览器地址栏输入`http://127.0.0.1:9091/users/index`可以得到：**送你到百度!**；而输入`http://127.0.0.1:9091/posts/index`可以得到：**posts/index.tmpl**。
### （六）加点东西，使得事情朝着有意思的方向进行！

我们为了使得网页画面更有趣，这里使用了多个文件，分别是index.css、index.js。编写：`index.css`:

```css
body {
    background-color: #00a7d0;
}
```
没错只有一句话。接下来编写 `index.js`:

```javascript
alert("Hello, Web!")
```
没错，也只有一句话。然后，我们把 css、js 文件引入到 `index.tmpl`中：

```go
{{define "users/index.tmpl"}}
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <meta http-equiv="X-UA-Compatible" content="ie=edge">
        <link rel="stylesheet" href="/xxx/index.css">
        <title>users/index</title>
    </head>
    <body>
    {{.title}}

    <script src="/xxx/index.js"></script>
    </body>
    </html>
{{end}}
```

接下来写`main.go`:

```go
package main

import (
	"github.com/gin-gonic/gin"
	"html/template"
	"net/http"
)

func main() {
	r := gin.Default()
	// 加载静态文件
	r.Static("/xxx", "/Users/luliang/GoLand/gin_practice/chap5/statics")
	
	r.LoadHTMLGlob("/Users/luliang/GoLand/gin_practice/chap5/templates/**/*")
	r.GET("/posts/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "posts/index.tmpl", gin.H{
			"title": "posts/index.tmpl",
		})
	})
	r.GET("/users/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "users/index.tmpl", gin.H{
			"title": "送你到百度!",
		})
	})

	r.Run(":9091")
}

```
因为我们要把 css、js 文件加载进去，因此要用 `r.Static()`函数把放置静态文件的目录加入进去。**为了复用**，第一个参数“/xxx”的意思是，只要index.tmpl文件中存在“/xxx”字段就直接指到我们设置的目录。
运行结果：首先 js 文件会首先渲染：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/3cc321ecf46846a6a2235ef9eee4e4ae_1720253815831.png?token=ANB4BCKIVDPQEZG2PCZHGW3GRD63M)
之后，css 文件会渲染：![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/00b20af54b554c749e19348a719ea21e_1720253815831.png?token=ANB4BCPSCA63REQ3SUA4BADGRD63O)

可以看到这里出现了一个超链接，那是因为我们在这个字符后面插入了一个函数：

```go
// 添加自定义函数
	r.SetFuncMap(template.FuncMap{
		"safe": func(str string) template.HTML {
			return template.HTML(str)
		},
	})
```
配合 `index.tmpl`,就可以了。
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/99077711704548b5b9e6bc986b9082b0_1720253815831.png?token=ANB4BCMU6IIS3DBBTTCKH73GRD63Q)
## 三、利用已有模板进行部署

通过以上的例子，终于学会了在模板中插入大量的css、js 进行渲染了！
首先找到某个网站，比如[站长之家](https://sc.chinaz.com/tag_moban/qianduan.html)，下载一个模板：![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/c4d8d1fabd394634a2199abddc1a3235_1720253815831.png?token=ANB4BCMZO5EHYG32RMGRFH3GRD63U)
选择第一个，下载之后解压：![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/f75b1a70464743b18c786f893f9d09ff_1720253823252.png?token=ANB4BCLCBZWOOEPIZYE43TTGRD63Y)

static 外面的` index.html`有用，其它都可以忽略。把 static 里面的文件夹复制到我们的工作目录：![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/c4f1ad3aceb5450388453349cbdca707_1720253823252.png?token=ANB4BCOCKKLF456XWHDT5HTGRD634)
同时，把 `index.html`文件复制到 `posts`（无所谓，强迫症而已），就可以使用了。

```go
	r.GET("/home", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.html", nil)
	})
```
这样，在地址栏输入：`http://127.0.0.1:9091/home`，就可以看到效果了：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/cd783d95c55e4211bbd156985e3129eb_1720253823252.png?token=ANB4BCIMIO7VHQ7TEOT3ZIDGRD64A)
渲染的相当完美！这里会存在一些小细节，比如在 index.html文件中需要把 css
、js文件的地址进行变更，变更到你放置这些文件的地址：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/0bf73972e1a9458f931d430a1b981ec5_1720253823252.png?token=ANB4BCIKRH7WV7RYERFW2M3GRD64E)
代码附上`main.go`：

```go
package main

import (
	"github.com/gin-gonic/gin"
	"html/template"
	"net/http"
)

func main() {
	r := gin.Default()
	// 加载静态文件
	r.Static("/xxx", "/Users/luliang/GoLand/gin_practice/chap5/statics")
	// 添加自定义函数
	r.SetFuncMap(template.FuncMap{
		"safe": func(str string) template.HTML {
			return template.HTML(str)
		},
	})
	r.LoadHTMLGlob("/Users/luliang/GoLand/gin_practice/chap5/templates/**/*")
	r.GET("/posts/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "posts/index.tmpl", gin.H{
			"title": "posts/index.tmpl",
		})
	})
	r.GET("/users/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "users/index.tmpl", gin.H{
			"title": "<a href='https://www.baidu.com'>送你到百度!</a>",
		})
	})
	r.GET("/home", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.html", nil)
	})

	r.Run(":9091")
}

```

这意味着，我们以后要想写网页，根本不需要进行大量的无意义的编程，利用 ChatGPT，我们可以写出大量的优秀的css、js 网页，我们要做的只是进行适量的改动，这将极大地丰富我们的创造力！

## 四、总结
本文从简单到难，对Go Web中 gin 框架下的模板渲染进行了简单的阐述。

*全文完，感谢阅读！*
