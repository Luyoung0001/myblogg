---
title: 【Golang】多线程爬虫的实现
date: 2023-05-05 19:23:51
tags: golang 爬虫 开发语言
categories: Golang
---

<!--more-->

## 〇、前言
Golang 是一种并发友好的语言，使用 goroutines 和 channels 可以轻松地实现多线程爬虫。具体地说，实现的是多协程。协程是一种比线程更轻量化的最小逻辑可运行单位，它不受操作系统调度，由用户调度。因此对于协程并发的控制，有较高的要求。
## 一、goroutine（Go 协程）

Go 协程（Goroutine）是与其他函数同时运行的函数。可以认为 Go 协程是轻量级的线程，由 Go 运行时来管理。在函数调用前加上 go 关键字，这次调用就会在一个新的 goroutine 中并发执行。当被调用的函数返回时，这个 goroutine 也自动结束。

比如：

```go

func son() {
	for {
		fmt.Printf("son says:hello, world!\n")
		time.Sleep(time.Second)
	}
}
func father() {
	go son()
	for {
		fmt.Printf("father says:你好，世界!\n")
		time.Sleep(time.Second)
	}
}
func main() {
	father()

}

```

运行结果：

> father says:你好，世界!
son says:hello, world!
son says:hello, world!
father says:你好，世界!
father says:你好，世界!
son says:hello, world!
.....

在这个例子中，main() 函数里面执行 father()，而 father()中又开启了一个协程 son()，之后两个死循环分别执行。

当然，如果主协程运行结束时子协程还没结束，那么就会被 kill 掉。需要注意的是，如果这个函数有返回值，那么这个返回值会被丢弃。

```go
func main() {
	go loop()
	fmt.Println("hello,world!")
}
func loop() {
	for i := 0; i < 10000; i++ {
		fmt.Println(i)
	}
}
```

运行结果：

> hello,world!
> 0
> 1

 可以看到，子协程刚打印了 0、1 之后 main()函数就结束了，这显然不符合我们的预期。我们有多种方法来解决这个问题。比如在主协程里面 sleep()，或者使用 channel、waitGroup 等。

Go 协程（Goroutine）之间通过信道（channel）进行通信，简单的说就是多个协程之间通信的管道。信道可以防止多个协程访问共享内存时发生资源争抢的问题。Go 中的 channel 是 goroutine 之间的通信机制。这就是为什么我们之前说过 Go 实现并发的方式是：“**不是通过共享内存通信，而是通过通信共享内存。**”

比如：

```go
var (
	myMap = make(map[int]int, 10)
)

// 计算n！并放入到map里
func operation(n int) {
	res := 1
	for i := 1; i <= n; i++ {
		res *= i
	}
	myMap[n] = res
}

func Test3() {
	//我们开启多个协程去完成这个任务
	for i := 1; i <= 200; i++ {
		go operation(i)
	}
	time.Sleep(time.Second * 10)
	fmt.Println(myMap)
}
func main() {
	Test3()
}
```
运行结果：

> fatal error: concurrent map writes
goroutine 42 [running]:

这里产生了一个 fatal error，因为我们创建的 myMap 不支持同时访问，这有点像 Java 里面的非线程安全概念。因此我们需要一种“线程安全”的数据结构，就是 Go 中的channel。

## 二、channel(通道)
channel 分为无缓冲和有缓冲的。无缓冲是同步的，例如make(chan int)，就是一个送信人去你家门口送信，你不在家他不走，你一定要接下信，他才会走，无缓冲保证信能到你手上。 有缓冲是异步的，例如make(chan int, 1)，就是一个送信人去你家仍到你家的信箱，转身就走，除非你的信箱满了，他必须等信箱空下来，有缓冲的保证信能进你家的邮箱。
换句话说，有缓存的channel使用环形数组实现，当缓存未满时，向channel发送消息不会阻塞，当缓存满时，发送操作会阻塞，直到其他goroutine从channel中读取消息；同理，当channel中消息不为空时，读取消息不会阻塞，当channel为空时，读取操作会阻塞，直至其他goroutine向channel发送消息。

```go
// 非缓存channel
ch := make(chan int)
// 缓存channel
bch := make(chan int, 2)

```

