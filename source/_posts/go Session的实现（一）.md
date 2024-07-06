---
title: go Session的实现（一）
date: 2023-09-02 16:57:25
tags: golang 后端 前端 网络
categories: Golang 前端 Web
---

<!--more-->

## 〇、前言
众所周知，**http协议是无状态的**，这对于服务器确认是哪一个客户端在发请求是不可能的，因此为了能确认到，通常方法是让客户端发送请求时带上身份信息。容易想到的方法就是客户端在提交信息时，带上自己的账户和密码。但是这样存在着严重的安全问题，可以改进的方法就是，服务器给一个确定的客户端返回一个唯一 id，客户端将这个 id 保存在本地，每次发送请求时只需要携带着这个 id，就可以做到较好的验证（当然也存在着安全问题，这个后面再说）。

这个方法就是 现今很成熟的 session、cookie 技术。session和cookie的目的相同，都是为了克服http协议无状态的缺陷，但完成的方法不同。**session通过cookie**，在客户端保存session id，而将用户的其他会话消息保存在服务端的session对象中。与此相对的，**cookie需要将所有信息都保存在客户端**。因此cookie存在着一定的安全隐患，例如本地cookie中保存的用户名密码被破译，或cookie被其他网站收集。

本文将尝试着实现一个成熟的 go session，从而实现会话保持。
***思维导图如下：***
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/3d30f872a44b423ea3b4291e44b03b6f_1720253724529.png?token=ANB4BCPJSZ2JN4DMHWXTND3GRD6V6)


## 一、架构设计
### 0、思维导图
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/9cf6a34839ab41c48d301fce9d20bbc2_1720253732241.png?token=ANB4BCKNVYO3XI7EKLKKNA3GRD6WC)

### 1、管理器

```go
type Manager struct {
	cookieName  string
	lock        sync.Mutex
	provider    Provider
	maxLifeTime int64
}

```
其中 Provider 是一个接口：

```go
// Provider 接口
type Provider interface {
	SessionInit(sid string) (Session, error) // SessionInit函数实现Session的初始化，操作成功则返回此新的Session变量
	SessionRead(sid string) (Session, error) // SessionRead函数返回sid所代表的Session变量.如果不存在，那么将以sid为参数调用SessionInit函数创建并返回一个新的Session变量
	SessionDestroy(sid string) error         // SessionDestroy函数用来销毁sid对应的Session变量
	SessionGC(maxLifeTime int64)             // SessionGC根据maxLifeTime来删除过期的数据
}
```
这里又定义了一个Provider 结构体，它实现了 Provider 接口：

```go
// Provider 实现接口 Provider

func (pder *Provider) SessionInit(sid string) (session.Session, error) {
	// 根据 sid 创建一个 SessionStore
	pder.lock.Lock()
	defer pder.lock.Unlock()
	v := make(map[interface{}]interface{})
	// 同时更新两个字段
	newsess := &SessionStore{sid: sid, timeAccessed: time.Now(), value: v}
	// list 用于GC
	element := pder.list.PushBack(newsess)
	// 存放 kv
	pder.sessions[sid] = element
	return newsess, nil
}

func (pder *Provider) SessionRead(sid string) (session.Session, error) {
	if element, ok := pder.sessions[sid]; ok {
		return element.Value.(*SessionStore), nil
	} else {
		sess, err := pder.SessionInit(sid)
		return sess, err
	}
}

// 服务端 session 销毁

func (pder *Provider) SessionDestroy(sid string) error {
	if element, ok := pder.sessions[sid]; ok {
		delete(pder.sessions, sid)
		pder.list.Remove(element)
		return nil
	}
	return nil
}

// 回收过期的 cookie

func (pder *Provider) SessionGC(maxlifetime int64) {
	pder.lock.Lock()
	defer pder.lock.Unlock()

	for {
		element := pder.list.Back()
		if element == nil {
			break
		}
		if (element.Value.(*SessionStore).timeAccessed.Unix() + maxlifetime) < time.Now().Unix() {
			// 更新两者的值

			// 垃圾回收
			pder.list.Remove(element)
			// 删除 map 中的kv
			delete(pder.sessions, element.Value.(*SessionStore).sid)
		} else {
			break
		}
	}
}

func (pder *Provider) SessionUpdate(sid string) error {
	pder.lock.Lock()
	defer pder.lock.Unlock()
	if element, ok := pder.sessions[sid]; ok {
		// 这里更新也就更新了个时间,这意味着 session 的生命得到了延长
		element.Value.(*SessionStore).timeAccessed = time.Now()
		pder.list.MoveToFront(element)
		return nil
	}
	return nil
}
```
管理器 Manager 实现的方法：

