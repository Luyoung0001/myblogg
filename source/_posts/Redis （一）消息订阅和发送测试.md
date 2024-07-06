---
title: Redis （一）消息订阅和发送测试
date: 2023-07-31 21:47:22
tags: redis bootstrap 数据库
categories: Golang Web
---

<!--more-->

## 〇、redis 配置

### 1、概况
本文基于 Ubuntu20.04 云服务器配置Redis，且在本地进行 Redis 测试。
### 2、目录概况 

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/b00269f787ab457b87dbff470e451601_1720253746882.png?token=ANB4BCNQZTBEHAWGDAD4JN3GRD6W6)

## 一、配置文件
位于 `/config/app.yml` 中，目的用于 Redis 初始化：

```bash
redis:
  addr: "39.104.**.28:6379"
  password: "password"
  DB: 0
  poolSize: 30
  minIdleConn: 30
```
## 二、main 文件

```go
package main

import (
	"ChatTest/router"
	"ChatTest/utils"
	"github.com/gin-gonic/gin"
)

func main() {
	utils.InitConfig()
	utils.InitRedis()
	r := gin.Default()
	r = router.Router()
	r.Run(":8000")
}

```
## 二、初始化文件

位于 `/utils/system_init.go` 文件中，目的用于初始化相关：
```bash
package utils

import (
	"fmt"
	"github.com/go-redis/redis/v8"
	"github.com/spf13/viper"
)

var (
	Red *redis.Client
)

// 初始化初始化文件

func InitConfig() {
	viper.SetConfigName("app")
	viper.AddConfigPath("/Users/luliang/GoLand/ChatTest/config") //带绝对路径
	err := viper.ReadInConfig()
	if err != nil {
		fmt.Println(err)
	}
}

// 初始化 Redis

func InitRedis() {
	Red = redis.NewClient(&redis.Options{
		Addr:         viper.GetString("redis.addr"),
		Password:     viper.GetString("redis.password"),
		DB:           viper.GetInt("redis.DB"),
		PoolSize:     viper.GetInt("redis.minIdleConn"),
		MinIdleConns: viper.GetInt("redis.minIdleConn"),
	})
	fmt.Println("config Redis:", viper.Get("redis"))
}

```
## 三、路由文件
路由文件位于 `/router/app.go` 中，目的是建立路由：
```go
package router

import (
	"ChatTest/service"
	"github.com/gin-gonic/gin"
)

func Router() *gin.Engine {
	r := gin.Default()
	r.GET("/send", service.SendMsg)
	r.GET("/recv", service.RecvMsg)
	return r
}
```
## 四、实现服务
位于`/service/message.go`中，是 HandlerFunc，且实现服务:

```go
package service

import (
	"ChatTest/utils"
	"context"
	"github.com/gin-gonic/gin"
)

func SendMsg(c *gin.Context) {
	cmd := utils.Red.Publish(context.Background(), "myRedis", "Hello, MyRedis0001!")
	if cmd != nil {
		c.JSON(200, gin.H{
			"code":    0,
			"message": "发送成功!",
		})
		return
	}
	c.JSON(200, gin.H{
		"code":    -1,
		"message": "发送失败!",
	})

}

func RecvMsg(c *gin.Context) {
	pubSub := utils.Red.Subscribe(context.Background(), "myRedis")
	defer pubSub.Close()

	ch := pubSub.Channel()

	for msg := range ch {
		c.JSON(200, gin.H{
			"code":    0,
			"message": msg.Payload,
		})
		// 根据业务逻辑决定是否终止循环并返回响应
		return
	}

	// 如果没有接收到消息，可以根据需要返回响应
	c.JSON(200, gin.H{
		"code":    -1,
		"message": "接受失败!",
	})
}

```

## 五、运行流程

这里面的核心就是Redis 的连接，以及在 Redis 中发布消息和订阅消息了。
### 1、消息的发布

```go
cmd := utils.Red.Publish(context.Background(), "myRedis", "Hello, MyRedis0001!")
```
调用Publish() 函数发布一条消息，这个 Publish() 是 go-redis中封装好的方法。


### 2、消息的订阅

```go
pubSub := utils.Red.Subscribe(context.Background(), "myRedis")
```

可以看到，go-redis 中使用消息的订阅和发布功能，可以使得消息发送和接受的过程异常简单！

*全文完，感谢阅读！*

