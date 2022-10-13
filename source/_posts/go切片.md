---
title: go-你不知道的切片
tags:
  - golang 
  - 切片
categories:
	- golang
---

### 切片概念
Slice又称动态数组，依托数组实现，可以方便的进行扩容、传递等，实际使用中比数组更灵活。
正因为灵活，如果不了解其内部实现机制，有可能遭遇莫名的异常现象。Slice的实现原理很简单，本节试图根据真实的使用场景，在源码中总结实现原理。

Slice（切片）代表变长的序列，序列中每个元素都有相同的类型。一个slice类型一般写作[]T，其中T代表slice中元素的类型；slice的语法和数组很像，只是没有固定长度而已。

数组和slice之间有着紧密的联系。一个slice是一个轻量级的数据结构，提供了访问数组子序列元素的功能，而且slice的底层确实引用一个数组对象。一个slice由三个部分构成：**指针、长度和容量**。指针指向第一个slice元素对应的底层数组元素的地址。
注意的是**slice的第一个元素并不一定就是数组的第一个元素**。长度对应slice中元素的数目；长度不能超过容量，容量一般是从slice的开始位置到底层数据的结尾位置。**内置的len和cap函数分别返回slice的长度和容量**。

切片的底层源码
```go
//切片的结构体由3部分构成，
//Pointer 是指向一个数组的指针，len代表当前切片的长度，cap是当前切片的容量。cap总是大于等于len的。
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/55fab33aaf4a48e8a1b2a20ef46768db.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/97b02064da034f4d98464a4595abe023.png)
### 创建切片的方式
**1.声明空切片(nil)**
```go
//声明切片
	var s1 []int
	if s1 == nil {
		fmt.Println("是空")    //是空
	} else {
		fmt.Println("不是空")
	}
	fmt.Println(s1, len(s1), cap(s1))  //[] 0 0
```
** 2.make切片**
```go
var s3 []int = make([]int, 10)
fmt.Println(s3, len(s3), cap(s3))  //[0 0 0 0 0 0 0 0 0 0] 10 10

var s4 []int = make([]int, 0, 10)
fmt.Println(s4, len(s4), cap(s4)) //[] 0 10
```
**3.从数组中切片**

```go
	arr := [5]int{1, 2, 3, 4, 5}
	var s6 []int
	// 前包后不包
	s6 = arr[1:4]  
	fmt.Println(s6, len(s6), cap(s6)) //[2 3 4] 3 4
	s6 = arr[1:4:4]  
	fmt.Println(s6, len(s6), cap(s6))//[2 3 4] 3 3
```
发现：通过 s6 = arr[1:4]   操作后，我们发现s6的切片的len=3,cap=4。 s6的指针指向了s数组索引为2的数组位置，s6切片所指向的数据为{2,3,4} 这也是**左闭右开**原则，所以长度len=3, 但因为指针指向s数组索引位置2，**其当前切片的可用的容量是从当前指针位置往后数**，即当前指针到最后有4个位置空间。即cap=4。 说明了arr数组的len和cap并没有因此改变。

### 切片的初始化操作

|操作|含义  |
|--|--|
| s[n] |切片s中索引位置为n的项  |
| s[:] |从切片s的索引位置0到len(s)-1处所获取的切片|
| s[low:] |从切片s的索引位置low到len(s)-1处所获取的切片  |
| s[:high] |从切片s的索引位置0到high处所获取的切片 ，len=high |
| s[low:high] |从切片s的索引位置low到high处所获取的切片,len=high-low  |
| s[low:high:max] |从切片s的索引位置low到high处所获取的切片,len=high-low ,cap=max-high   |
| len(s) |切片s中长度总是<=cap(s) |
| cap(s) |切片s中容量总是>=len(s) |


### Slice 扩容
使用append向Slice追加元素时，如果Slice空间不足，将会触发Slice扩容，扩容实际上是重新分配一块更大的内存，将原Slice数据拷贝进新Slice，然后返回新Slice，扩容后再将数据追加进去。

```go
func main() {
	slice := []int{10, 20, 30, 40}
	newSlice := append(slice, 50)
	fmt.Printf("Before slice = %v, Pointer = %p, len = %d, cap = %d\n", slice, &slice, len(slice), cap(slice))
	fmt.Printf("Before newSlice = %v, Pointer = %p, len = %d, cap = %d\n", newSlice, &newSlice, len(newSlice), cap(newSlice))
	newSlice[1] += 10
	slice[3] += 10
	fmt.Printf("After slice = %v, Pointer = %p, len = %d, cap = %d\n", slice, &slice, len(slice), cap(slice))
	fmt.Printf("After newSlice = %v, Pointer = %p, len = %d, cap = %d\n", newSlice, &newSlice, len(newSlice), cap(newSlice))
}

