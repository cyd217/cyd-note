---
title: go-内存管理篇（一） TCMalloc
tags:
  - golang 
  - 内存管理
categories:
	- golang
description: 
toc: true
toc_number: true
cover:  /img/post/c1494648e2ed49e9bf0ffcc3efd8f63c.png
---

## 1.内存为什么需要管理
我们知道**处理速度与存储容量**是成反比的。也就是说，性能越大的计算机硬件资源，越是稀缺，所以合理的利用和分配就越重要。大部分程序逻辑临时用的数据，全部都存在内存之中，比如，变量、全局变量、函数跳转地址、静态库、临时开辟的内存结构体（对象）等。当存储的东西越来越多，也就发现物理内存的容量依然是不够用，那么对物理内存的利用率和合理的分配，管理就变得非常的重要。
Golang编程语言给开发者提供了一套内存管理模式，所以开发者有必要了解一下Golang做了哪些助力的功能。

## 2.TcMalloc是什么?
Go内存管理是基于`TCMalloc`基础上进行设计的，所以在学习Go内存管理之前先学习`TCMalloc`原理。
TCMalloc(Thread Cache Malloc)是线程级别的内存管理模式。`TCMalloc`是用来替代**传统的malloc内存分配函数**。它有**减少内存碎片，适用于多核**，更好的并行性支持等特性。


## 3.TcMalloc分配原则

