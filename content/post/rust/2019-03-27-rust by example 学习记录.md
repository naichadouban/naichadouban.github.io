---
title: rust by example 学习记录
date: 2019-03-27T19:30:00
tags: ["rust"]
categories: ["rust"]
---
文章不是教程，只是记录一些自己可能忘记的代码片段.
中文：
https://rustwiki.org/zh-CN/rust-by-example/
英文：
https://doc.rust-lang.org/stable/rust-by-example/f

> 中文读起来很快，如果有些地方感觉不知所云，看不懂，一定要打开英文看看，看英文就是慢点，但是一段话到底是在讲什么，很透彻。

# 自定义类型
结构体、枚举[使用use，C风格用法，测试实例：链表]、常量

```rust
// 隐藏未使用代码警告的属性
#![#[allow(dead_code)]
```

```rust
// `to_owned()` 从一个字符串 slice 创建一个具有所有权的 `String`。
let dave     = Person::Info { name: "Dave".to_owned(), height: 72 };
```
一个使用enum的例子
```rust
enum Person {
    Engineer,
    Scientist,
    Height(i32),
    Weight(i32),
    Info { name: String, height: i32 },
}

fn inspect(p: Person) {
	// 明确地 `use` 各个名称使他们直接可用而不需要手动加上作用域。
    use Person::{Engineer};
    match p {
    	// 上面使用use了，所以这里可以不用加作用域了
        Engineer => println!("is a engineer"),
        Person::Scientist => println!("is a scientist"),
        Person::Height(i) => println!("has a height of {}.", i),
        Person::Weight(i) => println!("has a weight of {}", i),
        Person::Info { name, height } => {
            println!("{} is {} tall!", name, height);
        }
    }
}

fn main() {
    let person = Person::Height(18);
    let amira = Person::Weight(10);
    let dave = Person::Info {name:"Dive".to_owned(),height:72};
    let rebecca = Person::Scientist;
    let rohan = Person::Engineer;
    inspect(person);
    inspect(amira);
    inspect(dave);
    inspect(rebecca);
    inspect(rohan);
}
```

> enum的一个常见用法就是创建链表

# 变量绑定
可变变量、作用域和隐藏、变量先声明

# 类型转换
rust在基本类型之间没有提供隐式类型转化（implicit type conversion,coercion)(强制类型转化)。但是使用关键字`as`可以显式转化(explict type conversion,casting)

# 函数
## 方法
方法部分一个比较好的实例
```rust
// Pair 含有的资源，两个堆分配的整型
struct Pair(Box<i32>,Box<i32>);

impl Pair {
    // 这个方法“消费”调用者对象的资源
    // self 为 `self:Self`的语法糖
    fn destroy(self){
        // 解构 `self`
        let Pair(first,second)=self;
        println!("destroy Pair({},{})",first,second);
        // `first`和`second`离开作用域后释放
    }
}
fn main() {
    let pair = Pair(Box::new(1),Box::new(2));
    pair.destroy();
    // 报错！前面的 `destroy` 调用“消费了” `pair`
    //pair.destroy();
    // 试一试 ^ 将此行注释去掉
}
```
## 闭包
### 捕获
闭包本身是相当灵活的，可以实现按照所需功能来让闭包运行而不用类型标注。这允许变量捕获灵活地适应使用情况，有时是移动（moving）有时是借用(borrowing)
闭包可以捕获变量：
- 通过引用：&T
- 通过可变引用:&mut T
- 通过值：T

大白话：就是说闭包可以根据自己的使用需要来决定到底使用上面三种方式中的那种。并且优先使用`&T`,不满足时才会使用下面的两种。

```rust
fn main() {
    use std::mem;
    let color = "Green";
    // 闭包打印color，它会借用 &color ，并将该借用和闭包存储到print变量中。
    // 它会一直保持借用状态直到 print 离开作用域。
    // println 只需要引用即可，所以他没有强加更多的限制（后面的两种）
    let print =|| println!("color :{}", color);

    print(); // 调用闭包 Green
    print(); // 调用闭包 Green

    let mut count = 0;
    // 闭包要使count的值增加，可以使用&mut count 或者 count
    // 但是&mut count 限制更少，所以采用它，仅仅借用count[&mut count]即可。

    // inc 前面加 mut ，是因为一个可变的引用 `&mut count`是存储在其内部。
    // 因此，调用该闭包转变成一个需要加`mut`的闭包
    let mut inc = ||{
        count += 1;
        println!("count:{}", count);
    }
    inc(); // count:1
    inc(); // count:2

    // 下面这行是报错的，因为你再次借用了。
    //let reborrow = &mut count;

    // 再实验一下： 不可复制类型（non-copy type）
    let movable = Box::new(3);
    // 因为下面的闭包中，mem::drop()要求使用T，所以这必须通过使用值来实现。
    // 如果是可复制类型，将会复制值到闭包而不会用到原始值，
    // 不可复制类型的话，就必须移动（move），因此变量movable 就被移动到了闭包中。
    let consume = ||{
        println!("movable:{}", movable);
        mem::drop(movable);
    };
    // consume 消费（consume）了该变量，所以这只能调用一次。
    consume() 

    // consume()  报错
}
```

