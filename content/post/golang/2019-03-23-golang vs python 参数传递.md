---
title: golang vs python 参数传递.md
date: 2019-01-07
tags: ["golang","python"]
categories: ["golang","python"]
---

如果非要用一句话来概括的话，golang都是值传递，python都是引用传递

# golang
在 Golang 中函数之间传递变量时总是以值的方式传递的，无论是 int,string,bool,array 这样的内置类型（或者说原始的类型），还是 slice,channel,map 这样的引用类型，在函数间传递变量时，都是以值的方式传递，也就是说传递的都是值的副本。

简单来说，就看你传递的是什么，
你传递的是int,string,bool,array 这样的值类型，我就把这些值拷贝一份
你传递的是slice,channel,map这样引用类型，我就把这些引用拷贝一份

所有都是值传递了。

需要注意：引用类型和引用传递表达的是不同的概念。

> 所有类型也是对象
引用传递通过指针实现
其他默认是值传递

# python
这是一篇很好的文章
http://winterttr.me/2015/10/24/python-passing-arguments-as-value-or-reference/

python中全都是对象引用，有不可变对象（number，string，tuple等）和可变对象之分（list，dict等）只是引用对象是否具备可更改的能力，从而体现出类似于“传值”和“引用”的不同外在表现
你传可变的值到函数，就相当于是引用传递了，因为函数中对此值再操作，地址也不变。
你传不可变的值到函数，就相当于是值传递了，因为函数中再对此值造作，就需要创造新的对象，变量就要指向新的地址。

```python
a = 1
print id(1)
print id(a)

b = a
print id(b)
```
在python中，上面三个print打印出来的结果都是相同的


