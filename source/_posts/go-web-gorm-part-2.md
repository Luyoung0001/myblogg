---
title: "Go Web 开发：GORM 项目封装（二）"
date: 2023-06-04 22:49:01
tags:
  - "Go"
  - "GORM"
  - "database"
categories:
  - "Backend Development"
---

<!--more-->

## 〇、前言
本文将会写一个前后端分离的的小项目，本文将会只实现后端。
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/37ed9ab42ca44802a2d5569d2dbafc14_1720253774709.png?token=ANB4BCIW3U5E3ZZJHQEISCTGRD6Y2)
## 一、定义全局变量与模型
本文需要一个数据库，因此将这个数据库定义为全局变量将会非常轻松。
```go
var (
	DB *gorm.DB
)

type Todo struct {
	ID     int    `json:"id"`
	Title  string `json:"title"`
	Status bool   `json:"status"`
}

```
## 二、建立数据库

```sql
database create bubble;
use bubble;
```
可以看到，已经创建了 一个名为bubble的mysql数据库：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/675e7b472fe74d3ba11aff36194724a9_1720253774709.png?token=ANB4BCLMVO4WRWTIRARYLMDGRD6Y6)
## 三、连接数据库
在 gorm 框架下，对于 mysql 等多种数据库的使用极其方便：
```go
func initMySQL() (err error) {
	// 连接数据库
	dsn := "root:pwd@tcp(127.0.0.1:3306)/bubble?charset=utf8mb4&parseTime=True&loc=Local"
	DB, err = gorm.Open("mysql", dsn)
	if err != nil {
		return err
	}
	// 判断是否连通
	return DB.DB().Ping()
}
```
## 四、载入前端静态文件
部署一个项目，假设前端可靠，那么就可以放心得写后端了：

```go
// 连接数据库
	err := initMySQL()
	if err != nil {
		panic(err)
	}
	// 模型绑定
	DB.AutoMigrate(&Todo{}) // todos
	defer func(DB *gorm.DB) {
		err := DB.Close()
		if err != nil {
			panic(err)
		}
	}(DB)

	r := gin.Default()
	r.Static("/static", "/Users/***/GoLand/gin_practice/chap18/static")
	r.LoadHTMLGlob("/Users/***/GoLand/gin_practice/chap18/templates/*")
	r.GET("/bubble", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.html", nil)
	})
```
## 五、路由组的实现
我们可以根据前端的各种请求，写一个路由组：

```go
v1Group := r.Group("v1")
	{
		// 待办事项
		v1Group.POST("/todo", func(c *gin.Context) {


		})
		// 查看所有的待办事项
		v1Group.GET("/todo", func(c *gin.Context) {

		})
		// 查看某一个待办事项
		v1Group.GET("/todo/:id", func(c *gin.Context) {

		})
		// 修改(更新) 某一个事项
		v1Group.PUT("/todo/:id", func(c *gin.Context) {

		})
		// 删除
		v1Group.DELETE("/todo/:id", func(c *gin.Context) {

		})

	}
```

## 六、功能的实现
添加事项：
```go
	v1Group.POST("/todo", func(c *gin.Context) {
			// 前端页面提交待办事项,接受请求
			// 返回响应
			var todo Todo
			c.BindJSON(&todo)
			// 从请求中把数据捞出来,存储到数据库
			if err := DB.Create(&todo).Error; err != nil {
				// 失败时的响应
				c.JSON(http.StatusOK, gin.H{
					"err": err.Error(),
				})
				return
			} else {
				// 成功的响应
				c.JSON(http.StatusOK, todo)
				//c.JSON(http.StatusOK, gin.H{
				//	"code": 2000,
				//	"msg":  "Ok",
				//	"data": todo,
				//})
			}

		})
```

查询事项：

```go
	v1Group.GET("/todo", func(c *gin.Context) {
			// 查询todo数据库中的数据所有的数据
			var todoList []Todo
			if err = DB.Find(&todoList).Error; err != nil {
				// 如果出错
				c.JSON(http.StatusOK, gin.H{
					"err": err.Error(),
				})
				return
			} else {
				// 如果成功
				c.JSON(http.StatusOK, todoList)

			}
		})
```
修改事项：

```go
	v1Group.PUT("/todo/:id", func(c *gin.Context) {
			// 拿到 id,然后查询,修改
			id, ok := c.Params.Get("id")
			if !ok {
				c.JSON(http.StatusOK, gin.H{
					"error": "id不存在",
				})
				return
			}
			var todo Todo
			if err = DB.Where("id=?", id).First(&todo).Error; err != nil {
				// 如果出错
				c.JSON(http.StatusOK, gin.H{
					"err": err.Error(),
				})
				return

			} // 更新
			c.BindJSON(&todo)
			if err = DB.Save(&todo).Error; err != nil {
				c.JSON(http.StatusOK, gin.H{
					"error": err.Error(),
				})
				return

			}
			// 成功
			c.JSON(http.StatusOK, todo)
		})
```
删除事项：

```go
		v1Group.DELETE("/todo/:id", func(c *gin.Context) {
			// 拿到 id,然后查询,修改
			id, ok := c.Params.Get("id")
			if !ok {
				c.JSON(http.StatusOK, gin.H{
					"error": "id不存在",
				})
				return
			}
			var todo Todo
			if err = DB.Where("id=?", id).Delete(&todo).Error; err != nil {
				// 如果出错
				c.JSON(http.StatusOK, gin.H{
					"err": err.Error(),
				})
				return
			}
			// 成功
			c.JSON(http.StatusOK, todo)
		})
```
这样，就完成了整个项目的后端处理。

## 七、运行起来的样子
点击运行，可以看到一个还算清爽的界面（前端写得好，与我无瓜）：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/1250a21ae8ea43a7a4f9aaf4fd252de7_1720253774709.png?token=ANB4BCJK5MICGRLK44DBEMLGRD6ZA)
添加几个事项：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/d3cfe0ab50ef45d09a30303f26fccfeb_1720253774709.png?token=ANB4BCM2JWCOFB4LQTF3A5LGRD6ZE)
查看一下数据库：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/83f2528048364af9a5a4ac9f8e541b8a_1720253782424.png?token=ANB4BCJ7BBJ7KEOB7VVPXJTGRD6ZG)
可以看到，数据库成功地将前端提交的表达做了持久化存储。再标记几个已完成：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/93069fc0eb4349689a5596f16a11f1b6_1720253782424.png?token=ANB4BCNY7OQZUBVF2XC7BFDGRD6ZK)
 再看看数据库：

 ![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/ec89186c11d04d7cbd98de8051cc9a43_1720253782424.png?token=ANB4BCKJIS7EDQY4ZFGCB3DGRD6ZM)
可以看到成功地将 status 标记成了 1。撤销看看：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/878b1788f5db4bca8e8b4e55e1e7ffce_1720253782424.png?token=ANB4BCOB24OHWMWSC2RF3YDGRD6ZO)

数据库：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/ff4feab2b12a4ed480e07d8370818a26_1720253782424.png?token=ANB4BCJLOEOBSF5GT6MFAKLGRD6ZQ)
删除看看：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/9113578ab4984201b99553af33f16a8a_1720253788978.png?token=ANB4BCPC5TPRUK2TCM6GFDDGRD6ZU)

数据库：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/81a7758830e74580b55939fad1a42ff1_1720253788978.png?token=ANB4BCLMS3UJXDXHU6LEGNLGRD6ZW)
 可以看到，后端的功能都正常。

*全文完，感谢阅读。*