channel和map类似，make创建了底层数据结构的引用，当赋值或参数传递时，只是拷贝了一个channel的引用，其指向同一channel对象，与其引用类型一样，channel的空值也为nil。使用==可以对类型相同的channel进行比较，只有指向相同对象或同为nil时，结果为true。
###  2.1  channel 的初始化
channel在使用前，需要初始化，否则永远阻塞。

```go
ch := make(chan int)
ch <- x
y <- ch
```

### 2.2 channel的关闭
golang提供了内置的close函数，对channel进行关闭操作。

```go
// 初始化channel
ch := make(chan int)
// 关闭channel ch
close(ch)
```

关于channel的关闭，需要注意以下事项：
- 关闭未初始化的channle(nil)会panic
- 重复关闭同一channel会panic
- 向以关闭channel发送消息会panic
- 从已关闭channel读取数据，不会panic，若存在数据，则可以读出未被读取的消息，若已被读出，则获取的数据为零值，可以通过ok-idiom的方式，判断channel是否关闭
- channel的关闭操作，会产生广播消息，所有向channel读取消息的goroutine都会接受到消息

## 三、waitGroup 的使用
正常情况下，新激活的goroutine的结束过程是不可控制的，唯一可以保证终止goroutine的行为是main goroutine的终止。
也就是说，我们并不知道哪个goroutine什么时候结束。
但很多情况下，我们正需要知道goroutine是否完成。这需要借助sync包的WaitGroup来实现。
WatiGroup是sync包中的一个struct类型，用来收集需要等待执行完成的goroutine。下面是它的定义：

```go
type WaitGroup struct {
        // Has unexported fields.
}
    A WaitGroup waits for a collection of goroutines to finish. The main
    goroutine calls Add to set the number of goroutines to wait for. Then each
    of the goroutines runs and calls Done when finished. At the same time, Wait
    can be used to block until all goroutines have finished.

A WaitGroup must not be copied after first use.

func (wg *WaitGroup) Add(delta int)
func (wg *WaitGroup) Done()
func (wg *WaitGroup) Wait()
```

waitGroup有三个方法：
- Add()：每次激活想要被等待完成的goroutine之前，先调用Add()，用来设置或添加要等待完成的goroutine数量。
例如Add(2)或者两次调用Add(1)都会设置等待计数器的值为2，表示要等待2个goroutine完成
- Done()：每次需要等待的goroutine在真正完成之前，应该调用该方法来人为表示goroutine完成了，该方法会对等待计数器减1。
- Wait()：在等待计数器减为0之前，Wait()会一直阻塞当前的goroutine
也就是说，Add()用来增加要等待的goroutine的数量，Done()用来表示goroutine已经完成了，减少一次计数器，Wait()用来等待所有需要等待的goroutine完成。

比如：

```go
var wg sync.WaitGroup // 创建同步等待组对象
func main() {
	//设置等待组中，要执行的goroutine的数量
	wg.Add(2)
	go fun1()
	go fun2()
	fmt.Println("main进入阻塞状态,等待wg中的子goroutine结束")
	wg.Wait() //表示main goroutine进入等待，意味着阻塞
	fmt.Println("main解除阻塞")

}
func fun1() {
	for i := 1; i <= 10; i++ {
		fmt.Println("fun1.i:", i)
	}
	wg.Done() //给wg等待中的执行的goroutine数量减1.同Add(-1)
}
func fun2() {
	defer wg.Done()
	for j := 1; j <= 10; j++ {
		fmt.Println("\tfun2.j,", j)
	}
}
```
运行结果：

> main进入阻塞状态,等待wg中的子goroutine结束 fun1.i: 1
>         fun2.j, 1
>         fun2.j, 2
>         fun2.j, 3
>         fun2.j, 4
>         fun2.j, 5
>         fun1.i: 2
>         fun1.i: 3 
>         fun1.i: 4 
>         ... 
>         main解除阻塞
>         
可以看到起到了很好的控制效果。
如果用第一个例子来说明，效果更好：

```go
func main() {
	var wg sync.WaitGroup // 创建同步等待组对象
	wg.Add(1)
	go loop(&wg)
	wg.Wait()
	fmt.Println("hello,world!")
}
func loop(wg *sync.WaitGroup) {
	defer wg.Done()
	for i := 0; i < 100; i++ {
		fmt.Println(i)
	}
}
```
运行结果：

> 0
> 1
> ...
> 99
hello,world!