```go
// 创建 Session

func (manager *Manager) SessionStart(w http.ResponseWriter, r *http.Request) (session Session) {
	manager.lock.Lock()
	defer manager.lock.Unlock()
	cookie, err := r.Cookie(manager.cookieName)
	if err != nil || cookie.Value == "" {
		// 查看是否为当前客户端注册过名为 gosessionid 的 cookie,如果没有注册过,就为客户端创建一个该 cookie

		// 创建 sessionID
		sid := manager.sessionID()
		// 创建一个 session 接口,这其实是一个 创建完成的 SessionStore ,SessionStore 实现了该接口
		session, _ = manager.provider.SessionInit(sid)
		// 创建 cookie
		cookie := http.Cookie{Name: manager.cookieName, Value: url.QueryEscape(sid), Path: "/", HttpOnly: true, MaxAge: int(manager.maxLifeTime)}
		http.SetCookie(w, &cookie)
	} else {
		sid, _ := url.QueryUnescape(cookie.Value)
		session, _ = manager.provider.SessionRead(sid)
	}
	return
}

// Session 重置

func (manager *Manager) SessionDestroy(w http.ResponseWriter, r *http.Request) {
	cookie, err := r.Cookie(manager.cookieName)
	if err != nil || cookie.Value == "" {
		return
	} else {
		manager.lock.Lock()
		defer manager.lock.Unlock()
		err := manager.provider.SessionDestroy(cookie.Value)
		if err != nil {
			return
		}
		expiration := time.Now()
		cookie := http.Cookie{Name: manager.cookieName, Path: "/", HttpOnly: true, Expires: expiration, MaxAge: -1}
		http.SetCookie(w, &cookie)
	}
}

// Session 回收

func (manager *Manager) GC() {
	manager.lock.Lock()
	defer manager.lock.Unlock()
	manager.provider.SessionGC(manager.maxLifeTime)
	// 每 20秒触发一次
	time.AfterFunc(time.Second*20, func() { manager.GC() })
}
```

### 2、sessions存放
在 Provider 结构体中：

```go
sessions map[string]*list.Element // 存放 sessionStores
list     *list.List               // 用来做gc
```
sessions 中存放不同客户端的 session，而 list 中也会同时刷新，它用来回收过期的 session。
每一个session用 SessionStore 结构体来存储。

Session 接口：
```go
// Session 接口
type Session interface {
	Set(key, value interface{}) error // 设置 session 的值
	Get(key interface{}) interface{}  // 获取 session 的值
	Delete(key interface{}) error     // 删除 session 的值
	SessionID() string                // 返回当前 session 的 ID
}
```
这个接口，由 SessionStore 实现：
```go
// SessionStore 结构体

type SessionStore struct {
	sid          string                      // session id唯一标识
	timeAccessed time.Time                   // 最后访问时间
	value        map[interface{}]interface{} // 值
}
```

```go
// SessionStore 实现 Session 接口

func (st *SessionStore) Set(key, value interface{}) error {
	st.value[key] = value
	err := pder.SessionUpdate(st.sid)
	if err != nil {
		return err
	}
	return nil
}

func (st *SessionStore) Get(key interface{}) interface{} {
	err := pder.SessionUpdate(st.sid)
	if err != nil {
		return nil
	}
	if v, ok := st.value[key]; ok {
		return v
	} else {
		return nil
	}
}

func (st *SessionStore) Delete(key interface{}) error {
	delete(st.value, key)
	err := pder.SessionUpdate(st.sid)
	if err != nil {
		return err
	}
	return nil
}

func (st *SessionStore) SessionID() string {
	return st.sid
}
```

## 二、实现细节
### 1、provider 注册表

```go
// provider 注册表
var provides = make(map[string]Provider)
```
任何一个 Maneger 在创建之前，都需要在 provider 注册表中注册。因此在创建一个全局注册表pder，并注册，这应该是 init 的：

