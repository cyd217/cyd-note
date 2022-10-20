
---
title: golang-常量
tags:
  - golang 
categories:
	- golang
description: 
toc: true
toc_number: true
cover:  /img/post/6833939bly1giph4fomxoj20zk0m8axp.jpg
---

## const
常量：常量标识恒定不变的值，区别于变量var。var-变量不赋值存在默认值，但是常量声明是必须显示赋值。
常量关键字：const
常量不能使用 “:=(**在函数内部， 声明并初始化变量)**” 语法定义。

**定义单个常量**
```go
const a = 1000
const b float64 = 1.0
```
**批量定义常量**

```go
const (
   statusOk = 200
   notFound = 404
   serverError = 500
)
```

## 语法糖
**批量声明常量时，如果某一行没有写=，那么就和上一行一致**

```go
const (
   n1 = 100      //100
   n2 			 //100
   n3 			 //100
   d, e = 100, 200  //100 200
   a, b         //100 200
)
```

## iota
1.在const关键字出现时将被重置为0；
2.const中每增加**一行**常量声明，将使 iota 计数一次
3.iota作用于itoa使用前的最近的const，如果itoa后续后又出现const

```go
const (
	a1 = iota //0
	a2
	a3
)

const (
	b1 = iota //0
	b2
	b3
)

func main() {
	fmt.Println(a1)  //0
	fmt.Println(a2)  //1
	fmt.Println(a3)  //2

	fmt.Println(b1)  //0
	fmt.Println(b2)	 //1
	fmt.Println(b3)  //2
}
```

### iota特殊用法

使用 ‘_’忽略某一生成的计数器值
```go
const (
	a0 = 100
	a1 = iota //1
	a2        //2
	_         //3
	a3        //4
)
```

itoa插队(个人理解为覆盖掉itoa的计数值，计数器还在继续)

```go
const (
	a1 = iota //0
	a2        //1
	a0 = 100
	_         //3
	a3        //100
	a4 = iota //5
)
```
分析：a3的值为100,因为itoa计数在a0被赋值100后，后续常量的值以字面值常量100为准，但是itoa计数器还在计数，只是没有赋值给常量，需要显示的通过iota恢复

多常量并列定义
在同一个常量组里，可以再多个常量定义中使用iota，多个常量按照列单独计数。互不干涉。

```go
const (
	aa, a = iota, iota // aa = 0 b=0
	bb, _              //bb = 0, _忽略当前计数 增加
	cc, c              //计数增加
	dd, d = iota, 100  //dd = 3, d 被重新赋值为100，继续计数
	ee, e              // ee = 4, e = 100, 打断计数复制动作，但是并没有中短计数，
	ff, f = iota, iota //ff = 5, f = 5 显式恢复计数赋值
)
```

##  用法
const和itoa模拟枚举
go语言没有关键字enum，一般是通过一组常量（等差、等比-有规则）来模拟实现枚举类型，如下：

```go
const (
	Su = iota
	Mo
	Tu
	We
	Th
	Fr
	Sa
)

func main() {

	fmt.Println(Su, Mo, Tu, We, Th, Fr, Sa) //0 1 2 3 4 5 6
}
```

## 常量和变量的区别
常量是只读，声明赋值后无法修改，变量可以重复修改其内容值。常量通产会被编译器在预处理阶段直接展开，作为指令数据使用。
常量在运行时不分配存储地址，变量会分配地址
