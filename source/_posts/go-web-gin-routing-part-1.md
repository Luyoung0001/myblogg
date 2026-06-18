---
title: "Go Web 开发：Gin 路由基础（一）"
date: 2023-05-31 21:33:59
tags:
  - "Go"
  - "Gin"
  - "routing"
categories:
  - "Backend Development"
---

<!--more-->

## 〇、前言
在前面，已经在[这篇文章](http://t.csdn.cn/ey0jM)中详细地讨论了 gin 框架下的模板渲染问题，这篇文章主要对 gin 框架的使用进行讨论。
## 一、不同的路由

以下可以选择不同的路由进行渲染：

```go
r := gin.Default()
	type usr struct {
		Name string `json:"name"`
		Msg  string
		Age  int
	}
	r.GET("/json", func(c *gin.Context) {
		data := gin.H{
			"name":    "小王子!",
			"massage": "hello",
			"age":     18,
		}
		c.JSON(http.StatusOK, data)
	})
	r.GET("/another_json", func(c *gin.Context) {
		data := usr{
			"小王子!",
			"hello,golang!",
			18,
		}
		c.JSON(http.StatusOK, data)
	})
```
当我们在浏览器中输入不同的 url （本例中仅仅是路径不同）时，服务器就会返回不同的数据，这是很好理解的。
### （一）URL
URL是Uniform Resource Locator的缩写，它是用于标识互联网上资源位置的字符串。URL由几个组件组成，包括协议（如HTTP或HTTPS）、主机名、端口号（可选）、路径和查询参数（可选）。URL的主要目的是在网络上定位资源，如网页、图像、视频等。

下面是一个URL的示例：
`https://www.example.com:8080/path/to/resource?param1=value1&param2=value2`

在这个示例中：

- 协议是HTTP（或HTTPS，如果使用加密连接；
- 主机名是www.example.com；
- 端口号是8080（在这个示例中是可选的，默认使用协议的默认端口）；
- 路径是/path/to/resource；
- 查询参数是param1=value1和param2=value2（用于向服务器传递额外的信息）；
- URL在浏览器中用于访问网页，也在许多其他应用程序中用于定位网络资源。

## 二、查询参数的获取
当我们在浏览器中输入不同的参数事，服务器必须对这些参数进行获取和处理，这样才能正确处理用户的需求，返回用户想要的数据。比如我们在浏览器中输入：`https://www.google.com/search?q=你好`这个 `q=你好`就是参数字段。服务器获取到参数 q 后，就对`你好`进行查询，然后返回给用户，这样就完成了搜索。

一下是一个gin框架下的参数获取示例：

```go

func main() {
	r := gin.Default()
	r.GET("/web", func(c *gin.Context) {
		// 获取浏览器发送的 query 字段
		// name := c.Query("query")
		// http://127.0.0.1:9001/web?query=杨超越&age=18
		name := c.DefaultQuery("query", "someone")
		age := c.DefaultQuery("age", "age")
		c.JSON(http.StatusOK, gin.H{
			"name": name,
			"age":  age,
		})
	})
	r.Run(":9001")
}

```
main 函数的作用就是获取用户输入的 url 中的 `query参数`以及 `age 参数`，并处理。处理的过程具体为展示给用户他们输入的 `name`、`age`。运行结果为（杨超越 2023 年 5 月 31 日时的年龄为 24 岁，下面的年龄仅供参考）：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/44da9d6708e3483aa5f8049cac674e3d_1720253795679.png?token=ANB4BCOKRCNCIPE2PUGK5ELGRD62I)
## 三、获取表单数据

我们有时候会有让用户登录的需求，后台需要对用户输入的用户名以及登录密码进行判断和处理。实际上，一般返回这种数据就是表单数据的返回。在 gin 框架里有很方便的函数。以下是一个示例。
### （一）表单数据的生成和处理
以下是一个网页 login.html，里面要求用户分别填写用户名以及密码：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>login</title>
</head>
<body>
<form action="/login" method="post" novalidate autocomplete="off">
    <div>
        <label for="username">username:</label>
        <input type="text" name="username" id="username">
    </div>
    <div>
        <label for="password">password:</label>
        <input type="password" name="password" id="password">
    </div>
    <div>
        <input type="submit" value="登陆">
    </div>

</form>
</body>

</html>
```
这个html 文件有三个 type类型的数据，分别是type="text"、type="password"、type="submit"，他是用 post 方法进行提交的。main函数是这样渲染的：

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	r := gin.Default()
	r.LoadHTMLFiles("/Users/luliang/GoLand/gin_practice/chap8/login.html", "/Users/luliang/GoLand/gin_practice/chap8/index.html")
	r.GET("/login", func(c *gin.Context) {
		c.HTML(http.StatusOK, "login.html", nil)
	})
	r.POST("/login", func(c *gin.Context) {
		// 获取 form 表单的数据
		//username := c.PostForm("username")
		//password := c.PostForm("password")
		// 第二种方式
		username := c.DefaultPostForm("username", "用户名")
		password := c.DefaultPostForm("password", "密码")
		c.HTML(http.StatusOK, "index.html", gin.H{
			"Name":     username,
			"Password": password,
		})
	})
	r.Run(":9001")

}

```
首先，通过GET方法渲染上述 login.html文件，然后 login文件中文submit 将会把表单返回到 context类型的 c：