//
Before slice = [10 20 30 40], Pointer = 0xc4200b0140, len = 4, cap = 4
Before newSlice = [10 20 30 40 50], Pointer = 0xc4200b0180, len = 5, cap = 8
After slice = [10 20 30 50], Pointer = 0xc4200b0140, len = 4, cap = 4
After newSlice = [10 30 30 40 50], Pointer = 0xc4200b0180, len = 5, cap = 8
```

例如，当向一个capacity为4，且length也为4的Slice再次追加1个元素时，就会发生扩容，如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/24db7391f5c0497bbba81219d312e262.png)

扩容操作只关心容量，会把原Slice数据拷贝到新Slice，追加数据由append在扩容结束后完成。上图可见，扩容后新的Slice长度仍然是4，但容量由4提升到了8，原Slice的数据也都拷贝到了新Slice指向的数组中。扩容之前，新旧数组互不影响。
扩容容量的选择遵循以下规则：
如果原Slice容量小于1024，则新Slice容量将扩大为原来的2倍；
如果原Slice容量大于等于1024，则新Slice容量将扩大为原来的1.25倍；
使用append()向Slice添加一个元素的实现步骤如下：
假如Slice容量够用，则将新元素追加进去，Slice.len++，返回原Slice
原Slice容量不够，则将Slice先扩容，扩容后得到新Slice，将新元素追加进新Slice，Slice.len++，返回新的Slice。


**扩容之后的数组一定是新的么**
```go
func main() {
	array := [4]int{10, 20, 30, 40}
	slice2 := array[0:2]
	newSlice2 := append(slice2, 50)
	fmt.Printf("Before slice2 = %v, Pointer = %p, len = %d, cap = %d\n", slice2, &slice2, len(slice2), cap(slice2))
	fmt.Printf("Before newSlice2 = %v, Pointer = %p, len = %d, cap = %d\n", newSlice2, &newSlice2, len(newSlice2), cap(newSlice2))
	newSlice2[1] += 10
	fmt.Printf("After slice2 = %v, Pointer = %p, len = %d, cap = %d\n", slice2, &slice2, len(slice2), cap(slice2))
	fmt.Printf("After newSlice2 = %v, Pointer = %p, len = %d, cap = %d\n", newSlice2, &newSlice2, len(newSlice2), cap(newSlice2))
	fmt.Printf("After array = %v\n", array)

//Before slice2 = [10 20], Pointer = 0xc000004108, len = 2, cap = 4
//Before newSlice2 = [10 20 50], Pointer = 0xc000004120, len = 3, cap = 4
//After slice2 = [10 30], Pointer = 0xc000004108, len = 2, cap = 4
//After newSlice2 = [10 30 50], Pointer = 0xc000004120, len = 3, cap = 4
//After array = [10 30 50 40]
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/47c587d021e04c659cee9b6a5d6db731.png)

在这种情况下，扩容以后并没有新建一个新的数组，**扩容前后的数组都是同一个**，**这也就导致了新的切片修改了一个值**，也影响到了老的切片了。并且 append() 操作也改变了原来数组里面的值。一个 append() 操作影响了这么多地方，如果原数组上有多个切片，那么这些切片都会被影响！无意间就产生了莫名的 bug！这种情况，由于原数组还有容量可以扩容，所以执行 append() 操作以后，会在原数组上直接操作，所以这种情况下，扩容以后的数组还是指向原来的数组。


### 切片拷贝

```go
//copy 函数将数据从源 Slice复制到目标 Slice。它返回复制的元素数。
func copy（dst，src [] T）int
```

copy 函数支持在不同长度的 Slice 之间进行复制，若出现长度不一致，在复制时会按照最少的 Slice 元素个数进行复制

```go
func main() {
	dst := []int{1, 2, 3}
	src := []int{4, 5, 6, 7, 8}
	fmt.Printf("Before dst = %v, Pointer = %p, len = %d, cap = %d\n", dst, &dst, len(dst), cap(dst))
	fmt.Printf("Before src = %v, Pointer = %p, len = %d, cap = %d\n", src, &src, len(src), cap(src))
	n := copy(dst, src)
	fmt.Printf("n = %d After dst = %v, Pointer = %p, len = %d, cap = %d\n", n, dst, &dst, len(dst), cap(dst))
	fmt.Printf("n = %d After src = %v, Pointer = %p, len = %d, cap = %d\n", n, src, &src, len(src), cap(src))

//Before dst = [1 2 3], Pointer = 0xc000004078, len = 3, cap = 3
//Before src = [4 5 6 7 8], Pointer = 0xc000004090, len = 5, cap = 5
//n = 3 After dst = [4 5 6], Pointer = 0xc000004078, len = 3, cap = 3
//n = 3 After src = [4 5 6 7 8], Pointer = 0xc000004090, len = 5, cap = 5
}
```