![在这里插入图片描述](https://img-blog.csdnimg.cn/4e8d3679c618474794736c44117b8f4d.png)
`TCMalloc`则是为**每个Thread预分配一块缓存**，每个Thread在申请内存时首先会先从这个缓存区**ThreadCache**申请，且所有`ThreadCache`缓存区还共享一个叫**CentralCache的中心缓存**。
好处：
- `ThreadCache`做为每个线程独立的缓存，能够明显的提高Thread获取高命中的数据
- `ThreadCache`也是从堆空间一次性申请，即只触发一次系统调用即可。

**CentralCache是所有线程共享的缓存**，当`ThreadCache`的缓存不足时，就会从`CentralCache`获取，当`ThreadCache`的缓存充足或者过多时，则会将内存退还给`CentralCache`。但是`CentralCache`由于共享，那么访问一定是需要加锁的。**ThreadCache作为线程独立的第一交互内存，访问无需加锁**，`CentralCache`则作为`ThreadCache`临时补充缓存。`ThreadCache`和`CentralCache`可以解决**小对象**内存块的申请。


![在这里插入图片描述](https://img-blog.csdnimg.cn/3ea8159cc4a64f3a8a35fbdf6c9c07c6.png)


为了解决中对象和大对象的内存申请，`TCMalloc`依然有一个全局共享内存堆`PageHeap`。
`PageHeap`也是一次系统调用从虚拟内存中申请的，`PageHeap`很明显是全局的，所以访问一定是要加锁。`PageHeap`发现也没有内存的时候,会向OS申请内存。
作用:
- 当`CentralCache`没有足够内存时会从`PageHeap`取，当`CentralCache`内存过多或者充足，则将低命中内存块退还`PageHeap`。
- Thread需要大对象申请超过的Cache容纳的内存块单元大小，也会直接从`PageHeap`获取。

**TCMalloc优势：**
1、速度快
2、减少锁竞争。对于小对象，只有在对应线程分配的空闲块不足的时候，才会使用到锁；对于大对象，TCMalloc尝试使用有效的自旋锁。

## 4.对象的分类
|对象	|  容量|
|--|--|
| 小对象 |(0,256KB]  |
| 中对象 |(256KB, 1MB] |
| 大对象|	(1MB, +∞) |
	
## 5.TCMalloc模型相关基础结构
### 5.1.Page
`Pages`是`TCMalloc`管理的内存基本单位，默认大小是8KB。

### 5.2.Span
`Span`是`PageHeap`中管理内存页的单位，它是由一组连续的Page组成，TCMolloc以span为单位向系统申请内存。`span`是由`PageHeap`进行管理的，可以被拆分成多个相同的`page size`用于小对象使用；也可以作为一个整体被中大对象进行使用。Span可用于管理已移交给应用程序的大对象(多个Page组成的大对象)，或已拆分为一系列小对象的一组页面(一个或多个`Page`被`Size-Class`拆分固定大小的Object链表)。如果Span管理的是小对象，则会在Span中记录对象的Size-Class信息。
![在这里插入图片描述](https://img-blog.csdnimg.cn/150bc268b1b54578ade459836a20e021.png)
### 5.3.Size-Class
![在这里插入图片描述](https://img-blog.csdnimg.cn/19ee9897709b4232b120a045a93270d8.png)

由`Span`分裂出的对象，由**同一个Span分裂出的SizeClass大小相同**，`SizeClass`是对象内存实际的载体。小对象的分配被映射到不同大小的`Size-class`类型上。例如，一个12字节的分配将被四舍五入到16字节Size-class。Size-class的设计是为了在舍入到下一个最大的size类时尽量减少浪费的内存量。

### 5.4.ThreadCache
![在这里插入图片描述](https://img-blog.csdnimg.cn/02d5e6f4ca0b4564bbf16cee0a419238.png)
`ThreadCache`中对于每个`Size Class`都会有一个对应的`FreeList，FreeList`表示当前缓存中还有多少个空闲的内存可用。使用方对于从`TCMalloc`申请的小对象，会直接从**TreadCache**获取，实则是从FreeList中返回一个空闲的对象，如果对应的Size Class刻度下已经没有空闲的Span可以被获取了，则`ThreadCache`会从`CentralCache`中获取。当使用方使用完内存之后，归还也是直接归还给当前的`ThreadCache`中对应刻度下的的`FreeList`中。

### 5.5.CentralCache
`CentralCache`是各个线程共用的，所以与CentralCache获取内存交互是需要加锁的。CentralCache缓存的`Size Class`和`ThreadCache`的一样，这些缓存都被放在`Central Free List`中。`Central Free List`是当`ThreadCache`内存不足时,提供内存供`ThreadCache`使用。每种规格的`Size-Class`,都从不同的 `Span` 进行分配;每种规则的`Size-class`都有一个独立的内存分配单元。每一个`size-class`都会关联一个`span List`,这个list中所有`span`的大小都是相同的,每个span都已经被拆分为对应的`size-class`。
### 5.6.PageHeap
`PageHeap`是提供`CentralCache`的内存来源。`PageHead`与`CentralCache`不同的是`CentralCache`是与`ThreadCache`布局一模一样的缓存，主要是起到针对`ThreadCache`的一层二级缓存作用，且只支持小对象内存分配。而`PageHeap`则是针对`CentralCache`的三级缓存。弥补对于中对象内存和大对象内存的分配，`PageHeap`也是直接和操作系统虚拟内存衔接的一层缓存，当ThreadCache、CentralCache、PageHeap都找不到合适的Span，PageHeap则会调用操作系统内存申请系统调用函数来从虚拟内存的堆区中取出内存填充到PageHeap当中。小于等于128 list都按照链表来进行缓存管理；超过128的存储在一个有序的set。
作用
- 管理未使用的内存。
- 当没有合适大小的可用内存来满足分配请求时, 它负责从操作系统获取内存。
- 将不需要的内存返回给操作系统。
![在这里插入图片描述](https://img-blog.csdnimg.cn/8a3f63e126824661a95134647a0faac2.png)
### 5.7.内存回收
上面说的都是内存分配，内存回收的情况是怎样的？

应用程序调用free()或delete一个小对象时，仅仅是将其插入到ThreadCache中其size class对应的FreeList中而已，不需要加锁，因此速度也是非常快的。

只有当满足一定的条件时，`ThreadCache`中的空闲对象才会重新放回`CentralCache`中，以供其他线程取用。同样的，当满足一定条件时，`CentralCache`中的空闲对象也会还给`PageHeap`，`PageHeap`再还给系统。

### 5.8.小结
小对象分配流程大致如下：

- 将要分配的内存大小映射到对应的`size class`。
- 查看`ThreadCache`中该`size class`对应的`FreeList`。
  -  如果`Free List`非空，则移除`Free List`的第一个空闲对象并将其返回，分配结束。
  - 如果`Free List`是空的：
    -  从`CentralCache`中`size class`对应的`Central Free List `获取一堆空闲对象。
    - 如果`Central Free List`也是空的，则向`PageHeap`申请一个`span`。拆分成`size class`对应大小的空闲对象，放入`Central Free List `中。
  - 将这堆对象放置到`ThreadCache`中`size class`对应的`FreeList`中（第一个对象除外）。
  - 返回从`CentralCache`获取的第一个对象，分配结束。
## 6.内存碎片处理
内存碎片就是不能再分配给应用使用。分配**内部碎片和外部碎片**，内部碎片就是内部碎片是分配器分配的内存大于程序申请的内存，内部产生碎片；外部碎片就是内存块太小，不足以分配给应用使用。
对于TCMalloc是怎么处理内部碎片和外部碎片的？
内部碎片：TCMalloc提前分配了多种size-class：8， 16， 32， 48， 64， 80， 96， 112， 128， 144， 160， 176…
TCMalloc的目标就是产生最多12.5%的内存碎片。可以看到上面不是按照2的幂级数分配的大小，这是因为如果按照2的幂产生的碎片会更大。比如申请65字节，2幂申请的话会分配128，而按照TCMalloc只分配80，相应的减少了很多碎片。
-  16字节以内，每8字节划分一个size class：8,16
- 16~128字节，每16字节划分一个size class：32,48,64…
- 128B~256字节，按照每次增加x/8进行增加：128+128/8=144 以此类推
- 大于大于1024的 size-class 其实都以128对齐：

外部碎片：
`TCMalloc`的`CentralCache`向`PageHeap`申请内存的时候，是以`Page`为单位进行申请的。当申请1024的时候，
page(8192)%1024=0没有内存碎片，当时当申请class-size为1152的时候（8192%1152=128）产生128的外部碎片，为了使得内存碎片率最多12.5%，可以多申请几个`Page`来解决。也就是合并相邻的Page，可以减少外部碎片。
TCMalloc也考虑相同的`class-size`进行合并，这里的相同就是指分配的对象大小相同，取一个碎片更少的size进行使用。



## 参考文章
[https://wallenwang.com/2018/11/tcmalloc/#ftoc-heading-24](https://wallenwang.com/2018/11/tcmalloc/#ftoc-heading-24)
[https://blog.csdn.net/kelvin_yin/article/details/78997953](https://blog.csdn.net/kelvin_yin/article/details/78997953)
[https://zhuanlan.zhihu.com/p/572059278](https://zhuanlan.zhihu.com/p/572059278)