### 作为输入参量

闭包作为输入参数时，闭包的完整类型必须使用以下的其中一种 `trait` 来标注。他们的受限程序依次递减。
- `Fn`：闭包捕获引用(`&T`)
- `FnMut`:闭包捕获可变引用(`&mut T`)
- `FnOnce`:闭包捕获值(T)

假如想作为一个标记为`FnOnce`类型的参数，这意味着闭包可以捕获`&T` ,`&mut T`,`T`三种类型。但是编译器最终会选择哪个，取决于捕获的变量在闭包中如何使用。

This is because if a move is possiable ,then any type of borrow should also be possible.Note that the reverse is not true. if the parameter is annotated as `Fn`,then capturing variables by `&mut T` or `T` are not allowed.

```rust
// 把闭包作为参数并调用它
fn apply<F>(f:F)where
    F:FnOnce(){
    f();
}
// 把闭包作为参数，返回i32类型的值
fn apply_to_3<F>(f:F) ->i32 where F:Fn(i32)->i32{
    f(3)
}

fn main() {
    use std::mem;
    let greeting = "hello";
    // 不可复制的类型
    // `to_owned`从借用的数据中创建了自己有所有权的数据
    let mut farewell = "goodbye".to_owned();

    // 捕获两个变量：greeting 通过引用，farewell通过值。
    let diary = ||{
        // greeting 通过引用：需要 `Fn` trait
        println!("i said {}", greeting);
        // 迫使 farewell 以可变引用的方式被捕获，这种情况需要 `FnMut` trait
        farewell.push_str("!!!");
        println!("then i screamed {}", farewell);
        println!("now i can sleep");

        // 手动调用drop,迫使farewell以值的方式被捕获。这种情况需要 `FnOnce` trait
        mem::drop(farewell);

    };
    // 调用处理闭包的函数
    apply(diary);

    let double = |x| 2*x;
    println!("2 double:{}", apply_to_3(double));

}
```
上面的例子中，如果我们将`apply` 函数改变成下面的类型
```rust
// 把闭包作为参数并调用它
fn apply<F>(f:F)where
    F:FnMut(){
    f();
}
```
程序就不行了，因为声明`FnMut`类型已经限定了传入的闭包只能捕获`T` 和 `&T` 的类型，而在闭包中` mem::drop(farewell);` 是需要捕获值的类型的。所以报错。

### 闭包作为输出参量

# 模块
## 文件分层
```
$ tree .
.
|-- my
|   |-- inaccessible.rs
|   |-- mod.rs
|   `-- nested.rs
`-- split.rs
```
在split.rs中
```rust
// 此声明将会查找名为 `my.rs` 或 `my/mod.rs` 的文件，并将该文件的内容插入到
// 此作用域名为 `my` 的模块里面。
mod my;

fn function() {
    println!("called `function()`");
}
```

# crate 
如果crate中有mod声明，那么模块文件的内容将会在运行编译之前与crate文件合并。换句话说，模块不会单独进行编译，只有crate进行编译。

# 属性
## crate
`crate_type` 属性可以告知编译器 `crate `是一个二进制的可执行文件还是一个库（甚至是哪种类型的库），crate_time 属性可以设定 crate 的名称。
```rust
// 这个 crate 是一个库文件
#![crate_type = "lib"]
// 库的名称为 “rary”
#![crate_name = "rary"]
```
## cfg
例如控制代码在哪个系统上执行
```rust
// 这个函数仅当操作系统是 Linux 的时候才会编译
#[cfg(target_os = "linux")]
fn are_you_on_linux() {
    println!("You are running linux!")
}
```

# 泛型
## 函数

## 实现 implementation

## 特性 trait


# 作用域规则

## 生命周期
### 显式标注
编译器（有时也称为借用检查器）使用它来确保所有的借用都是有效的。
`foo<'a,'b>`

在这种情况下，foo的生命周期不能超出'a或'b的周期。
