
---
title: go泛型进阶
tags:
  - golang 
categories:
	- golang
description: 
toc: true
toc_number: true
cover:  /img/post/R-C.png
---

## 前言
前面的文章介绍了泛型的基础语法和使用要注意的地方。现在来看看泛型的优化方式。


## 接口作为泛型约束
有时候使用泛型编程时，我们会书写长长的类型约束。

```go
// 一个可以容纳所有int,uint以及浮点类型的泛型切片
type Slice[T int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64 | float32 | float64] []T
```
这个写法，让人头皮发麻。。。。。所以我们尝试将约束单独拿出来定义到接口中。

```go
type IntUintFloat interface {
	int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64 | float32 | float64
}

type Slice[T IntUintFloat] []T

func main() {
	s := make(Slice[int], 0)
	s = append(s, 1)
	fmt.Printf("%v\n", s)
}
```

那么我们还可不可以进行拆分成更小的接口呢？
答案是可以。

```go
package main
type Int interface {
    int | int8 | int16 | int32 | int64
}

type Uint interface {
    uint | uint8 | uint16 | uint32
}

type Float interface {
    float32 | float64
}

// 1
type Slice[T Int | Uint | Float] []T  // 使用 '|' 将多个接口类型组合


type SliceElement interface{
	Int | Uint | Float
}

//2
type Slice2[T SliceElement] []T  // 使用 '|' 将多个接口类型组合
```


## ~ : 指定潜在类型
我们知道go里面`类型定义`产生的是一个新的类型。

```go
type MyString string
```
`MyString` 是新类型 ，`string` 可以被称为` MyString `的潜在类型。潜在类型的含义是，某**个类型在本质上是哪个类型。潜在类型相同的不同类型的值之间是可以进行类型转换的**。golang为了支持泛型可以`潜在类型`,使用`~`符合。

限制：使用 ~ 时有一定的限制：
- ~后面的类型不能为接口
- ~后面的类型必须为基本类型

```go
type Int interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

type Uint interface {
    ~uint | ~uint8 | ~uint16 | ~uint32
}
type Float interface {
    ~float32 | ~float64
}

type Slice[T Int | Uint | Float] []T 

var s Slice[int] // 正确

type MyInt int
var s2 Slice[MyInt]  // MyInt底层类型是int，所以可以用于实例化

type MyMyInt MyInt
var s3 Slice[MyMyInt]  // 正确。MyMyInt 虽然基于 MyInt ，但底层类型也是int，所以也能用于实例化

type MyInt int

//错误的写法
type _ interface {
    ~MyInt   // 错误，~后的类型必须为基本类型
    ~error   // 错误，~后的类型不能为接口
}
```

## 空接口和 any
Go1.18之后的空接口应该这样理解：

> 虽然空接口内没有写入任何的类型，但它代表的是所有类型的集合，而非一个 空集。
> 
> 类型约束中指定 空接口 的意思是指定了一个包含所有类型的类型集，并不是类型约束限定了只能使用 空接口 来做类型形参。

空接口是一个包含了所有类型的类型集，所以我们经常会用到它。于是，Go1.18开始提供了一个和空接口 interface{} 等价的新关键词 any ，用来使代码更简单：

```go
type Slice[T any] []T // 代码等价于 type Slice[T interface{}] []T
```


## 不一样的接口

学习泛型之前的我们对于接口的写法。

```go
//方法集
type Animal interface {
	Name() (string, error)
	Age() (string, error)
}

```

学习泛型之后，我们写接口就变成这样。

```go
//类型集
type Float interface {
    ~float32 | ~float64
}
type Slice[T Float] []T 
```
换一个角度来重新思考上面这个`Animal `接口的话，会发现接口的定义实际上还能这样理解：
> 我们可以把 Animal  接口看成代表了一个 类型的集合，所有实现了 Name() ，Age() 这两个方法的类型都在接口代表的类型集合当中。

那如果这么写呢？

```go
type Animal2 interface {
	~int
	Age2() (int, error)
}

```
**代表 Animal2 代表的类型集满足两个条件**
 -    1.底层类型是 int 底层类型
 -   2.具有 Age2() (int, error) 方法
```go
package main

import "fmt"

type Animal2 interface {
	~int
	Age2() (int, error)
}
type AnimalImpl[T Animal2] struct {
	age T
}

//Animal2 实现
type myInt int

func (c myInt) Age2() (int, error) {
	return 1, nil
}

//没有实现
// type myString string

// func (c myString) Age2() (int, error) {
// 	return 1, nil
// }

func main() {

	var my myInt = 1
	var s1 AnimalImpl[myInt] = AnimalImpl[myInt]{
		age: my,
	}
	fmt.Println(s1)

	/*  报错 myString does not implement Animal2
	var mystring myString = "1"
	var s2 AnimalImpl[myString] = AnimalImpl[myString]{
		age: mystring,
	}
	fmt.Println(s2)
	*/
}

```



- 基本接口(Basic interface):如果接口中只存在方法，那么是基本接口。
- 一般接口(General interface)：接口中存在方法和类型，那么是一般接口。
- **一般接口类型不能用来定义变量，只能用于泛型的类型约束中**。



## 泛型接口

```go
package main

type Int64 interface {
	~int64 | ~uint64
}

type NumberPlayer[T Int64] interface {
	AddCounter() T
	MinusCounter() T
}

type Counter[T Int64] struct {
	t       T
	counter T
}

func (c *Counter[T]) AddCounter() T {
	c.t += c.counter
	return c.t
}

func (c *Counter[T]) MinusCounter() T {
	c.t -= c.counter
	return c.t
}

func NewNumberPlayer[T Int64](t T, counter T) NumberPlayer[T] {
	return &Counter[T]{
		t:       t,
		counter: counter,
	}
}

func main() {
	numbers := NewNumberPlayer(int64(32), int64(2))
	println(numbers.AddCounter())
	println(numbers.MinusCounter())

	numbersAgain := NewNumberPlayer(uint64(32), uint64(9))
	println(numbersAgain.AddCounter())
	println(numbersAgain.MinusCounter())
}
```
泛型类型要使用的话必须传入类型实参实例化才有意义。所以我们来尝试实例化一下这两个接口。


## 总结
- 接口有一般接口和基本接口。
- 一般接口不能用于定义类型遍历，只能用与约束。
- 泛型类型要使用的话必须传入类型实参实例化才有意义。
- 泛型不能与匿名方法或者函数一起使用
- 泛型不能与类型断言起使用。
- 泛型并不取代Go1.18之前用接口+反射实现的动态类型，泛型最适合的场景：当你需要针对不同类型书写同样的逻辑，使用泛型来简化代码是最好的 (比如你想写个队列，写个链表、栈、堆之类的数据结构）
- 泛型的三个重要概念，分别是：
    - 类型参数：泛型的抽象数据类型。
   - 类型约束：确保调用方能够满足接受方的程序诉求。
  - 类型推导：避免明确地写出一些或所有的类型参数。