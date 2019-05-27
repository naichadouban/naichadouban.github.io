---
title: go 变量声明初始化、new、make
date: 2019-01-07
toc: true
tags: ["golang","variable"]
categories: ["golang"]
---

# 变量的声明和初始化
## 实验一
起源于大远问，下面的代码会输出什么？
```golang
type Person struct {
	name string
	age int
}

func main() {
	var p Person
	fmt.Println(p.age)
}
```
按照我之前的理解，报错。因为p只是声明了，并没有初始化。
但是打印出来却是`0`

说明Person对象确实初始化了。

## 实验二
```golang
func main() {
	var p *Person
	fmt.Println(p) //<nil>
	fmt.Println(p.age) //panic: runtime error: invalid memory address or nil pointer dereference
}
```
> 声明一个变量，初始化的内容只跟变量的类型相关

声明了一个`*Person`类型的指针p,说明`p`初始化的内容就是指针的默认值，那就是`nil` 了。
打印p.age出错，证明了Person并没有初始化，当然是空指针错误了。

我们可以画图说明下两者的关系
![微信图片_20190106203742.jpg](https://upload-images.jianshu.io/upload_images/422094-61bd4e0fd3823b06.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 总结
1. nil 只能赋值给指针类型的变量，实际上nil就是是指针类型变量的零值。值类型的变量有各自的零值 比如 int 是 0 string 是 ""
2. 变量的声明，无论是值类型还是指针类型的变量，声明之后，变量都会占有一块内存，并且被初始化为一个零值，被初始化的内容只跟变量的类型有关（注意：`Post`跟 `*Post是两种类型`
var i int // 值类型 因此 i 是 0
var p Person // 值类型 因此 p.title 是"", p.num 是 0
var po *Person // 初始化的不是Post类型，而是*Post类型，即指针类型，因此p是nil p.title 会报错  

# make和new
new 和 make 都可以用来分配空间，初始化类型。他们和上面有什么关系吗？
## new(T) 返回的是 T 的指针
```golang
func main() {
	a := new(int)
	*a = 3
	fmt.Printf("%T,%p,%p,%v\n",a,&a,a,a) // *int,0xc000006028,0xc00000a0a8,0xc00000a0a8

	b := new(Person)
	fmt.Printf("%T,%p,%p,%v\n",b,&b,b,b.age) // *main.Person,0xc000006038,0xc000004460,0

	c := new(Person)
	c = &Person{"xuxiaofeng",26}
	fmt.Printf("%T,%p,%p,%v\n",c,&c,c,c.age) // *main.Person,0xc000006040,0xc0000044a0,26
}
```
画图表示一下
![微信图片_20190106203742.jpg](https://upload-images.jianshu.io/upload_images/422094-78d6fab85d216c4f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第三段中，new就相当于下面这两句话
```golang
var c *Person
c = &Person{"xuxiaofeng",26}
```
如果我们`new(*Person)`呢,道理还是同样的道理，只是多了一层指针，指针的指针，容易绕晕。
```golang
func main() {
	a := new(*Person)
	fmt.Printf("%T.%p,%p\n",a,&a,a)  // **main.Person.0xc000006028,0xc000006030
	//a = Person{}  // error
	//a = &Person{}  // error
	//*a = Person{}  //error
 	*a = &Person{"xuxiaofeng",16}
    fmt.Printf("%T.%p,%p\n",*a,&(*a),*a)  // *main.Person.0xc000006030,0xc000004460
	fmt.Println((**a).age)
}
```
画图表示一下
![微信图片_20190106203742.jpg](https://upload-images.jianshu.io/upload_images/422094-08fab9271d92a434.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

new就是初始化T为默认值，然后返回了*T，就是指向初始化的T的指针
## make 只能用于 slice,map,channel
make基本使用就不做介绍了，只要注意只能是slice,map,channel，这三种类型就可以了。这三种类型都是引用类型
### new slice
我们能否`new([]int)` new一个slice呢？
```golang
func main() {
	a := new([]int)
	//a[0] = 1  //(type *[]int does not support indexing)
	//(*a)[0] = 2  // error: index out of range
	fmt.Println(len(*a),cap(*a)) //0 0
	*a = append(*a,1)
	fmt.Printf("%T\n",a)  //*[]int
	fmt.Println((*a)[0])  //1
}
```
我们看也是可以的，只是给我们返回了长度和容量都是0的slice，而append之后，slice的底层数据已经不是原来的数组了。
### new map
```golang
func main() {
	a := new(map[string]int)
	(*a)["xuxaiofeng"] =26  //panic: assignment to entry in nil map
	fmt.Println(*a)

}
```
### new chan
```golang
func main() {
	a := new(chan int)
	go func() {
		(*a) <- 2
	}()
	t := <-*a
	fmt.Println(t)
}
//fatal error: all goroutines are asleep - deadlock!
//goroutine 1 [chan receive (nil chan)]:
```
说明我们创建的chan就是一个nil，还没有初始化，从一个nil中读数据，程序当然会deadlock

## 总结
new(T) 返回 T 的指针 *T 并指向 T 的零值。
make(T) 返回的初始化的 T，只能用于 slice，map，channel。

# p.name 和*p.name
```golang
func main() {
	p := new(Person)
	//p = Person{"xuxiaofeng",26} //cannot use Person literal (type Person) as type *Person in assignment
	p.name = "xuxiaofeng"
	fmt.Println(p)
}
```
p是*Person,是Person的指针，所以不能直接复制，要想复制也要这样`p = &Person{"xuxiaofeng",26}`,但是为什么调用字段的时候就可以`p.name = "xuxiaofeng"`？
>如果 x 是可寻址的，&x 的 filed 集合包含 m，x.m 和 (&x).m 是等同的，go 自动做转换，也就是 p.name和 (*p).name调用是等价的，go 在下面自动做了转换。


# 接口、结构体实现和结构体指针的实现
```golang
type Person interface {
	Eat()
	Drink()
}

type Student struct {
	age int
	name string
}
func (s Student)Drink(){
	fmt.Println(s.name +"正在喝酒")
}

func (s Student)Eat(){
	fmt.Println(s.name +"正在吃东西")
}
func main(){
	s := Student{30,"xuxioafeng"}
	testEat(s)  // success
	testEat(&s) // success
}
func testEat(p Person){
	p.Eat()
	p.Drink()
}
```
无论我们传的是`Student`本身`s`,还是`Student`对象的指针`&s`,都是可以调用成功了。
但是下面就不一样了
```golang
type Person interface {
	Eat()
	Drink()
}

type Student struct {
	age int
	name string
}
func (s *Student)Eat(){
	fmt.Println(s.name +"正在吃东西")
}
func (s Student)Drink(){
	fmt.Println(s.name +"正在喝酒")
}

func main(){
	s := Student{30,"xuxioafeng"}
	testEat(&s) // success
	testEat(s) // false
}
func testEat(p Person){
	p.Eat()
	p.Drink()
}


```
`func (s *Student)Eat(){}` 我们改成了指针的实现。 
这时候如果传`Student`本身`s`就不可能了。

看错误提示：Cannot use 's' (type Student) as type Person Type does not implement 'Person' as 'Eat' method has a pointer receiver。
就是说我们没有实现接口的`Eat`方法

=======
---
title: go 变量声明初始化、new、make
date: 2019-01-07
tags: ["golang","变量声明和初始化"]

categories: ["golang"]
---

# 变量的声明和初始化
## 实验一
起源于大远问，下面的代码会输出什么？
```golang
type Person struct {
	name string
	age int
}

func main() {
	var p Person
	fmt.Println(p.age)
}
```
按照我之前的理解，报错。因为p只是声明了，并没有初始化。
但是打印出来却是`0`

说明Person对象确实初始化了。

## 实验二
```golang
func main() {
	var p *Person
	fmt.Println(p) //<nil>
	fmt.Println(p.age) //panic: runtime error: invalid memory address or nil pointer dereference
}
```
> 声明一个变量，初始化的内容只跟变量的类型相关

声明了一个`*Person`类型的指针p,说明`p`初始化的内容就是指针的默认值，那就是`nil` 了。
打印p.age出错，证明了Person并没有初始化，当然是空指针错误了。

我们可以画图说明下两者的关系
![微信图片_20190106203742.jpg](https://upload-images.jianshu.io/upload_images/422094-61bd4e0fd3823b06.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 总结
1. nil 只能赋值给指针类型的变量，实际上nil就是是指针类型变量的零值。值类型的变量有各自的零值 比如 int 是 0 string 是 ""
2.  变量的声明，无论是值类型还是指针类型的变量，声明之后，变量都会占有一块内存，并且被初始化为一个零值，被初始化的内容只跟变量的类型有关（注意：`Post`跟 `*Post
是两种类型`
var i int // 值类型 因此 i 是 0
var p Person // 值类型 因此 p.title 是"", p.num 是 0
var po *Person // 初始化的不是Post类型，而是*Post类型，即指针类型，因此p是nil p.title 会报错  

# make和new
new 和 make 都可以用来分配空间，初始化类型。他们和上面有什么关系吗？
## new(T) 返回的是 T 的指针
```golang
func main() {
	a := new(int)
	*a = 3
	fmt.Printf("%T,%p,%p,%v\n",a,&a,a,a) // *int,0xc000006028,0xc00000a0a8,0xc00000a0a8

	b := new(Person)
	fmt.Printf("%T,%p,%p,%v\n",b,&b,b,b.age) // *main.Person,0xc000006038,0xc000004460,0

	c := new(Person)
	c = &Person{"xuxiaofeng",26}
	fmt.Printf("%T,%p,%p,%v\n",c,&c,c,c.age) // *main.Person,0xc000006040,0xc0000044a0,26
}
```
画图表示一下
![微信图片_20190106203742.jpg](https://upload-images.jianshu.io/upload_images/422094-78d6fab85d216c4f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第三段中，new就相当于下面这两句话
```golang
var c *Person
c = &Person{"xuxiaofeng",26}
```
如果我们`new(*Person)`呢,道理还是同样的道理，只是多了一层指针，指针的指针，容易绕晕。
```golang
func main() {
	a := new(*Person)
	fmt.Printf("%T.%p,%p\n",a,&a,a)  // **main.Person.0xc000006028,0xc000006030
	//a = Person{}  // error
	//a = &Person{}  // error
	//*a = Person{}  //error
 	*a = &Person{"xuxiaofeng",16}
    fmt.Printf("%T.%p,%p\n",*a,&(*a),*a)  // *main.Person.0xc000006030,0xc000004460
	fmt.Println((**a).age)
}
```
画图表示一下
![微信图片_20190106203742.jpg](https://upload-images.jianshu.io/upload_images/422094-08fab9271d92a434.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

new就是初始化T为默认值，然后返回了*T，就是指向初始化的T的指针
## make 只能用于 slice,map,channel
make基本使用就不做介绍了，只要注意只能是slice,map,channel，这三种类型就可以了。这三种类型都是引用类型
### new slice
我们能否`new([]int)` new一个slice呢？
```golang
func main() {
	a := new([]int)
	//a[0] = 1  //(type *[]int does not support indexing)
	//(*a)[0] = 2  // error: index out of range
	fmt.Println(len(*a),cap(*a)) //0 0
	*a = append(*a,1)
	fmt.Printf("%T\n",a)  //*[]int
	fmt.Println((*a)[0])  //1
}
```
我们看也是可以的，只是给我们返回了长度和容量都是0的slice，而append之后，slice的底层数据已经不是原来的数组了。
### new map
```golang
func main() {
	a := new(map[string]int)
	(*a)["xuxaiofeng"] =26  //panic: assignment to entry in nil map
	fmt.Println(*a)

}
```
### new chan
```golang
func main() {
	a := new(chan int)
	go func() {
		(*a) <- 2
	}()
	t := <-*a
	fmt.Println(t)
}
//fatal error: all goroutines are asleep - deadlock!
//goroutine 1 [chan receive (nil chan)]:
```
说明我们创建的chan就是一个nil，还没有初始化，从一个nil中读数据，程序当然会deadlock

## 总结
new(T) 返回 T 的指针 *T 并指向 T 的零值。
make(T) 返回的初始化的 T，只能用于 slice，map，channel。

# p.name 和*p.name
```golang
func main() {
	p := new(Person)
	//p = Person{"xuxiaofeng",26} //cannot use Person literal (type Person) as type *Person in assignment
	p.name = "xuxiaofeng"
	fmt.Println(p)
}
```
p是*Person,是Person的指针，所以不能直接复制，要想复制也要这样`p = &Person{"xuxiaofeng",26}`,但是为什么调用字段的时候就可以`p.name = "xuxiaofeng"`？
>如果 x 是可寻址的，&x 的 filed 集合包含 m，x.m 和 (&x).m 是等同的，go 自动做转换，也就是 p.name和 (*p).name调用是等价的，go 在下面自动做了转换。


# 接口、结构体实现和结构体指针的实现
```golang
type Person interface {
	Eat()
	Drink()
}

type Student struct {
	age int
	name string
}
func (s Student)Drink(){
	fmt.Println(s.name +"正在喝酒")
}

func (s Student)Eat(){
	fmt.Println(s.name +"正在吃东西")
}
func main(){
	s := Student{30,"xuxioafeng"}
	testEat(s)  // success
	testEat(&s) // success
}
func testEat(p Person){
	p.Eat()
	p.Drink()
}
```
无论我们传的是`Student`本身`s`,还是`Student`对象的指针`&s`,都是可以调用成功了。
但是下面就不一样了
```golang
type Person interface {
	Eat()
	Drink()
}

type Student struct {
	age int
	name string
}
func (s *Student)Eat(){
	fmt.Println(s.name +"正在吃东西")
}
func (s Student)Drink(){
	fmt.Println(s.name +"正在喝酒")
}

func main(){
	s := Student{30,"xuxioafeng"}
	testEat(&s) // success
	testEat(s) // false
}
func testEat(p Person){
	p.Eat()
	p.Drink()
}


```
`func (s *Student)Eat(){}` 我们改成了指针的实现。 
这时候如果传`Student`本身`s`就不可能了。

看错误提示：Cannot use 's' (type Student) as type Person Type does not implement 'Person' as 'Eat' method has a pointer receiver。
就是说我们没有实现接口的`Eat`方法

>>>>>>> 6da34f0459eccf958c470f5fc458327949c04c6d
> 说明：还是和上面一样，这是golang的解引用。我们传`&s`的时候，先找`&s`，没有的话就会去`s`上找了