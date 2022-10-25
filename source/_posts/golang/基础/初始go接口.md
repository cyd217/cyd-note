---
title: 初始go接口
tags:
  - golang 
categories:
	- golang
description: 
toc: true
toc_number: true
cover:  /img/post/6833939bly1giph4fomxoj20zk0m8axp.jpg
---


## 一.其他语言 
其他语言中所提供的接口概念：接口主要作为不同组件之间的契约存在。对契约的实现是强制的（侵入式接口），你必须声明你的确实现了该接口。为了实现一个接口，你需要从该接口继承。
>interface IFoo {
void Bar();
}
// Java文法 // ...
class Foo implements IFoo {
}
// C++文法 // ...
class Foo : public IFoo {
}    



“侵入式”的主要表现在于实现类需要明确声明自己实现了 某个接口。

## 二.go语言
go语言中接口与其他语言的接口也略有不同，是一种非侵入式接口，实现类的时候，只需要关心自己应该提供哪些方法，不用再纠结接口需要拆得多细才 合理。接口由使用方按需定义，而不用事前规划。一个类只需要实现了接口要求的所有函数，我们就说这个类实现了该接口。
```go
type Phone interface {
   call()
}
type Nokia struct {
    name string
}

// 接口的实现是隐式的
func (phone Nokia) call() {
    fmt.Println("我是 Nokia，是一台电话")
}
```
三.go接口实现多态
```go
package main

import (
    "fmt"
    "strconv"
)

// 定义一个接口
type Good interface {
    settleAccount() int
    orderInfo() string
}

type Phone struct {
    name string
    quantity int
    price int
}

func (phone Phone) settleAccount() int {
    return phone.quantity * phone.price
}
func (phone Phone) orderInfo() string{
    return "您要购买" + strconv.Itoa(phone.quantity)+ "个" +
        phone.name + "计：" + strconv.Itoa(phone.settleAccount()) + "元"
}

type FreeGift struct {
    name string
    quantity int
    price int
}

func (gift FreeGift) settleAccount() int {
    return 0
}
func (gift FreeGift) orderInfo() string{
    return "您要购买" + strconv.Itoa(gift.quantity)+ "个" +
        gift.name + "计：" + strconv.Itoa(gift.settleAccount()) + "元"
}

func calculateAllPrice(goods []Good) int {
    var allPrice int
    for _,good := range goods{
        fmt.Println(good.orderInfo())
        allPrice += good.settleAccount()
    }
    return allPrice
}
func main()  {
    iPhone := Phone{
        name:     "iPhone",
        quantity: 1,
        price:    8000,
    }
    earphones := FreeGift{
        name:     "耳机",
        quantity: 1,
        price:    200,
    }

    goods := []Good{iPhone, earphones}
    allPrice := calculateAllPrice(goods)
    fmt.Printf("该订单总共需要支付 %d 元", allPrice)
}
```

## 四.空接口的使用（重点）
### 4.1：定义
空接口没有定义任何方法口，也因此，我们可以说所有类型都至少实现了空接口。
每一个接口都包含两个属性，一个是值，一个是类型。
而对于空接口来说，这两者都是 nil，可以使用 fmt 来验证一下
```go
package main

import (
    "fmt"
)

func main() {
    var i interface{}
    fmt.Printf("type: %T, value: %v", i, i)
}
//type: <nil>, value: <nil>
```

### 4.2空接口使用
第一，通常我们会直接使用 interface{} 作为类型声明一个实例，而这个实例可以承载任意类型的值。
```go
// 声明一个空接口实例
    var i interface{}

    // 存 int 没有问题
    i = 1
    fmt.Println(i)

    // 存字符串也没有问题
    i = "hello"
    fmt.Println(i)

    // 存布尔值也没有问题
    i = false
    fmt.Println(i)
```
第二，如果想让你的函数可以接收任意类型的值 ，也可以使用空接口;
第三，你也定义一个可以接收任意类型的 array、slice、map、strcut，例如这边定义一个切片;
```go
func main() {
    any := make([]interface{}, 5)
    any[0] = 11
    any[1] = "hello world"
    any[2] = []int{11, 22, 33, 44}
    for _, value := range any {
        fmt.Println(value)
    }
}
```

### 4.3空接口几个要注意的坑（我刚学时的错误）
坑1：空接口可以承载任意值，但不代表任意类型就可以承接空接口类型的值
```go
  // 声明a变量, 类型int, 初始值为1
    var a int = 1

    // 声明i变量, 类型为interface{}, 初始值为a, 此时i的值变为1
    var i interface{} = a

    // 声明b变量, 尝试赋值i  报错
    var b int = i
 ```
坑2：：当空接口承载数组和切片后，该对象无法再进行切片
```go
 sli := []int{2, 3, 5, 7, 11, 13}

    var i interface{}
    i = sli
     //报错
    g := i[1:3]
    fmt.Println(g)
```
坑3：当你使用空接口来接收任意类型的参数时，它的静态类型是 interface{}，但动态类型（是 int，string 还是其他类型）我们并不知道，因此需要使用类型断言。

这里还有一点要说明   空接口调用函数时的隐式转换
```go
func myfunc(i interface{})  {

    switch i.(type) {
    case int:
        fmt.Println("参数的类型是 int")
    case string:
        fmt.Println("参数的类型是 string")
    }
}
func main() {
    a := 10
    b := "hello"
    myfunc(a)
    myfunc(b)
如果写在外面  则报错
/*switch a.(type) {
    case int:
        fmt.Println("参数的类型是 int")
    case string:
        fmt.Println("参数的类型是 string")
    }
*/


}
```
1和3是最容易犯问题，唉。。。。。。。。。。。。。

如果写的有错误，可以告诉我，嘿嘿。