```go
// 创建全局 pder
var pder = &Provider{list: list.New()}
func init() {
	pder.sessions = make(map[string]*list.Element)
	session.Register("memory", pder)
}
```
注册器：
```go
func Register(name string, provider Provider) {
	if provider == nil {
		panic("session: Register provide is nil")
	}
	if _, dup := provides[name]; dup {
		panic("session: Register called twice for provide " + name)
	}
	provides[name] = provider
}
```
### 2、全局管理器
```go
var globalSessions *session.Manager
func init() {
	globalSessions, _ = session.NewManager("memory", "gosessionid", 3600)
	go globalSessions.GC()
}
```
这个管理器就是一个 cookie 管理器，它只对cookie名字为`gosessionid`的 cookie 负责。

```go
func NewManager(provideName, cookieName string, maxlifetime int64) (*Manager, error) {
	provider, ok := provides[provideName]
	if !ok {
		return nil, fmt.Errorf("session: unknown provide %q (forgotten import?)", provideName)
	}
	return &Manager{provider: provider, cookieName: cookieName, maxLifeTime: maxlifetime}, nil
}
```
### 3、案例演示
现在已经初始化好了，就等着客户端访问了。
现在我们写一个很简单的计数器，前端访问的时候，自动+1：

```go
func count(c *gin.Context) {
	sess := globalSessions.SessionStart(c.Writer, c.Request)
	ct := sess.Get("countnum")
	if ct == nil {
		err := sess.Set("countnum", 1)
		if err != nil {
			return
		}
	} else {
		// 更新
		err := sess.Set("countnum", ct.(int)+1)
		if err != nil {
			return
		}
	}
	t, err := template.ParseFiles("template/count.html")
	if err != nil {
		fmt.Println(err)
	}
	c.Writer.Header().Set("Content-Type", "text/html")
	err = t.Execute(c.Writer, sess.Get("countnum"))
	if err != nil {
		return
	}
}
```
当中的`count.html`这样写：

```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Count</title>
</head>

<body>
  <h1>Hi. Now count:{{.}}</h1>
</body>

</html>
```
`main.go`这样写：

```go
package main

import (
	_ "Go_Web/memory"
	"Go_Web/session"
	"fmt"
	"github.com/gin-gonic/gin"
	"html/template"
	"net/http"
)

// 全局 sessions 管理器
var globalSessions *session.Manager

// init 初始化

func init() {
	globalSessions, _ = session.NewManager("memory", "gosessionid",20)
	go globalSessions.GC()
}

func count(c *gin.Context) {
	sess := globalSessions.SessionStart(c.Writer, c.Request)
	ct := sess.Get("countnum")
	if ct == nil {
		err := sess.Set("countnum", 1)
		if err != nil {
			return
		}
	} else {
		// 更新
		err := sess.Set("countnum", ct.(int)+1)
		if err != nil {
			return
		}
	}
	t, err := template.ParseFiles("template/count.html")
	if err != nil {
		fmt.Println(err)
	}
	c.Writer.Header().Set("Content-Type", "text/html")
	err = t.Execute(c.Writer, sess.Get("countnum"))
	if err != nil {
		return
	}
}

func main() {
	r := gin.Default()
	r.GET("/count", count)
	err := r.Run(":9000")
	if err != nil {
		return
	}

}
```
我们把 session 的过期时间设为 20 秒，这样可以 更快的看到过期效果。
现在把服务器启动，来看看整个过程。
编译运行之后，在浏览器访问 count:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/c0cb43c384164d7488e221059700628d_1720253732241.png?token=ANB4BCIWCWXK4VXWU3RUZYLGRD6WG)

看下 cookie:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/a0be6121e20c40a589d416ac57a960c8_1720253732241.png?token=ANB4BCOPRWF7X6WFVF7NO73GRD6WK)
可以继续点击，这个只要在 20 秒之内点击，cookie 就不回过期，因为每次发送请求都会更新 sessionStore:

```go
err := sess.Set("countnum", ct.(int)+1)
```

```go
// SessionStore 实现 Session 接口

func (st *SessionStore) Set(key, value interface{}) error {
	st.value[key] = value
	err := pder.SessionUpdate(st.sid)
	if err != nil {
		return err
	}
	return nil
}
```

