---
title: Go Web下gin框架使用（二）
date: 2023-06-01 20:00:00
tags:
	- golang
	- 后端
	- 前端
	- 网络
categories: Golang 前端
---

<!--more-->

## 〇、gin 路由
Gin是一个用于构建Web应用程序的Go语言框架，它具有简单、快速、灵活的特点。在Gin中，可以使用路由来定义URL和处理程序之间的映射关系。

```go
r := gin.Default()
	// 访问 /index 这个路由
	// 获取信息
	r.GET("/index", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"method": "GET",
		})
	})
	// 提交
	r.POST("/index", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"method": "POST",
		})

	})
	// 更新数据
	r.PUT("/index", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"method": "PUT",
		})

	})
	// 删除数据
	r.DELETE("/index", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"method": "DELETE",
		})

	})
```
还有一个可以匹配所有请求方法的Any方法如下：
```go
// 请求方法大杂烩
	r.Any("/user", func(c *gin.Context) {
		switch c.Request.Method {
		case "GET":
			c.JSON(http.StatusOK, gin.H{
				"method": "GET",
			})
		case "PUT":
			c.JSON(http.StatusOK, gin.H{
				"method": "PUT",
			})
		}
		c.JSON(http.StatusOK, gin.H{
			"method": "Any",
		})

	})
```

如果没有配到路由怎么办？我们有时候输入了一个错误的地址，这时候服务器应该报 404 错误并且返回一个特定的页面：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/a0796091f1f54911b39869ec22e9a8ee_1720253788978.png?token=ANB4BCM7DX3Q47XHQEWLQMLGRD6Z2)
在 gin 框架下，我们可以利用`r.NoRoute()`这样写：

```go
	r.NoRoute(func(c *gin.Context) {
		c.JSON(http.StatusNotFound, gin.H{
			"msg": "你似乎来到了陌生的地带~",
		})

	})
```
运行结果为：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/a693bbad7d1c43b88101cd2d92d5fce7_1720253788978.png?token=ANB4BCIFKXWIJXFFQLVHFETGRD6Z4)

## 一、路由组
如果我们想实现很多的这样的路由：
- http://127.0.0.1:9001/user/a
- http://127.0.0.1:9001/user/b
- http://127.0.0.1:9001/user/c
- http://127.0.0.1:9001/user/d

如果一条一条写，将会很繁琐，于是路由组的概念就被提出来了。路由组是**递归定义**的，也就是说，路由组的成员也可以是一个路由组：
- http://127.0.0.1:9001/user/a/1
- http://127.0.0.1:9001/user/a/2
- http://127.0.0.1:9001/user/a/3
- http://127.0.0.1:9001/user/a/4


```go
videoGroup := r.Group("/video")
	{
		videoGroup.GET("/index", func(c *gin.Context) {
			c.JSON(http.StatusOK, gin.H{
				"msg": "/video/index",
			})

		})
		videoGroup.GET("/xxx", func(c *gin.Context) {
			c.JSON(http.StatusOK, gin.H{
				"msg": "video/xxx",
			})

		})
		videoGroup.GET("/ooo", func(c *gin.Context) {
			c.JSON(http.StatusOK, gin.H{
				"msg": "video/ooo",
			})

		})
	}
```
运行结果如下：

![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/83b9cd8081994784b1146918a6978bbb_1720253788978.png?token=ANB4BCPBDWZJUM7D266UU43GRD6Z6)

## 二、中间件
Gin框架允许开发者在处理请求的过程中，加入用户自己的钩子（Hook）函数。这个钩子函数就叫中间件，中间件适合处理一些公共的业务逻辑，比如登录认证、权限校验、数据分页、记录日志、耗时统计等。

比如以下就是一个很简单的中间件的示例：

```go
func indexHandler(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"index": "Ok",
	})
}
```
配合 main 函数使用：

```go
func main() {
	r := gin.Default()
	r.GET("/index", indexHandler,func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"shop": "welcome!",
		})
	})
	r.Run(":9001")

}
```
GET请求的时候，程序会自动执行 `indexHandler()`这个函数，然后才会继续执行后面的匿名函数。因此`indexHandler()`就叫做中间件。

再来看看更为一般的中间件：

```go
// 定义一个新的中间件
func m1(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"m1": "before!",
	})

	start := time.Now()
	c.Next()	// 停止调用后续的函数
	cost := time.Since(start)
	fmt.Printf("花费时间:%v\n", cost)
	c.JSON(http.StatusOK, gin.H{
		"m1": "after!",
	})

}
```
这个中间件有具有更完整的功能了。首先，它会打印`"m1": "before!"`，之后它会获取一个时间戳，然后会调用后面的`handler`，也就是匿名函数，这个指示由`c.Next()`发出，这个函数运行完成后会打印真个过程消耗的时间。之后这个中间件继续会打印`"m1": "after!"`。

运行结果如下：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/f7ee2fe204824884ae701a00ee2ddc05_1720253795679.png?token=ANB4BCLS3OB6HV6OWV4P6F3GRD62A)

*全文完，感谢阅读。*
