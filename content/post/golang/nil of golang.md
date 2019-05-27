---
title: "Nil of golang"
date: 2019-05-26T19:53:33+08:00
draft: false
tags: ["nil","golang"]
categories: ["golang"]
toc: true
---

> golang中的nil到底是什么？

之前一直以为指针的零值就是nil，就 `var i int`时`i`就是`0`一样。

但是我们运行下面的程序
https://play.golang.org/p/ANfHmNm3U8W

```go
func main() {
	var s []string
	fmt.Println(s==nil)
}
```

也是会得到一个 `true`的结果。


https://www.jianshu.com/p/dd80f6be7969

这篇文章已经讲的很好了。`nil`不光是指针的零值。好几种类型的零值都是`nil`