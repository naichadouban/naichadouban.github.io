---
title: golang byte()
date: 2019-05-27T16:43:01+08:00
lastmod: 2019-05-27T16:43:01+08:00
draft: false
keywords: []
description: ""
tags: []
categories: []
author: "奶茶豆瓣"

---

<!--more-->

# 问题
看比特币脚本的时候看到这样一行代码
```
b.script = append(b.script, byte((OP_1-1)+val))
```
其中：`OP_1=0x51` `val=1`,`b.script`是`[]byte`类型

其实上面的代码就可以简写成这样：

```
b.script = append(b.script, byte(80))
```

# 疑惑

既然是byte数据添加，那直接`b.script = append(b.script, 80)` 就可以了，干嘛还要加个`byte` ?

# 实验

```
func main() {
	s:= make([]byte,10)
	s = append(s,byte(800))
	fmt.Println(s)
}
```
如果我们在编辑器中打出带`byte()`的代码，编辑器就可以直接报错。
运行报错：`constant 80000 overflows byte`

但是如果不加`byte`

```
func main() {
	s:= make([]byte,10)
	s = append(s,800)
	fmt.Println(s)
}
```

编辑器没有报错，但是编译时也会报越界错误。

？ 难道加`byte()`就是让代码看上去更严谨易读吗？