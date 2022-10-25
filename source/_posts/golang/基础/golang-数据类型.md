---
title: golang-数据类型
tags:
  - golang 
categories:
	- golang
description: 
toc: true
toc_number: true
cover:  https://img-blog.csdnimg.cn/b9089782b51645598d522106a4cfdf54.png
---

每一门语言都有自己的数据结构，Go 语言也不例外，总共有两大类，值类型（基础类型、聚合类型）、引用类型。本文简单介绍一下这些类型。
![在这里插入图片描述](https://img-blog.csdnimg.cn/b9089782b51645598d522106a4cfdf54.png)
**聚合类型**
聚合类型的值由内存中的一组变量构成。数组和结构体都是聚合类型，数组和结构体的长度都是固定的。数组中的的元素类型必须都相同，而结构体中的元素可以不同。
**引用类型**
引用是另外一种数据类型，引用都间接指向变量或者状态，通过引用来操作数据会让该数据的全部引用都受影响。


## 值类型和引用类型
值类型：变量直接存储值，**内存通常在栈上分配**，栈在函数调用完会被释放。
引用类型： 变量存储的是一个地址，这个地址存储最终的值。内存通常在堆上分配，通过GC回收。

在 Go 语言中，**函数的参数传递只有值传递**，而且传递的实参都是原始数据的一份拷贝。如果拷贝的内容是值类型的，那么在函数中就无法修改原始数据；如果拷贝的内容是指针（或者可以理解为引用类型 map、chan 等），那么就可以在函数中修改原始数据。


## 类型比较
### 基本类型
比较的两个变量类型必须相等。而且，golang没有隐式类型转换，比较的两个变量必须类型完全一样，类型定义也不行。如果要比较，先做类型转换再比较。
- 类型完全不一样的，不能比较
- 类型再定义关系，不能比较，可以强转比较
- 类型别名关系，可以比较

```go
func main() {

	type A int
	var a int = 1
	var b A = 1
	//fmt.Println(a == b)      //invalid operation: a == b (mismatched types int and A)
	fmt.Println(a == int(b)) //true

	type C = int
	var c C = 1
	fmt.Println(a == c) //true
}
```
### 复合类型的变量比较
复合类型是逐个字段，逐个元素比较的。需要注意的是，array 或者struct中每个元素必须要是可比较的，如果某个array的元素 or struct的成员不能比较(比如是后面介绍的slice，map等)，则此复合类型也不能比较。

**数组类型变量比较**
数组的长度是类型的一部分，如果数组长度不同，无法比较
逐个元素比较类型和值。每个对应元素的比较遵循基本类型变量的比较规则。跟struct一样，如果item是不可比较的类型，则array也不能做比较。
**struct类型变量比较**
逐个成员比较类型和值。每个对应成员的比较遵循基本类型变量的比较规则。

```go
func main() {

	type Student struct {
		Name string
		Age  int
	}

	a := Student{"minping", 30}
	b := Student{"minping", 30}
	fmt.Println(a == b)   //true
	fmt.Println(&a == &b) //false

	type Student2 struct {
		Name string
		Age  int
		Info []string
	}

	a2 := Student2{
		Name: "minping",
		Age:  30,
	}
	b2 := Student2{
		Name: "minping",
		Age:  30,
	}
	fmt.Println(a2 == b2) //invalid operation: a == b (struct containing []string cannot be compared)
}
```
可以看到，struct中有slice这种不可比较的成员时，整个struct都不能做比较，即使没有对slice那个成员赋值(slice默认值为nil)

### 引用类型的变量比较
**普通的变量引用类型&val和channel的比较规则**

引用类型变量存储的是某个变量的内存地址。所以引用类型变量的比较，判断的是这两个引用类型存储的是不是同一个变量。
- 如果是同一个变量，则内存地址肯定也一样，则引用类型变量相等，用"==“结果为true。
- 如果不是同一个变量，则内存地址肯定不一样，"=="结果为false。

**slice，map这种引用类型的比较**
map类型和slice一样，不能比较，只能与nil做比较。

**interface{}类型变量的比较**
接口类型的变量，包含该接口变量存储的值和值的类型两部分组成，分别称为**接口的动态类型和动态值**。**只有动态类型和动态值都相同时，两个接口变量才相同:**

```go
type Person interface {
    getName() string
}

type Student struct {
    Name string
}

type Teacher struct {
    Name string
}

func (s Student) getName() string {
    return s.Name
}

func (t Teacher) getName() string {
    return t.Name
}

func compare(s, t Person) bool {
    return s == t
}

func main() {

    s1 := Student{"minping"}
    s2 := Student{"minping"}
    t := Teacher{"minping"}

    fmt.Println(compare(s1, s2)) //true
    fmt.Println(compare(s1, t))  //false,类型不同
}
```
而且接口的动态类型必须要是可比较的，如果不能比较(比如slice，map)，则运行时会报panic。因为编译器在编译时无法获取接口的动态类型，所以编译能通过，但是运行时直接panic:

**函数类型的比较**
golang的func作为一等公民，也是一种类型，而且不可比较。

### slice和map的特殊比较
**[]byte类型的变量，使用工具包byte提供的函数就可以做比较**

```go
s1 := []byte{'f', 'o', 'o'}
s2 := []byte{'f', 'o', 'o'}
fmt.Println(bytes.Equal(s1, s2)) // true
s2 = []byte{'b', 'a', 'r'}
fmt.Println(bytes.Equal(s1, s2)) // false
s2 = []byte{'f', 'O', 'O'}
fmt.Println(bytes.EqualFold(s1, s2)) // true
s1 = []byte("źdźbło")
s2 = []byte("źdŹbŁO")
fmt.Println(bytes.EqualFold(s1, s2)) // true
s1 = []byte{}
s2 = nil
fmt.Println(bytes.Equal(s1, s2)) // true
```
**使用反射**

```go
m1 := map[string]int{"foo": 1, "bar": 2}
m2 := map[string]int{"foo": 1, "bar": 2}
// fmt.Println(m1 == m2) // map can only be compared to nil
fmt.Println(reflect.DeepEqual(m1, m2)) // true
m2 = map[string]int{"foo": 1, "bar": 3}
fmt.Println(reflect.DeepEqual(m1, m2)) // false
m3 := map[string]interface{}{"foo": [2]int{1,2}}
m4 := map[string]interface{}{"foo": [2]int{1,2}}
fmt.Println(reflect.DeepEqual(m3, m4)) // true
var m5 map[float64]string
fmt.Println(reflect.DeepEqual(m5, nil)) // false
fmt.Println(m5 == nil) // true
```

## 总结
1，复合类型，只有每个元素(成员)可比较，而且类型和值都相等时，两个复合元素才相等
2，slice，map不可比较，但是可以用reflect或者cmp包来比较
3，func作为golnag的一等公民，也是一个类型，也不能比较。
4，引用类型的比较是看指向的是不是同一个变量
5，类型再定义(type A string)不可比较，是两种不同的类型
6，类型别名(type A = string)可比较，是同一种类型。


## 拓展-类型别名与类型定义
**类型别名**
类型别名需要在别名和原类型之间加上赋值符号 = ，使用**类型别名定义的类型与原类型等价**，Go 语言内建的基本类型中就存在两个别名类型。
- byte 是 uint8 的别名类型；
- rune 是 int32 的别名类型；

```go
type MyString = string
```
定义 string 类型的别名，示例代码：

```go
func main() {
	type MyString = string
	str := "hello"
	a := MyString(str)
	b := MyString("A" + str)
	fmt.Printf("str type is %T\n", str)  //str type is string
	fmt.Printf("a type is %T\n", a)    //a type is string
	fmt.Printf("a == str is %t\n", a == str)  //a == str is true
	fmt.Printf("b > a is %t\n", b > a)  //b > a is false
}
```
别名类型与源类型是完全相同的；
别名类型与源类型可以在源类型支持的条件下进行相等判断、比较判断、与 nil 是否相等判断一致；

**类型定义**
类型定义是定义一种新的类型，它与源类型是不一样的。看下面代码：

```go
unc main() {
	type MyString string
	str := "hello"
	a := MyString(str)
	b := MyString("A" + str)
	fmt.Printf("str type is %T\n", str)  //str type is string
	fmt.Printf("a type is %T\n", a)    //a type is main.MyString
	fmt.Printf("a value is %#v\n", a)  //a value is "hello" 
	fmt.Printf("b value is %#v\n", b)  //b value is "Ahello"
	// fmt.Printf("a == str is %t\n", a == str)
	fmt.Printf("b > a is %t\n", b > a)   //b > a is false
}

```
可以看到 MyString 类型为 main.MyString 而原有的 str 类型为 string，两者是不同的类型，如果使用下面的判断相等语句

```go
fmt.Printf("a == str is %t\n", a == str)
```
会有编译错误提示

![在这里插入图片描述](https://img-blog.csdnimg.cn/0ab7a6d00c5d4635976b1fa122c46d00.png)
对于这里的类型再定义来说，string 可以被称为 MyString2 的潜在类型。潜在类型的含义是，某个类型在本质上是哪个类型。潜在类型相同的不同类型的值之间是可以进行类型转换的。

因此，MyString2 类型的值与 string 类型的值可以使用类型转换表达式进行互转。但对于集合类的类型[]MyString2 与 []string 来说这样做却是不合法的，因为 []MyString2 与 []string 的潜在类型不同，分别是 []MyString2 和 []string 。
**另外，即使两个不同类型的潜在类型相同，它们的值之间也不能进行判等或比较，它们的变量之间也不能赋值。**