---
title: "Gin 源码笔记：gin.Context 的职责与结构"
date: 2023-08-01 11:40:21
tags:
  - "Go"
  - "Gin"
  - "source code"
categories:
  - "Backend Development"
---

<!--more-->

## 〇、前言
Context 是 gin 中最重要的部分。 例如，它允许我们在中间件之间传递变量、管理流程、验证请求的 JSON 并呈现 JSON 响应。

Context 中封装了原生的 Go HTTP 请求和响应对象，同时还提供了一些方法，用于获取请求和响应的信息、设置响应头、设置响应状态码等操作。

在 Gin 中，Context 是通过中间件来传递的。在处理 HTTP 请求时，Gin 会依次执行注册的中间件，每个中间件可以对 Context 进行一些操作，然后将 Context 传递给下一个中间件。
## 一、gin.Context 结构

```go
type Context struct {
	writermem responseWriter
	Request   *http.Request
	Writer    ResponseWriter

	Params   Params
	handlers HandlersChain
	index    int8
	fullPath string

	engine       *Engine
	params       *Params
	skippedNodes *[]skippedNode

	// 该互斥锁保护键映射。
	mu sync.RWMutex

	// Keys 是专门用于每个请求上下文的键/值对。
	Keys map[string]any

	// 错误是附加到使用此上下文的所有处理程序/中间件的错误列表。
	Errors errorMsgs

	// 已接受定义了用于内容协商的手动接受格式的列表。
	Accepted []string

	// queryCache 缓存 c.Request.URL.Query() 的查询结果。
	queryCache url.Values

	// formCache 缓存 c.Request.PostForm，其中包含从 POST、PATCH 或 PUT 正文参数解析的表单数据。
	formCache url.Values

	// SameSite 允许服务器定义 cookie 属性，从而使浏览器无法随跨站点请求发送此 cookie。
	sameSite http.SameSite
}
```
这里面需要重点关注的的几个字段就是：

```go
	writermem responseWriter
	Request   *http.Request
	Writer    ResponseWriter
	Params    Params
	params    *Params

```

### 1、writermem responseWriter

```go
type responseWriter struct {
	http.ResponseWriter
	size   int
	status int
}
```
responseWriter 包含了一个 interface 类型 ResponseWriter：

```go
type ResponseWriter interface {
	// Header returns the header map that will be sent by WriteHeader.
	Header() Header
	// Write writes the data to the connection as part of an HTTP reply.
	Write([]byte) (int, error)
	// WriteHeader sends an HTTP response header with the provided status code.
	WriteHeader(statusCode int)
}
```
### 2、Request   *http.Request
Request 表示服务器接收到并由客户端发送的 HTTP 请求，这也是一个复杂的结构：
```go
type Request struct {
    Method           string
    URL              *url.URL
    Proto            string
    ProtoMajor       int
    ProtoMinor       int
    Header           Header
    Body             io.ReadCloser
    GetBody          func() (io.ReadCloser, error)
    ContentLength    int64
    TransferEncoding []string
    Close            bool
    Host             string
    Form             url.Values
    PostForm         url.Values
    MultipartForm    *multipart.Form
    Trailer          Header
    RemoteAddr       string
    RequestURI       string
    TLS              *tls.ConnectionState
    Cancel           <-chan struct{}
    Response         *Response
    ctx              context.Context
}
```
可以看到里面提供了各种我们常用的字段：

```go
	Method           string
    URL              *url.URL
    Header           Header
    Host             string
    Form             url.Values
    PostForm         url.Values
    Response         *Response
    ctx              context.Context
```
看到这里有一个 *Response：

```go
type Response struct {
    Status           string
    StatusCode       int
    Proto            string
    ProtoMajor       int
    ProtoMinor       int
    Header           Header
    Body             io.ReadCloser
    ContentLength    int64
    TransferEncoding []string
    Close            bool
    Uncompressed     bool
    Trailer          Header
    Request          *Request
    TLS              *tls.ConnectionState
}
```
关于：Response是导致创建此请求的重定向响应，该字段**仅在客户端重定向期间填充**。看到这儿就放心了，不然也太繁杂了。

关于 Context：上下文携带截止日期、取消信号和跨 API 边界的其他值。 Context 的方法可以同时被多个 goroutine 调用。

context.Context：是 Go 标准库中的一个接口类型，用于在 Goroutine 之间传递上下文信息。

context.Context 可以在 Goroutine 之间传递信息，例如传递请求 ID、数据库连接、请求超时等信息。context.Context 的具体实现是由各种库和框架提供的，Gin 框架中提供了一个 gin.Context 的实现，用于在 Gin 框架中使用 context.Context。

```go
type Context interface {
	// 返回时间
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```
这个 Context 依然是一个 interface，interface 的优点很明显，能简化结构，只需要写清楚使用协议就好。

### 3、Writer    ResponseWriter
这个也是一个接口，它提供了一系列方法来写请求内容什么的：
```go
type ResponseWriter interface {
    http.ResponseWriter
    http.Hijacker
    http.Flusher
    http.CloseNotifier
    Status() int
    Size() int
    WriteString(string) (int, error)
    Written() bool
    WriteHeaderNow()
    Pusher() http.Pusher
}
```
比如：

```go
func (w *responseWriter) Write(data []byte) (n int, err error) {
	w.WriteHeaderNow()
	n, err = w.ResponseWriter.Write(data)
	w.size += n
	return
}

func (w *responseWriter) WriteString(s string) (n int, err error) {
	w.WriteHeaderNow()
	n, err = io.WriteString(w.ResponseWriter, s)
	w.size += n
	return
}

func (w *responseWriter) Status() int {
	return w.status
}
...

```
### 4、Params    Params
首先：