## 四、爬虫
爬虫的功能是爬取豆瓣top250的电影的数据，并将爬到的数据永久花存储。
思路是首先爬取所有的链接，这个链接的提取通过10 个并行的goroutine处理，然后存储到 channel 中。然后立即创建 250个 goroutine，每一个协程分别爬取一个链接。 再将爬到的数据存储到本地。
### 4.1 爬虫配置

```go
type SpiderConfig struct {
	InitialURL   string // 初始 URL
	MaxDepth     int    // 最大深度
	MaxGoroutine int    // 最大并发数
}
```
### 4.2 爬虫数据

```go
type SpiderData struct {
	URL       string // 链接
	FilmName  string // 电影名
	Director  string // 导员
	Actors    Actor  // 演员列表
	Year      string // 年份
	Score     string // 评分
	Introduce string // 简介
}
type Actor struct {
	actor1 string
	actor2 string
	actor3 string
	actor4 string
	actor5 string
	actor6 string
}
```
### 4.3 开启并行

```go
func spider(config SpiderConfig, chLinks chan string, wg *sync.WaitGroup) {

	for i := 0; i < 10; i++ {
		fmt.Println("正在爬取第", i, "个信息")
		go Spider(strconv.Itoa(i*25), chLinks, wg)
	}

}
```
### 4.4 爬取某个链接

```go
func Spider(page string, chLinks chan string, wg *sync.WaitGroup) {

	// client
	client := http.Client{}
	URL := "https://movie.douban.com/top250?start=" + page + "&filter="
	req, err := http.NewRequest("GET", URL, nil)
	if err != nil {
		fmt.Println("req err", err)
	}
	// UA伪造，明显比 python复杂得多
	req.Header.Set("Connection", "keep-alive")
	req.Header.Set("Cache-Control", "max-age=0")
	req.Header.Set("sec-ch-ua-mobile", "?0")
	req.Header.Set("Upgrade-Insecure-Requests", "1")
	req.Header.Set("User-Agent", "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36")
	req.Header.Set("Accept", "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9")
	req.Header.Set("Sec-Fetch-Site", "same-origin")
	req.Header.Set("Sec-Fetch-Mode", "navigate")
	req.Header.Set("Sec-Fetch-User", "?1")
	req.Header.Set("Sec-Fetch-Dest", "document")
	req.Header.Set("Referer", "https://movie.douban.com/chart")
	req.Header.Set("Accept-Language", "zh-CN,zh;q=0.9")

	resp, err := client.Do(req)
	if err != nil {
		fmt.Println("resp err", err)
	}
	defer func(Body io.ReadCloser) {
		err := Body.Close()
		if err != nil {
			fmt.Println("close err", err)
		}
	}(resp.Body)
	// 网页解析
	docDetail, err := goquery.NewDocumentFromReader(resp.Body)

	if err != nil {
		fmt.Println("解析失败！", err)
	}
	// 选择器
	// #content > div > div.article > ol > li:nth-child(1) > div > div.info > div.hd > a
	title := docDetail.Find("#content > div > div.article > ol > li").
		Each(func(i int, s *goquery.Selection) { // 继续找
			link := s.Find("div > div.pic > a")
			linkTemp, OK := link.Attr("href")
			if OK {
				chLinks <- linkTemp
			}
		})
	title.Text()
	wg.Done()
}
```

### 4.5 爬取某个链接的电影数据