```go
	r.GET("/login", func(c *gin.Context) {
		c.HTML(http.StatusOK, "login.html", nil)
	})
```
之后，login 里面会用 POST 方法请求，在这个请求里，对该请求的处理方法是对另一个 html 文件进行渲染：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>index</title>
</head>
<body>
<h1>hello,{{ .Name }}</h1>
<h1>你的密码是{{ .Password}}</h1>

</body>
</html>
```

这时，只需要把提价的数据从 c 中拿出来就好了：

```go
// 获取 form 表单的数据
		//username := c.PostForm("username")
		//password := c.PostForm("password")
		// 第二种方式
		username := c.DefaultPostForm("username", "用户名")
		password := c.DefaultPostForm("password", "密码")
```
拿出来之后，就可以对我们的 html 文件进行渲染了：

```go
	c.HTML(http.StatusOK, "index.html", gin.H{
			"Name":     username,
			"Password": password,
		})
```

运行结果为：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/e447b71e08984826a595298aa0ce5372_1720253795679.png?token=ANB4BCIV23UQMLOWNDEYD3DGRD62K)
我们输入数据之后，点击登陆，进行POST 请求，渲染的新的结果为：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/b5d5acdb0d02495188f446321f133e91_1720253801825.png?token=ANB4BCN5RXW7PHFDDC2ZZGDGRD62O)

为了更清晰地展示这一过程，这里会用 Postman这个软件进行模拟。

### （二）Postman进行分析
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/d72813f6ec984957a556959e7bec45c9_1720253801825.png?token=ANB4BCNR4CNVISAJJFLGEYDGRD62S)

我们在对这个 `url=http://127.0.0.1:9001/login`进行 GET 请求时，可以看到服务器进行的是：
```go
	r.GET("/login", func(c *gin.Context) {
		c.HTML(http.StatusOK, "login.html", nil)
	})
```
如果我们把 GET 请求换成 POST 请求，这时候就应该进行的是index.html渲染：

```go
r.POST("/login", func(c *gin.Context) {
		// 获取 form 表单的数据
		//username := c.PostForm("username")
		//password := c.PostForm("password")
		// 第二种方式
		username := c.DefaultPostForm("username", "用户名")
		password := c.DefaultPostForm("password", "密码")
		c.HTML(http.StatusOK, "index.html", gin.H{
			"Name":     username,
			"Password": password,
		})
	})
```

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/5704d896ef7a4787b235a8a9492cc42e_1720253801825.png?token=ANB4BCI6UYFDTSI5PCGJYSLGRD62W)
打开检查器，也可以看到它进行的是 POST 请求，携带的参数为：![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/f6254b869b2048c494109628e2d16175_1720253801825.png?token=ANB4BCMN35KO3UCRW4GVVHLGRD62Y)
渲染的是 index.html。
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/44232e3d05224ac98c3bbfc992febebf_1720253801825.png?token=ANB4BCNED2MWCVJOZYBLE73GRD622)
##  四、路径参数
路径参数和上面的参数不一样：

```go
func main() {
	r := gin.Default()
	r.GET("/user/:name/:age", func(c *gin.Context) {
		name := c.Param("name")
		age := c.Param("age")
		c.JSON(http.StatusOK, gin.H{
			"name": name,
			"age":  age,
		})
	})
	r.GET("/blog/:year/:month", func(c *gin.Context) {
		year := c.Param("year")
		month := c.Param("month")
		c.JSON(http.StatusOK, gin.H{
			"year":  year,
			"month": month,
		})
	})
	r.Run(":9001")

}

```
运行一下：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/be804b5913dd41d4bd744ab9c1f9f33c_1720253809975.png?token=ANB4BCKVYISBHUHAHW4VGOLGRD624)
可以看到，服务端会将 URL 的路径参数提取出来并进行处理：

```go
r.GET("/user/:name/:age", func(c *gin.Context) {
		name := c.Param("name")
		age := c.Param("age")
		c.JSON(http.StatusOK, gin.H{
			"name": name,
			"age":  age,
		})
	})
```
## 五、参数绑定
为了能够更方便的获取请求相关参数，提高开发效率，我们可以基于请求的Content-Type识别请求数据类型并利用反射机制自动提取请求中QueryString、form表单、JSON、XML等参数到结构体中。 下面的示例代码演示了.ShouldBind()强大的功能，它能够基于请求自动提取JSON、form表单和QueryString类型的数据，并把值绑定到指定的结构体对象：