```go
func (pder *Provider) SessionUpdate(sid string) error {
	pder.lock.Lock()
	defer pder.lock.Unlock()
	if element, ok := pder.sessions[sid]; ok {
		// 这里更新也就更新了个时间,这意味着 session 的生命得到了延长
		element.Value.(*SessionStore).timeAccessed = time.Now()
		pder.list.MoveToFront(element)
		return nil
	}
	return nil
}
```

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/64944a7571734eec94151da19155adae_1720253732241.png?token=ANB4BCKKDZ2XGPQG5YSRPN3GRD6WM)
不要点击等 20 秒等它过期，再点一下：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/d825e884abc1476d8838eb9dd6a26f65_1720253732241.png?token=ANB4BCJWYAIJHU2ON774QJDGRD6WQ)
可以看到已经过期了，再查看下 cookie：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/fddcc1aeec694e888b9e80a838060eeb_1720253740236.png?token=ANB4BCKZJDPTJUGUVM3D7XLGRD6WS)
可以看到 sessionId 并没有变，这是因为就算本地 cookie过期，当发送请求时，服务器依然会拿到这个 cookie。
session 过期的时候，服务器会执行：

```go
// 回收过期的 cookie

func (pder *Provider) SessionGC(maxlifetime int64) {
	pder.lock.Lock()
	defer pder.lock.Unlock()

	for {
		element := pder.list.Back()
		if element == nil {
			break
		}
		if (element.Value.(*SessionStore).timeAccessed.Unix() + maxlifetime) < time.Now().Unix() {
			// 更新两者的值

			// 垃圾回收
			pder.list.Remove(element)
			// 删除 map 中的kv
			delete(pder.sessions, element.Value.(*SessionStore).sid)
		} else {
			break
		}
	}
}
```
这意味着，pder 中的list 和 sessions 中都不存在 键为`countnum`的`sessionStore`。但是依然会执行：

```go
	sid, _ := url.QueryUnescape(cookie.Value)
	session, _ = manager.provider.SessionRead(sid)
```
SessionRead():
```go
func (pder *Provider) SessionRead(sid string) (session.Session, error) {
	if element, ok := pder.sessions[sid]; ok {
		return element.Value.(*SessionStore), nil
	} else {
		sess, err := pder.SessionInit(sid)
		return sess, err
	}
}
```
执行SessionRead()的时候，由于 session 已经被删除，只能执行`pder.SessionInit(sid)`了，因此，服务器会创建一个和原来一样的 sessionId。之后count()自然就会执行`err := sess.Set("countnum", 1)`。
```go
ct := sess.Get("countnum")
	if ct == nil {
		err := sess.Set("countnum", 1)
		if err != nil {
			return
		}
	} else {
		// 更新
		err := sess.Set("countnum", ct.(int)+1)
		if err != nil {
			return
		}
	}
```
至此，整个过程就完了。
## 二、session 劫持
session劫持是一种广泛存在的比较严重的安全威胁，在session技术中，客户端和服务端通过session的标识符来维护会话， 但这个标识符很容易就能被嗅探到，从而被其他人利用。它是中间人攻击的一种类型。

这个服务是靠着 sessionid维持的，所以一旦这个 sessionid 泄露，被另一个客户端获取，就可以冒名顶替干一些操作（把过期时间设置长一点）。
首先在 Chrome 中访问服务器的服务，点击到随便一个数字：

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/e4c865f50c6244068a7355819d1e7d0a_1720253740236.png?token=ANB4BCIMYN4NY7EGD5ULOMDGRD6WU)
然后打开 cookie，复制：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/1924e7e8229441f082f5093c78716ca2_1720253740236.png?token=ANB4BCPRKGQEYXZHBPJNIDDGRD6WY)
再打开FireFox，随便找一个 cookie 管理器，创建一个 cookie：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/85d7535d88a445779d064e410ad113c2_1720253740236.png?token=ANB4BCM63PWD5SL4BO5UI7LGRD6W2)
保存，直接访问服务器count 服务：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/dc25f1a76b2a4badab350ab5c1f532fe_1720253740236.png?token=ANB4BCM7XEKRA4DG6X5OTB3GRD6W4)
可以看到已经实现了“冒名顶替”。

*全文完，感谢阅读。*