源码如下

```go
func slicecopy(to, fm slice, width uintptr) int {
// 如果源切片或者目标切片有一个长度为0，那么就不需要拷贝，直接 return 
    if fm.len == 0 || to.len == 0 {
        return 0
    }
// n 记录下源切片或者目标切片较短的那一个的长度
    n := fm.len
    if to.len < n {
        n = to.len
    }
// 如果入参 width = 0，也不需要拷贝了，返回较短的切片的长度
    if width == 0 {
        return n
    }

    ...

    size := uintptr(n) * width
    if size == 1 {
   // 如果只有一个元素，那么指针直接转换即可
        *(*byte)(to.array) = *(*byte)(fm.array) 
    } else {
     // 如果不止一个元素，那么就把 size 个 bytes 从 fm.array 地址开始，拷贝到 to.array 地址之后
        memmove(to.array, fm.array, size)
    }
    return n
}
```
- 若源 Slice 或目标 Slice 存在长度为 0 的情况，则直接返回 0（因为压根不需要执行复制行为）
- 通过对比两个 Slice，获取最小的 Slice 长度。便于后续操作
- 若 Slice 只有一个元素，则直接利用指针的特性进行转换
- 若 Slice 大于一个元素，则从 fm.array 复制 size 个字节到 to.array 的地址处（会覆盖原有的值）
- 
还有一个拷贝的方法，这个方法原理和 slicecopy 方法类似，不在赘述了，注释写在代码里面了。

```go
func slicestringcopy(to []byte, fm string) int {
    // 如果源切片或者目标切片有一个长度为0，那么就不需要拷贝，直接 return 
    if len(fm) == 0 || len(to) == 0 {
        return 0
    }
    // n 记录下源切片或者目标切片较短的那一个的长度
    n := len(fm)
    if len(to) < n {
        n = len(to)
    }
   .....
    // 拷贝字符串至字节数组
    memmove(unsafe.Pointer(&to[0]), stringStructOf(&fm).str, uintptr(n))
    return n
}
```

```go
func main() {
	slice := make([]byte, 3)
	fmt.Printf("Before dst = %v, Pointer = %p, len = %d, cap = %d\n", slice, &slice, len(slice), cap(slice))
	n := copy(slice, "abcdef")

	fmt.Printf("n = %d After src = %v, Pointer = %p, len = %d, cap = %d\n", n, slice, &slice, len(slice), cap(slice))

// Before dst = [0 0 0], Pointer = 0xc000004078, len = 3, cap = 3
// n = 3 After src = [97 98 99], Pointer = 0xc000004078, len = 3, cap = 3
}

```


### 遍历切片