```go
func main() {
	type user struct {
		Username string `form:"username" json:"uname"`
		Password string `form:"password" json:"pwd"`
	}
	r := gin.Default()
	r.GET("/user", func(c *gin.Context) {
		var u user
		err := c.ShouldBind(&u) // 按值传递
		if err != nil {
			c.JSON(http.StatusBadRequest, gin.H{
				"msg":   "请求的参数不正确!",
				"error": err.Error(),
			})
		} else {
			c.JSON(http.StatusOK, gin.H{
				"message": "ok!",
				"user":    u,
			})
		}

	})
	r.POST("/user", func(c *gin.Context) {
		var u user
		err := c.ShouldBind(&u)
		if err != nil {
			c.JSON(http.StatusBadRequest, gin.H{
				"msg":   "请求的参数不正确!",
				"error": err.Error(),
			})
		} else {
			c.JSON(http.StatusOK, gin.H{
				"message": "ok!",
				"user":    u,
			})
		}

	})

	r.Run(":9001")
}
```
看看这个请求的运行效果：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/d2f2097da950441e81d1cadec4d7fa3f_1720253809975.png?token=ANB4BCI3ZXB5RYLXNBUCCULGRD63A)
`ShouldBind()`就是这么强大！

## 六、文件上传
我们有时候需要对文件上传，上传文件在前端由用户进行操作，我们后端接收到用户文件上传的文件时，就对该文件进行一些可能的操作。
前端页面：

```html
<!DOCTYPE html>
<html>
<head>
    <title>index</title>
</head>
<body>
<form action="/upload" method="post" enctype="multipart/form-data">
    <input type="file" name="f1">
    <input type="submit" value="upload">
</form>
</body>
</html>

```
这里通过 post 方法，将文件submit 到了context 类型的 c对象中，我们继续处理：

```go
r := gin.Default()
	r.LoadHTMLFiles("/Users/luliang/GoLand/gin_practice/chap11/index.html")
	r.GET("/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.html", nil)

	})
	r.POST("/upload", func(c *gin.Context) {
		// 读取文件
		f, err := c.FormFile("f1")
		if err != nil {
			c.JSON(http.StatusBadRequest, gin.H{
				"error": err.Error(),
			})
		} else {
			// 将读取的文件保存到本地
			// dsf := fmt.Sprintf("./%s", f.Filename)
			dsf := path.Join("./", f.Filename)
			err = c.SaveUploadedFile(f, dsf)
			if err != nil {
				return
			} // 将文件 f 保存到 dsf 中
			c.JSON(http.StatusOK, gin.H{
				"status": "OK",
			})

		}

	})
```
过程为，先通过 GET请求，访问到index.html：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/78ec14cd74554e8baa61a91c68e90ea4_1720253809975.png?token=ANB4BCMDXGWTJNSEM67J7VTGRD63C)随便选一个文件：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/5fca271bf98746d491315b2172439267_1720253809975.png?token=ANB4BCPA6TNCD3TCTJGT6FDGRD63E)
点击上传：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/0fc913b8289942838077c76cf08d3a7f_1720253809975.png?token=ANB4BCOBMFJWQC243HWOTITGRD63G)
在这个过程中，我们对 context 的c进行获取文件，存储并发送了一个合适的响应：

```go
r.POST("/upload", func(c *gin.Context) {
		// 读取文件
		f, err := c.FormFile("f1")
		if err != nil {
			c.JSON(http.StatusBadRequest, gin.H{
				"error": err.Error(),
			})
		} else {
			// 将读取的文件保存到本地
			// dsf := fmt.Sprintf("./%s", f.Filename)
			dsf := path.Join("./", f.Filename)
			err = c.SaveUploadedFile(f, dsf)
			if err != nil {
				return
			} // 将文件 f 保存到 dsf 中
			c.JSON(http.StatusOK, gin.H{
				"status": "OK",
			})

		}
	})
```

## 七、路由重定向
这个过程很简单，就是当用户访问了某些请求之后，我们可以将该请求转接到另一个 URL 上去，类似于 DNS 劫持：

```go
func main() {
	r := gin.Default()
	r.GET("/index", func(c *gin.Context) {
		//c.JSON(http.StatusOK,gin.H{
		//	"status":"Ok",
		//})
		c.Redirect(http.StatusMovedPermanently, "https://www.google.com")

	})
	r.GET("/a", func(c *gin.Context) {
		// 跳转到 b
		c.Request.URL.Path = "/b" // 修改请求 URL
		r.HandleContext(c)        // 继续处理 c

	})
	// 处理所有的方法
	r.GET("/b", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"status": "ok",
		})

	})

	r.Run(":9001")

}

```

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/83e6b7afbaa740bcb8c6ceea55da09b7_1720253815831.png?token=ANB4BCOU2WF63WOOL4BPHHTGRD63I)
可以看到它重复定向到了`url=http://127.0.0.1:9001/a`。

*全文完，感谢阅读。*