```go
type Param struct {
	Key   string
	Value string
}
...

type Params []Param
```
可以看到 Params 是Param的 slice，params  则是字段 Params 的一个指针，方便参数传递，避免值拷贝。

它有一些方法：

```go

func (ps Params) Get(name string) (string, bool) {
	for _, entry := range ps {
		if entry.Key == name {
			return entry.Value, true
		}
	}
	return "", false
}
func (ps Params) ByName(name string) (va string) {
	va, _ = ps.Get(name)
	return
}
```
通过 Get() 可以获取参数列表中的某个参数。


## 二、gin.Context 的功能
它提供的大量的方法。gin.Context 是 Gin 框架中的一个结构体类型，用于封装 HTTP 请求和响应的信息，以及提供一些方法，用于获取请求和响应的信息、设置响应头、设置响应状态码等操作。gin.Context 只在 Gin 框架内部使用，用于处理 HTTP 请求和响应。它与 HTTP 请求和响应一一对应，**每个 HTTP 请求都会创建一个新的 gin.Context 对象**，并在处理过程中传递。

```go
BindWith(obj any, b binding.Binding) error
reset()
Copy() *gin.Context
HandlerName() string
HandlerNames() []string
Handler() gin.HandlerFunc
FullPath() string
Next()
IsAborted() bool
Abort()
AbortWithStatus(code int)
AbortWithStatusJSON(code int, jsonObj any)
AbortWithError(code int, err error) *gin.Error
Error(err error) *gin.Error
Set(key string, value any)
Get(key string) (value any, exists bool)
MustGet(key string) any
GetString(key string) (s string)
GetBool(key string) (b bool)
GetInt(key string) (i int)
GetInt64(key string) (i64 int64)
GetUint(key string) (ui uint)
GetUint64(key string) (ui64 uint64)
GetFloat64(key string) (f64 float64)
GetTime(key string) (t time.Time)
GetDuration(key string) (d time.Duration)
GetStringSlice(key string) (ss []string)
GetStringMap(key string) (sm map[string]any)
GetStringMapString(key string) (sms map[string]string)
GetStringMapStringSlice(key string) (smss map[string][]string)
Param(key string) string
AddParam(key string, value string)
Query(key string) (value string)
DefaultQuery(key string, defaultValue string) string
GetQuery(key string) (string, bool)
QueryArray(key string) (values []string)
initQueryCache()
GetQueryArray(key string) (values []string, ok bool)
QueryMap(key string) (dicts map[string]string)
GetQueryMap(key string) (map[string]string, bool)
PostForm(key string) (value string)
DefaultPostForm(key string, defaultValue string) string
GetPostForm(key string) (string, bool)
PostFormArray(key string) (values []string)
initFormCache()
GetPostFormArray(key string) (values []string, ok bool)
PostFormMap(key string) (dicts map[string]string)
GetPostFormMap(key string) (map[string]string, bool)
get(m map[string][]string, key string) (map[string]string, bool)
FormFile(name string) (*multipart.FileHeader, error)
MultipartForm() (*multipart.Form, error)
SaveUploadedFile(file *multipart.FileHeader, dst string) error
Bind(obj any) error
BindJSON(obj any) error
BindXML(obj any) error
BindQuery(obj any) error
BindYAML(obj any) error
BindTOML(obj any) error
BindHeader(obj any) error
BindUri(obj any) error
MustBindWith(obj any, b binding.Binding) error
ShouldBind(obj any) error
ShouldBindJSON(obj any) error
ShouldBindXML(obj any) error
ShouldBindQuery(obj any) error
ShouldBindYAML(obj any) error
ShouldBindTOML(obj any) error
ShouldBindHeader(obj any) error
ShouldBindUri(obj any) error
ShouldBindWith(obj any, b binding.Binding) error
ShouldBindBodyWith(obj any, bb binding.BindingBody) (err error)
ClientIP() string
RemoteIP() string
ContentType() string
IsWebsocket() bool
requestHeader(key string) string
Status(code int)
Header(key string, value string)
GetHeader(key string) string
GetRawData() ([]byte, error)
SetSameSite(samesite http.SameSite)
SetCookie(name string, value string, maxAge int, path string, domain string, secure bool, httpOnly bool)
Cookie(name string) (string, error)
Render(code int, r render.Render)
HTML(code int, name string, obj any)
IndentedJSON(code int, obj any)
SecureJSON(code int, obj any)
JSONP(code int, obj any)
JSON(code int, obj any)
AsciiJSON(code int, obj any)
PureJSON(code int, obj any)
XML(code int, obj any)
YAML(code int, obj any)
TOML(code int, obj any)
ProtoBuf(code int, obj any)
String(code int, format string, values ...any)
Redirect(code int, location string)
Data(code int, contentType string, data []byte)
DataFromReader(code int, contentLength int64, contentType string, reader io.Reader, extraHeaders map[string]string)
File(filepath string)
FileFromFS(filepath string, fs http.FileSystem)
FileAttachment(filepath string, filename string)
SSEvent(name string, message any)
Stream(step func(w io.Writer) bool) bool
Negotiate(code int, config gin.Negotiate)
NegotiateFormat(offered ...string) string
SetAccepted(formats ...string)
hasRequestContext() bool
Deadline() (deadline time.Time, ok bool)
Done() <-chan struct{}
Err() error
Value(key any) any
```

以上就是 gin.Context的大致结构和功能。


*全文完，感谢阅读。*




