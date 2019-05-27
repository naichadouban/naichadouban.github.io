---
title: go指针类型转化 unsafe Pointer
date: 2019-01-04
tags: ["golang","unsafe","Pointer"]
categories: ["golang"]
toc: true
---
# golang强类型
golang是一种`强类型`的`静态`语言。
强类型就是说一旦类型定义了，就不能够再改变它的类型。
静态是说，程序在运行前检测类型，而不是像JavaScript和python等动态语言，运行时才检测

> 为了安全考虑，golang时不允许两种`指针类型` 之间相互转化的。

# 指针类型转换
即使有相同的底层类型也是不能转换的
```golang
func main() {
	var i Myint = 2
	var j int =3
	j = i   // cannot use i (type Myint) as type int in assignment
	fmt.Println(j)
}
```
```golang
func main() {
	i := 10
	fmt.Printf("%T:%v,%p,%T\n",i,i,&i,&i)  // int:10,0x416020,*in

	var j float64 = 3.434
	fmt.Printf("%T:%v,%p,%T\n",j,j,&j,&j)  // float64:3.434,0x416038,*float64
	
	// 类型转化
	m := float64(i)
	fmt.Printf("%T:%v,%p,%T\n",m,m,&m,&m) // float64:10,0x416060,*float64
	// 指针类型转化
	n := (*float64)(&i)  //cannot convert &i (type *int) to type *float64
	fmt.Printf("%T:%v,%p,%T\n",n,n,&n,&n)
}
```
程序中我们需明白
>1. `*int`，表示int类型的指针，`*float` 表示float类型的指针
2. `float64(i)`就是普通的类型转化，golang时允许的
3. `(*float64)(&i)` 中 `(*float64)` 括号中整体是一个类型，表示float64类型的指针类型

通过上面的例子，我们知道不同的指针类型是不同转换的，但是我们就需要转化怎么办？
就需要用到unsafe包的 Pointer了

# unsafe.Pointer
## 介绍
unsafe.Pointer是一种特殊类型的指针，它可以包含任意类型的地址
我们可以在golang的源码中看到如下的定义
```golang
type ArbitraryType int
type Pointer *ArbitraryType
```
说明unsafe.Pointer其实就是`*int` ，一个通用类型的指针
示例一
```golang
func main() {
	i := 10
	fmt.Printf("%T,%v\n",&i,&i)  // *int,0xc00004c080
	var j unsafe.Pointer =  unsafe.Pointer(&i) // unsafe.Pointer,0xc00004c080
	fmt.Printf("%T,%v",j,j)
}
```
如果我们直接` var j unsafe.Pointer =  &i` 的话，就会报错：cannot use &i (type *int) as type unsafe.Pointer in assignment。说民unsafe.Pointer是一种类型，上面的示例只是类型强转。
## 利用unsafe.Pointer再不同`*T`之间转换
```golang
func main() {
	var i int = 10
	var j *float64 = (*float64)(unsafe.Pointer(&i))
	fmt.Printf("%T.%T\n",j,*j) // *float64.float64
	*j = *j * 3
	fmt.Printf("%v,%v",i,j) //30,0xc00004c080
}
```
上面的代码中，我们通过操作`j`指针就改变了`i`的值。
虽然毫无意义，但是证明了通过`unsafe.Pointer`，我们可以将`*int`转化成`*float64`
## unsafe.Pointer 四原则
> 1.任何指针都可以转化为unsafe.Pointer
2.unsafe.Pointer 可以转化为任何指针
3.uintptr可以转化为unsafe.Pointer
4.unsafe.Pointer可以转化为uintptr

规则1,2我们前面已经演示过了，3,4是干什么的？
`*T` 不能计算偏移量，也不能计算便宜。但是uintptr可以。所以，涉及到指针运算的时候，我们可以转化为`uintptr`计算，之后再转化回去。利用`uintprt` 我们可以访问特定的内存，可以达到对特定内存的读写

## uintptr
写一个uintptr直接操作内存的代码
```golang
import (
	"fmt"
	"unsafe"
)

type user struct {
	name string
	age int64
}

func main() {
	u := new(user)
	fmt.Printf("%T\n",u)
	pname := (*string)(unsafe.Pointer(u))
	*pname = "xuxiaofeng"

	page := (*int64)(unsafe.Pointer(uintptr(unsafe.Pointer(u))+unsafe.Offsetof(u.age)))
	*page = 26
	// print u
	fmt.Println(u)  //&{xuxiaofeng 26}
}
```
`name`值是user的第一个字段，不用偏移
`age` 是第二个字段，需要便宜。就需要用到uintptr和unsafe.Pointer

**注意事项**
> 如若改成下面的代码，虽然从逻辑上是对的
但是这里会牵涉到GC，如果我们的这些临时变量被GC，那么导致的内存操作就错了，我们最终操作的，就不知道是哪块内存了，会引起莫名其妙的问题。
```golang
    temp:=uintptr(unsafe.Pointer(u))+unsafe.Offsetof(u.age)
    pAge:=(*int)(unsafe.Pointer(temp))
    *pAge = 20
```