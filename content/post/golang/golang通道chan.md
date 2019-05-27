---
title: "golang 通道chan"
date: 2019-05-26T01:37:56+08:00
draft: false
comments: true
tags: ["golang","chan","goroutine"]
categories: ["golang"]
author: "徐晓峰"
toc: true

---

# 多协程之通道关闭clone()

用`clone(chan)`关闭通道的话，用此通道接受的`goroutine`都可以收到消息。

如果只是往通道中发个消息，只有一个接受者可以接收到。

```golang
func main() {
	wg := sync.WaitGroup{}
	quitCh := make(chan int, 1)
	wg.Add(2)
	go func() {
	out:
		for {
			select {
			case <-quitCh:
				fmt.Print("goroutine 1 收到了退出请求")
				break out
			default:

			}
		}
		wg.Done()

	}()

	go func() {
	out:
		for {
			select {
			case <-quitCh:
				fmt.Print("goroutine 2 收到了退出请求")
				break out
			default:

			}
		}
		wg.Done()

	}()
	time.Sleep(time.Second * 3)
	close(quitCh)
	// quitCh <- 3
	wg.Wait()
}
```

在主协程中，我们`clone(quitCh)`,然后子协程中收到消息后退出。

如果我们不用close(quitCh)，而是往quitCh中发送一个信息`quitCh<-3`，那么两个协程只会有一个收到。
就达不到我们控制子协程退出的目的。

# close(ch)之后会收到什么？

```golang
func main() {
	wg := sync.WaitGroup{}
	quitCh := make(chan int)
	wg.Add(1)
	go func() {
		msg, ok := <-quitCh

		fmt.Println(msg, ok) // 0 false
		wg.Done()
	}()
	time.Sleep(time.Second)
	close(quitCh)
	wg.Wait()

}
```

我们这种情况是无缓冲通道，关闭后读到的数据永远都是0和false

事实上，无论通道中有无数据。都是这样。

# 无缓冲通道和缓冲为1的区别？

c1:=make(chan int)        无缓冲

c2:=make(chan int,1)      有缓冲

c1<-1                            

无缓冲的 不仅仅是 向 c1 通道放 1 而是 一直要有别的携程 <-c1 接手了 这个参数，那么c1<-1才会继续下去，要不然就一直阻塞着

而 c2<-1 则不会阻塞，因为缓冲大小是1 只有当 放第二个值的时候 第一个还没被人拿走，这时候才会阻塞。

打个比喻

无缓冲的  就是一个送信人去你家门口送信 ，你不在家 他不走，你一定要接下信，他才会走。

无缓冲保证信能到你手上

有缓冲的 就是一个送信人去你家仍到你家的信箱 转身就走 ，除非你的信箱满了 他必须等信箱空下来。

有缓冲的 保证 信能进你家的邮箱









