---
title: "Go channel 理解"
date: 2019-08-09 15:56:10
tags: [go,channel]
categories: [golang] 
---

[TOC]

### channel

- 为什么需要channel

> 1)使用全局变量加锁同步来解决go routine的通讯，但不完美
> 
> 2)主线程在等待所有goroutine全部完成的时间很难确定。
> 
> 3)如果主线程休眠时间长了，会加长等待时间，如果等待时间短了，可能还有goroutine处于工作状态，这时也会随主线程的退出而销毁
> 
> 4)通过全局变量加锁同步来实现通讯，也并不利用多个协程对全局变量的读写操作。
> 
> 5)上面种种分析都在呼唤一个新的通讯机制-channe

- channel 基本介绍

> 1)channle本质就是一个数据结构-队列
> 2)数据是先进先出【FIFO:firstinfirstout】
> 3)线程安全，多goroutine访问时，不需要加锁，就是说channel本身就是线程安全的
> 4)channel有类型的，一个string的channel只能存放string类型数据。

- channel的注意点~

>使用内置函数close可以关闭channel，不能再写入数据，只能读取数据.
在遍历channel时,如果channel没有关闭,则会有deadlock错误.
如果channel已经关闭,则正常遍历,遍历完数组就退出.

#### channel简单的实现

要求写在注释里面~

```
package main

import "fmt"

// 开启一个writeData协程,向管道intChan写入50个整数
// 开启一个readData协程,向管道intChan取出writeData写入的数据
// 两个协程操作一个管道,readData这个协程读完数据,给一个信号让主函数退出.
// 主线程需要等待两个协程完成才能退出管道

func WriteData(intChan chan int) {
	for i := 1; i <= 50; i++ {
		intChan <- i
		fmt.Println("WriteData write=",i)
	}
	close(intChan)
}

func ReadData(intChan chan int, ExitChan chan bool) {
	for {
		v, ok := <-intChan
		//读取intChan里面的数据,读完就break
		if !ok {
			break
		}
		fmt.Printf("ReadData read=%v\n", v)
	}
	//读完后给主函数一个退出信号,并关闭这个channel
	ExitChan <- true
	close(ExitChan)
}

func main() {
	intChan := make(chan int,50)
	ExitChan := make(chan bool,1)
	go WriteData(intChan)
	go ReadData(intChan,ExitChan)

	for {
		//读取ExitChan里面的数据,读完就break
		_,ok := <-ExitChan
		if !ok{
			break
		}
	}
}
```

### 使用channel提高的效率对比

开启channel可以启用go语言天然的高并发,高并发的有点可以写一个例子体验

需求: 需求：要求统计1-20000000的数字中，哪些是素数？

- 普通的for循坏方式

```
package main

import (
	"fmt"
	"math"
	"time"
)
func IsPrime(n int64) bool {
	if n < 2 {
		return false
	}
	for i := 2; i <= int(math.Sqrt(float64(n))); i++ {
		if int(n)%i == 0 {
			return false
		}
	}
	return true
}
func found() {
	start := time.Now().UnixNano()
	var i int64
	for i = 2; i <= 20000000; i++ {
		if IsPrime(i) {
			//fmt.Println(i)
		}
	}
	end := time.Now().UnixNano()
	fmt.Printf("普通方法耗时 =%v\n", end-start)
}
func main() {
	found()
}


```

- go 协程处理

```
package main

import (
	"fmt"
	"math"
	"time"
)

func IsPrime(n int64) bool {
	if n < 2 {
		return false
	}
	for i := 2; i <= int(math.Sqrt(float64(n))); i++ {
		if int(n)%i == 0 {
			return false
		}
	}
	return true
}


func putNum(intChan chan int) {
	for i := 1; i <= 20000000; i++ {
		intChan <- i
	}
	close(intChan) //放完数据即可关闭
}

func primeNum(intChan chan int, primeChan chan int, exitChan chan bool) {

	for {
		num, ok := <-intChan
		if !ok {
			break
		}

		if IsPrime(int64(num)) {
			primeChan <- num
		}
	}
	// fmt.Println("没有数据退出")
	// 不能关闭primeChan,多协程可能导致安全问题
	exitChan <- true
}

func main() {
	start := time.Now().UnixNano()
	intChan := make(chan int, 1000)
	primChan := make(chan int, 10000000) //存入结果
	exitChan := make(chan bool, 4)

	go putNum(intChan)
	//起四个线程读取
	for i := 0; i < 4; i++ {
		go primeNum(intChan, primChan, exitChan)
	}
	// 起一个协程,阻塞统计是否统计完
	go func() {
		// 从exit取出4个结果,即可关闭primeChan
		for i := 0; i < 4; i++ {
			<-exitChan
		}
		end := time.Now().UnixNano()
		fmt.Println("使用协程耗时 =", end-start)
		close(primChan)

	}()

	for {
		_, ok := <-primChan
		//res,ok := <-primChan
		if !ok {
			break
		}
		//fmt.Printf("prime = %d\n",res)
	}
	fmt.Println("主线程退出")
}
// 每次得到的时间差不多,但是有区别
// 在8Vcpu的linux服务器上运行,起了8个goroutine,速度提升5倍.
// # go build goPrime.go 
// # ./goPrime 
// 普通方法耗时 =58192386339
// 使用协程耗时= 11656927279
// 主线程退出
```

计算一个1-20000000内的素数,goroutine速度提升了5倍.且可以使用`top`命令查询运行的时候,CPU在普通的for循环只占用100%,而用了8个goroutine的,CPU却直接飙升至700%.

想要了解更多的channel,可以看看Mooc老师的`logProcess`的那个教学视频[mooc](https://www.imooc.com/learn/982),很不错,利用golang将的高并发特效,这个是跟着老师码下来的源码[点击看源码](https://gogs.wangke.co/go/demoLogprocess/src/master/log_process.go)

- 参考

[mooc](https://www.imooc.com/learn/982)