```go

func main() {

	b := [5]int{2, 3, 5, 7, 11}
	fmt.Printf("  b = %v, Pointer = %p, len = %d, cap = %d\n", b, &b, len(b), cap(b))
	for i, _ := range b {
		b[i] -= 1
	}
	fmt.Printf("  b = %v, Pointer = %p, len = %d, cap = %d\n", b, &b, len(b), cap(b))

	a := [...]int{2, 3, 5, 7, 11}
	for i, prime := range a {
		fmt.Printf("value = %d , value-addr = %x , slice-addr = %x\n", prime, &prime, &a[i])
		prime -= 1
	}
	fmt.Printf("  a = %v, Pointer = %p, len = %d, cap = %d\n", a, &a, len(a), cap(a))
	for i, _ := range a {
		a[i] -= 1
	}
	fmt.Printf("  a = %v, Pointer = %p, len = %d, cap = %d\n", a, &a, len(a), cap(a))
}

// b = [2 3 5 7 11], Pointer = 0xc00000a3c0, len = 5, cap = 5
//  b = [1 2 4 6 10], Pointer = 0xc00000a3c0, len = 5, cap = 5
//value = 2 , value-addr = c0000120d0 , slice-addr = c00000a450
//value = 3 , value-addr = c0000120d0 , slice-addr = c00000a458
//value = 5 , value-addr = c0000120d0 , slice-addr = c00000a460
//value = 7 , value-addr = c0000120d0 , slice-addr = c00000a468
//value = 11 , value-addr = c0000120d0 , slice-addr = c00000a470
//  a = [2 3 5 7 11], Pointer = 0xc00000a450, len = 5, cap = 5
//  a = [1 2 4 6 10], Pointer = 0xc00000a450, len = 5, cap = 5

```
![在这里插入图片描述](https://img-blog.csdnimg.cn/d690a3a73b8c448cbc0df3840bc3c48a.png)

从上面结果我们可以看到，如果用 range 的方式去遍历一个切片，拿到的 Value 其实是切片里面的值拷贝。所以每次打印 Value 的地址都不变。由于 Value 是值拷贝的，并非引用传递，所以直接改 Value 是达不到更改原切片值的目的的，需要通过 &a[i] 获取真实的地址。

- 被遍历的容器值是aContainer的一个副本。 注意，只有aContainer的直接部分被复制了。 此副本是一个匿名的值，所以它是不可被修改的。
- 如果aContainer是一个**数组**，那么在遍历过程中对此数组元素的修改不会体现到循环变量中。 原因是此数组的副本（被真正遍历的容器）和此数组不共享任何元素。
- 如果aContainer是一个**切片**（或者映射），那么在遍历过程中对此切片（元素的修改将体现到循环变量中。 原因是此切片的副本和此切片（或者映射）共享元素。在遍历中的每个循环步，aContainer副本中的一个键值元素对将被赋值给循环变量。 所以对循环变量的直接修改将不会体现在aContainer中的对应元素中。 （因为这个原因，并且for-range循环是遍历映射条目的唯一途径，所以最好不要使用大尺寸的映射键值和元素类型，以避免较大的复制负担。）
- 所有被遍历的键值对将被赋值给同一对循环变量实例。


遍历一个nil映射或者nil切片是允许的。这样的遍历可以看作是一个空操作。
一些关于遍历映射条目的细节：
- 映射中的条目的遍历顺序是不确定的。或者说，同一个映射中的条目的两次遍历中，条目的顺序很可能是不一致的，即使在这两次遍历之间，此映射并未发生任何改变。
- 如果在一个映射中的条目的遍历过程中，一个还没有被遍历到的条目被删除了，则此条目保证不会被遍历出来。
- 如果在一个映射中的条目的遍历过程中，一个新的条目被添加入此映射，则此条目并不保证将在此遍历过程中被遍历出来。


**把数组指针当做数组来使用**
对于某些情形，我们可以把数组指针当做数组来使用。
我们可以通过在range关键字后跟随一个数组的指针来遍历此数组中的元素。
对于大尺寸的数组，这种方法比较高效，因为复制一个指针比复制一个大尺寸数组的代价低得多。
下面的例子中的两个循环是等价的，它们的效率也基本相同。

```go
func main() {
    var a [100]int
    for i, n := range &a { // 复制一个指针的开销很小
        fmt.Println(i, n)
    }
    for i, n := range a[:] { // 复制一个切片的开销很小
        fmt.Println(i, n)
    }
}
```

### 使用 slice 遇到的"坑"
使用 slice 时，如果不清楚 slice 的底层原理，还是会遇到一些意想不到的问题，而有些问题有时会很隐蔽，这里总结下，避免踩坑。

**slice 初始化的"坑"**
初始化一个 slice 有两种方式:

```go
直接声明: 比如 var s []int
使用 make 关键字，比如: s := make([]int, 0)
```
这两种方式有什么区别呢？在日常 coding 中应该如何选择，这是一个容易被忽视的问题。
首先，先说下区别：
**直接声明 slice 的方式内部是不申请内存空间的，slice 内部 array 指针指向nil。
使用 make 关键字会申请包含 0 个元素的内存空间，底层 array 指针指向申请的内存。**
![在这里插入图片描述](https://img-blog.csdnimg.cn/13454bb327464b2a85ba2a27ef03eebe.png)
清楚两者的区别后，再说下遇到的"坑"。有次在开发服务接口声明 slice 之后，然后通过 json.Marshal 序列化 slice，但是序列化的结果是有区别的。
```go
json.Marshal(直接声明): 返回 null
json.Marshal(make关键字初始化): 返回 []
```


**slice 扩容的"坑"**

这个坑在面试中经常会遇到，当 slice 作为函数参数时，如果在函数内部发生了扩容，这时再修改 slice 中的值是不起作用的，因为修改发生在新的 array 内存中，对老的 array 内存不起作用。

```go
package main

import "fmt"

func main() {
	var s = []int{1, 2, 3}
	modifySlice(s)
	fmt.Println(s) // 打印 [1 2 3]
}

func modifySlice(s []int) {
	s = append(s, 4)
	s[0] = 4
}
```

### 总结
创建切片时可根据实际需要预分配容量，尽量避免追加过程中扩容操作，有利于提升性能
切片拷贝时需要判断实际拷贝的元素个数
谨慎使用多个切片操作同一个数组，以防读写冲突
每个切片都指向一个底层数组
每个切片都保存了当前切片的长度、底层数组可用容量
使用len()、cap()计算切片长度、容量时，时间复杂度均为O(1)，不需要遍历切片
通过函数传递切片时，不会拷贝整个切片，因为切片本身只是个结构体而矣
使用 append() 向切片追加元素时有可能触发扩容，扩容后将会生成新的切片
