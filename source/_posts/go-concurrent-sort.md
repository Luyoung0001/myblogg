---
title: "Go 并发项目：多线程排序实现"
date: 2023-05-08 13:03:13
tags:
  - "Go"
  - "goroutine"
  - "sorting"
categories:
  - "Programming Projects"
---

<!--more-->

## 〇、前言
本案例实现一个多线程排序算法，能够对给定的整数数组进行排序，使用 goroutines 对其进行并发化优化。
## 一、随机数生成器

```go
func randProduce(randNums chan []int, wg *sync.WaitGroup) {
	for i := 0; i < 100; i++ {
		go rand1(randNums, wg)
	}
}
func rand1(randNums chan []int, wg *sync.WaitGroup) {

	r := rand.New(rand.NewSource(time.Now().Unix()))
	int1000 := make([]int, 1000000)
	for i := 0; i < 1000000; i++ {
		int1000[i] = r.Intn(1000000)
	}
	randNums <- int1000
	wg.Done()
}
```
## 二、使用 goroutines 并发地对各个子数组进行排序

```go
func sort0(randNums chan []int, sortNums chan []int, wg *sync.WaitGroup) {
	for i := 0; i < 100; i++ {
		go sort2(randNums, sortNums, wg)
	}
}
func sort2(randNums chan []int, sortNums chan []int, wg *sync.WaitGroup) {
	int1000_Old := <-randNums
	sort.Ints(int1000_Old)
	sortNums <- int1000_Old
	wg.Done()
}
```

## 三、合并已排序的子数组，得到最终排序结果

```go
func mergeAll(sortNums chan []int, wg *sync.WaitGroup) []int {
	defer wg.Done()
	temp2 := <-sortNums
	var temp1 []int
	for i := 1; i <= 99; i++ {
		temp1 = make([]int, 1000000*i+1000000)
		copy(temp1, temp2)
		temp1 = merge(temp1, 1000000*i+1000000, <-sortNums, 1000000)
		temp2 = make([]int, 1000000*i+1000000)
		copy(temp2, temp1)
	}
	return temp2
}

func merge(nums1 []int, m int, nums2 []int, n int) []int {
	temp := make([]int, m)
	copy(temp, nums1)

	t, j := 0, 0 //t为temp的索引，j为nums2的索引
	for i := 0; i < len(nums1); i++ {
		if t >= len(temp) {
			nums1[i] = nums2[j]
			j++
			continue
		}
		if j >= n {
			nums1[i] = temp[t]
			t++
			continue
		}
		if nums2[j] <= temp[t] {
			nums1[i] = nums2[j]
			j++
		} else {
			nums1[i] = temp[t]
			t++
		}
	}
	return nums1
}
```
## 四、main 函数控制流程

```go
func main() {
	fmt.Println("开始运行!")
	start := time.Now() // 获取当前时间
	wg := sync.WaitGroup{}
	wg.Add(201)
	randNums := make(chan []int, 100)
	sortNUms := make(chan []int, 100)

	go randProduce(randNums, &wg)
	go sort0(randNums, sortNUms, &wg)
	go mergeAll(sortNUms, &wg)

	wg.Wait()
	// fmt.Println(l)
	elapsed := time.Since(start)
	fmt.Println("该函数执行完成耗时：", elapsed)
}
```
## 五、思路
本案例采用了两个 channel，分别存储产生的的随机数slice和排好顺序的 slice，每一个 slice大小为 100 万，一共一百个 slice，也就是一亿个数据。
```go
	randNums := make(chan []int, 100)
	sortNUms := make(chan []int, 100)
```
程序一边产生随机数，一边将产生的随机数randNums发送到 sort 函数进行排序，排好顺序后将数据发送到sortNUms。这两个流程可以并行计算，因此：

```go
	go randProduce(randNums, &wg)
	go sort0(randNums, sortNUms, &wg)
```
合并也可以参与到并行计算之中，多加一个信号量就好：

```go
go mergeAll(sortNUms, &wg)
```

运行结果：

> (base) luliang@shenjian Sort % go build SortRoutine.go
(base) luliang@shenjian Sort % ./SortRoutine
开始运行!
该函数执行完成耗时： 50.317081625s

## 六、性能比较
可以写一个单线程的排序，但是数据产生还是多线程的：
```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
	"time"
)

func main() {
	fmt.Println("开始运行!")
	start := time.Now() // 获取当前时间

	randNums := make(chan int, 10000)
	go randProduce1(randNums)
	randNums1 := make([]int, 100000000)
	for i := 0; i < 100000000; i++ {
		randNums1[i] = <-randNums
	}
	sort.Ints(randNums1)
	elapsed := time.Since(start)
	fmt.Println("该函数执行完成耗时：", elapsed)
}
func randProduce1(randNums chan int) {
	for i := 0; i < 10000; i++ {
		go rand2(randNums)
	}
}
func rand2(randNums chan int) {

	r := rand.New(rand.NewSource(time.Now().Unix()))
	for i := 0; i < 10000; i++ {
		randNums <- r.Intn(10000000)
	}
}

```
运行结果为：

> (base) luliang@shenjian Sort % go build SortRoutine1.go
(base) luliang@shenjian Sort % ./SortRoutine1
开始运行!
该函数执行完成耗时： 54.869565792s

可以看到两种方法消耗的时间差不多，这是因为数据量还是太小，多线程生成数据、排序、以及合并开辟了大量的协程，这个会消耗一定的时间。