```go
func crawl(url string, wg *sync.WaitGroup) {
	client := http.Client{}
	URL := url
	req, err := http.NewRequest("GET", URL, nil)
	if err != nil {
		fmt.Println("req err", err)
	}
	// UA伪造，明显比 python复杂得多
	req.Header.Set("Connection", "keep-alive")
	req.Header.Set("Cache-Control", "max-age=0")
	req.Header.Set("sec-ch-ua-mobile", "?0")
	req.Header.Set("Upgrade-Insecure-Requests", "1")
	req.Header.Set("User-Agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36")
	req.Header.Set("Accept", "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9")
	req.Header.Set("Sec-Fetch-Site", "same-origin")
	req.Header.Set("Sec-Fetch-Mode", "navigate")
	req.Header.Set("Sec-Fetch-User", "?1")
	req.Header.Set("Sec-Fetch-Dest", "document")
	req.Header.Set("Referer", "https://movie.douban.com/chart")
	req.Header.Set("Accept-Language", "zh-CN,zh;q=0.9")

	resp, err := client.Do(req)

	if err != nil {
		fmt.Println("resp err", err)
	}
	defer func(Body io.ReadCloser) {
		err := Body.Close()
		if err != nil {

		}
	}(resp.Body)
	// 网页解析
	docDatail, err := goquery.NewDocumentFromReader(resp.Body)

	if err != nil {
		fmt.Println("解析失败！", err)
	}
	var data SpiderData
	// 选择器

	// #content > h1 > span:nth-child(1)	movie_name
	movie_name := docDatail.Find("#content > h1 > span:nth-child(1)").Text()
	// #info > span:nth-child(1) > span.attrs > a	director
	director := docDatail.Find("#info > span:nth-child(1) > span.attrs > a").Text()
	// #info > span.actor > span.attrs > span:nth-child(1) > a
	actor01, OK1 := docDatail.Find("#info > span.actor > span.attrs > span:nth-child(1) > a").Attr("hel")
	if OK1 {

	}
	actor02 := docDatail.Find("#info > span.actor > span.attrs > span:nth-child(2) > a").Text()
	actor03 := docDatail.Find("#info > span.actor > span.attrs > span:nth-child(3) > a").Text()
	actor04 := docDatail.Find("#info > span.actor > span.attrs > span:nth-child(4) > a").Text()
	actor05 := docDatail.Find("#info > span.actor > span.attrs > span:nth-child(5) > a").Text()
	actor06 := docDatail.Find("#info > span.actor > span.attrs > span:nth-child(6) > a").Text()

	// #content > h1 > span.year
	year := docDatail.Find("#content > h1 > span.year").Text()
	// #interest_sectl > div.rating_wrap.clearbox > div.rating_self.clearfix > strong
	score := docDatail.Find("#interest_sectl > div.rating_wrap.clearbox > div.rating_self.clearfix > strong").Text()
	//#link-report-intra > span.all.hidden
	introduce := docDatail.Find("#link-report-intra > span.all.hidden").Text()
	data.URL = URL
	data.FilmName = movie_name
	data.Director = director
	data.Actors.actor1 = actor01
	data.Actors.actor2 = actor02
	data.Actors.actor3 = actor03
	data.Actors.actor4 = actor04
	data.Actors.actor5 = actor05
	data.Actors.actor6 = actor06
	data.Year = year
	data.Score = score
	data.Introduce = introduce
	result := data2string(data)
	filename := strconv.Itoa(rand.Int()) + ".txt"
	f, err := os.Create(filename)

	if err != nil {
		fmt.Println(err)
	}
	_, err = f.Write([]byte(result))
	if err != nil {
		return
	}
	err = f.Close()
	if err != nil {
		return
	}
	defer wg.Done()

}
func data2string(data SpiderData) string {
	result := data.FilmName + data.Score + data.Director + data.Year + data.Introduce
	return result
}
```
### 4.6 main 函数开启爬虫

```go
func main() {
	// 先爬取初始URL,将爬来的 links放在 chan中
	// 定义一个 chLinks,这里面将会放置 250 个links
	chLinks := make(chan string, 1000) // 有缓冲的 chan
	config := SpiderConfig{
		InitialURL:   "https://movie.douban.com/top250",
		MaxDepth:     1,
		MaxGoroutine: 10,
	}
	wg := sync.WaitGroup{}
	wg.Add(10)
	spider(config, chLinks, &wg) // 主线程,并发爬取所有的 href
	wg.Wait()
	//for i := 0; i < 250; i++ {
	//	fmt.Println(i, <-chLinks)
	//}
	// 爬完了所有的链接,250 个链接放在 chLinks
	//建立250 个协程来爬每个链接
	wg.Add(250)
	for i := 0; i < 250; i++ {
		go crawl(<-chLinks, &wg)
	}
	wg.Wait()

}
```
爬取结果：
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/ce4dae7e22114a1e950a37a9d8001568_1720253831933.png?token=ANB4BCOYU3QPKG5QPU4S4SLGRD64K)


## 五、总结
本文实现了一个普通的多线（协）程爬虫，用来爬去某些数据。缺点是并没有用到并发深度的功能，因为爬取的数据结构不一样，因此本尝试并不是一个很好的练手项目。
还可以改进的是可以在爬到连接之后，立即对该链接进行开启协程爬取，本文是爬完之后才开始的。
















