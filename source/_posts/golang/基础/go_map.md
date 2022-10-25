---
title: golang-map
tags:
  - golang 
categories:
	- golang
description: 
toc: true
toc_number: true
cover:  /img/post/6833939bly1giph4fomxoj20zk0m8axp.jpg
---



### 什么是Map？
**声明 & 初始化map**
map是一个拥有**键值对元素**的无序集合。在这个集合中，**键的值是唯一的**，键对应的值可以通过键来获取、更新或移除。
在 Go 语言中， map是散列表的引用， map的类型是map[K]V ，其中 K 和 V 是字典的键和值对应的数据类型。 map 中所有的**键都拥有相同的数据类型**，同时**所有的值也都拥有相同的数据类型**，但是键的类型和值的类型不一定相同。**键的类型 K ，必须是可以通过操作符==来进行比较的数据类型**，所以 map 可以检测某一个键是否已经存在。

### go map的数据结构(重点)
**核心结体体**
![在这里插入图片描述](https://img-blog.csdnimg.cn/15f2a059687d46c59b52bb687069d9af.png)
**哈希桶（hmap）**
指整个哈希数组，数组内每个元素是一个桶。
**桶链**
哈希桶的一个桶以及该桶下挂着的所有溢出桶。
**桶（bucket）**
一个bmap结构，与溢出桶的区别在于它是哈希桶数组上的一个元素。严格来说hmap.buckets指向桶组成的数组，每个桶的头部是bmap，之后是8个key，再是8个value，最后是1个溢出指针。溢出指针指向额外的桶链表，用于存储溢出的数据
**溢出桶（overflow bucket）**
一个bmap结构，与桶区别是，它不是哈希桶数组的元素，而是挂在哈希桶数组上或挂在其它溢出桶上。
**负载因子(overFactor)**
表示平均每个哈希桶的元素个数（注意是哈希桶，不包括溢出桶）。如当前map中总共有20个元素，哈希桶长度为4，则负载因子为5。负载因子主要是来判断当前map是否需要扩容。
新、旧哈希桶
新、旧哈希桶的概念只存在于map扩容阶段，在哈希桶扩容时，会申请一个新的哈希桶，原来的哈希桶变成了旧哈希桶，然后会分步将旧哈希桶的元素迁移到新桶上，当旧哈希桶所有元素都迁移完成时，旧哈希桶会被释放掉。

**核心源码**
```go
const ( // 关键的变量
    bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits  // 一个桶最多存储8个key-value对
	loadFactorNum = 13 // 扩散因子：loadFactorNum / loadFactorDen = 6.5。
	loadFactorDen = 2  // 即元素数量 >= (hash桶数量(2^hmp.B) * 6.5 / 8) 时，触发扩容
)
// map的基础数据结构
type hmap struct {
	count     int	 // map存储的元素对计数，len()函数返回此值，所以map的len()时间复杂度是O(1)
	flags     uint8  // 记录几个特殊的位标记，如当前是否有别的线程正在写map、当前是否为相同大小的增长（扩容/缩容？）
	B         uint8  // hash桶buckets的数量为2^B个
	noverflow uint16 // 溢出的桶的数量的近似值
	hash0     uint32 // hash种子

	buckets    unsafe.Pointer // 指向2^B个桶组成的数组的指针，数据存在这里
	oldbuckets unsafe.Pointer // 指向扩容前的旧buckets数组，只在map增长时有效
	nevacuate  uintptr        // 计数器，标示扩容后搬迁的进度

	extra *mapextra // 保存溢出桶的链表和未使用的溢出桶数组的首地址
}

type mapextra struct {
    overflow    *[]*bmap  //overflow 为 hmap.buckets （当前）溢出桶的指针地址
    oldoverflow *[]*bmap  //oldoverflow 为 hmap.oldbuckets （旧）溢出桶的指针地址
    nextOverflow *bmap  //nextOverflow 为空闲溢出桶的指针地址
}
// 桶的实现结构
type bmap struct {
	// tophash存储桶内每个key的hash值的高字节
	// tophash[0] < minTopHash表示桶的疏散状态
	// 当前版本bucketCnt的值是8，一个桶最多存储8个key-value对
	tophash [bucketCnt]uint8
	// 特别注意：
	// 实际分配内存时会申请一个更大的内存空间A，A的前8字节为bmap
	// 后面依次跟8个key、8个value、1个溢出指针
	// map的桶结构实际指的是内存空间A
	keys    [bucketCnt]KeyType
	values  [bucketCnt]ValueType
	overflow *bmap //溢出bucket的地址
//上述中keys   values  和overflow并不是在结构体中显示定义的，而是直接通过指针运算进行访问的。
}

// map.go里很多函数的第1个入参是这个结构，从成员来看很明显，此结构标示了键值对和桶的大小等必要信息
// 有了这个结构的信息，map.go的代码就可以与键值对的具体数据类型解耦
// 所以map.go用内存偏移量和unsafe.Pointer指针来直接对内存进行存取，而无需关心key或value的具体类型
type maptype struct {
	typ        _type
	key        *_type
	elem       *_type
	bucket     *_type // internal type representing a hash bucket
	keysize    uint8  // size of key slot
	valuesize  uint8  // size of value slot
	bucketsize uint16 // size of bucket
	flags      uint32
}
```
- Go 用了增量扩容。而 buckets 和 oldbuckets 也是与扩容相关的载体，一般情况下只使用 buckets，oldbuckets 是为空的。但如果正在扩容的话，oldbuckets 便不为空，buckets 的大小也会改变
- 当 hint 大于 8 时，就会使用 *mapextra 做溢出桶。若小于 8，则存储在 buckets 桶中
- map底层创建时，会初始化一个hmap结构体，同时分配一个足够大的内存空间A。其中A的前段用于hash数组，A的后段预留给溢出的桶。于是hmap.buckets指向hash数组，即A的首地址；hmap.extra.nextOverflow初始时指向内存A中的后段，即hash数组结尾的下一个桶，也即第1个预留的溢出桶。所以当hash冲突需要使用到新的溢出桶时，会优先使用上述预留的溢出桶，hmap.extra.nextOverflow依次往后偏移直到用完所有的溢出桶，才有可能会申请新的溢出桶空间。
![在这里插入图片描述](https://img-blog.csdnimg.cn/0e450ff30ca54871a8f445552fa3ad15.png)
**bmap结构图**
当往map中存储一个kv对时，通过k获取hash值，**hash值的低八位和bucket数组长度取余**，定位到在数组中的那个下标，hash值的高八位存储在bucket中的tophash中，用来快速判断key是否存在，key和value的具体值则通过指针运算存储，当一个bucket满时，通过overfolw指针链接到下一个bucket。
![在这里插入图片描述](https://img-blog.csdnimg.cn/c2578e8ee4614cf9a9ce28536f62b9fa.png)

**为什么不是k1v1，k2v2….. 而是k1k2…v1v2…**，
我们看上面的注释说的 map[int64]int8,key是int64（8个字节），value是int8（一个字节），kv的长度不同，如果按照kv格式存放，则考虑内存对齐v也会占用int64，而按照后者存储时，8个v刚好占用一个int64,从这个就可以看出go的map设计之巧妙.


###  map创建
- 不带参数   make(map[keyType]valueType)
- 带参数      make(map[keyType]valueType, size)

**源码分析**
- 创建hmap，并初始化。
- 获取一个随机种子，保证同一个key在不同map的hash值不一样（安全考量）。
- 计算初始桶大小。
- 如果初始桶大小不为0， 则创建桶，有必要还要创建溢出桶结构。![在这里插入图片描述](https://img-blog.csdnimg.cn/e4f85d542dcf45a085c023fef7d250a7.png)


```go
// make(map[k]v, hint), hint即预分配大小
// 不传hint时，如用new创建个预设容量为0的map时，makemap只初始化hmap结构，不分配hash数组
func makemap(t *maptype, hint int, h *hmap) *hmap {
	// 1. 创建hmap，并初始化
	if h == nil {
		h = new(hmap)
	}
	// 获取一个随机种子
	h.hash0 = fastrand()

	// 计算初始桶大小。
	B := uint8(0)
	// overLoadFactor(hint, B)只有一行代码：return hint > bucketCnt && uintptr(hint) > loadFactorNum*(bucketShift(B)/loadFactorDen)
	// 即B的大小应满足 hint <= (2^B) * 6.5
	// 一个桶能存8对key-value，所以这就表示B的初始值是保证这个map不需要扩容即可存下hint个元素对的最小的B值
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// 这里分配hash数组
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		// makeBucketArray()会在hash数组后面预分配一些溢出桶，
		// h.extra.nextOverflow用来保存上述溢出桶的首地址
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}

//给map bucket分配内存
//  @param t map的类型
//  @param b 桶的个数2^b
//  @param dirtyalloc 是否要把返回的buckets指向dirtyalloc地址
//  @return buckets buckets地址
//  @return nextOverflow 溢出桶地址
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b) // base代表用户预期的桶的数量，即hash数组的真实大小
	nbuckets := base // nbuckets表示实际分配的桶的数量，>= base，这就可能会追加一些溢出桶作为溢出的预留
	if b >= 4 {
		// 这里追加一定数量的桶，并做内存对齐
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	// 这里大家可以思考下这个数组空间要怎么分配，其实就是n*sizeof(桶)，所以：
		// 每个桶前面是8字节的tophash数组，然后是8个key，再是8个value，最后放一个溢出指针
		// sizeof(桶) = 8 + 8*sizeof(key) + 8*sizeof(value) + 8
	
	return buckets, nextOverflow
}
```

**哈希桶初始大小**
在创建map时，当没有指定size大小或size为0时，不会创建哈希桶，会在插入元素时创建，避免只申请不使用导致的效率和内存浪费。当size不为0时，会根据size大小计算出哈希桶的大小。
- size <9，bucketLen = 1 (2^0) B=0
-  size < 14，bucketLen = 2 (2^1) B=1
- size < 27，bucketLen = 4 (2^2) B=2
- size < 53，bucketLen = 8 (2^3) B=3
- size < 104，bucketLen = 16 (2^4) B=4


### 插入或更新

> 向nil map赋值会引发panic​
map不支持并发读写

go map的插入操作，调用mapassign()函数。
- go map需要初始化才能使用，对空map插入会panic。hmap指针传递的方式，决定了map在使用前必须初始化
- go map不支持并发读写，会panic。如果一定要并发，请用sync.Map或自己解决冲突。

流程分析：
- 判断map是否为空、判断有没有协程并发写。
- 计算key的哈希值，设置写标志
- 如果buckets为空，申请一个长度为1的buckets。
- 找出改key对应的桶位置。
- 如果map正在迁移切该桶没有被迁移，迁移该桶。
- 遍历该桶，如果找到相同的key，返回val的位置。如果没有找到，找出下一个空位置，赋值tophash、key，返回val的位置。
- 判断map是否需要扩容，如果扩容，返回5的操作。
- 如果当前buckets和溢出buckets都没有位置了，添加一个溢出buckets，赋值tophash、key到第一个空位，返回val的位置。
-![](https://img-blog.csdnimg.cn/ae3ed1418edf4bffb0e74b0986a4a41c.png)

```go
// 新增或者替换key val  m[key]=val
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  //@Step1 如果map为nil，panic
  if h == nil {
    panic(plainError("assignment to entry in nil map"))
  }

  // 判断有没有协程正在写map
  if h.flags&hashWriting != 0 {
    throw("concurrent map writes")
  }
 // @Step2
  hash := alg.hash(key, uintptr(h.hash0)) // 这里得到uint32的hash值

  // 设置map写的标志
  h.flags ^= hashWriting

  //@Step3 buckets为nil，new一个
  if h.buckets == nil {
    h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
  }

again:
  //@Step4 找出key hash对应的桶
  bucket := hash & bucketMask(h.B)
  if h.growing() {
    //  @Step5 如果桶需要迁移，则把旧桶迁移
    growWork(t, h, bucket)
  }
  b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
  top := tophash(hash)

  var inserti *uint8
  var insertk unsafe.Pointer
  var elem unsafe.Pointer
  //@Step6 寻找map中有没有存在的key
  //  5-1 如果存在，更新key的值，则返回val的位置
  //  5-2 如果不存在，则记录bucket最近一个空位置的tophash 、key、value的位置
  //  5-2-1 判断bucket有没有溢出，如果没有溢出，则下一步。
  //  5-2-2 溢出了则找出下一个溢出桶，继续bucketloop上述操作
bucketloop:
  for {
    for i := uintptr(0); i < bucketCnt; i++ {
      if b.tophash[i] != top {
        //判断当前元素是否为空，如果为空，记录第一个为空的位置（方便找不到key时插入）
        if isEmpty(b.tophash[i]) && inserti == nil {
          inserti = &b.tophash[i]
          insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
          elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
        }
        if b.tophash[i] == emptyRest {
          break bucketloop
        }
        continue
      }
      k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
      if !t.key.equal(key, k) {
        continue
      }
      elem = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
      //找到了key
      goto done
    }
    //在常规桶中没有找到数据，在溢出桶继续找
    ovf := b.overflow(t)
    if ovf == nil {
      break
    }
    b = ovf
  }

  //@Step7 如果添加一个元素会造成bucket超过负载（6.5），或者溢出bucket太多
  //  扩充桶，返回上面逻辑bucketloop继续寻找val位置
  /// overLoadFactor函数用来判断map是否由于数据太多，需要增量1倍扩容
  //负载因子 > 6.5时，也即平均每个bucket存储的键值对达到6.5个。
  //overflow数量 > 2^15时，也即overflow数量超过32768时。

  // tooManyOverflowBuckets函数用来判断map是否需要等量迁移，map由于删除操作，溢出bucket很多，但是数据分布很稀疏，我们可以通过等量迁移，将数据更加紧凑的存储在一起，节约空间
  if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
    hashGrow(t, h)
    goto again 
  }

  //@Step8 当前bucket和溢出桶都满了，重新添加一个溢出桶
  if inserti == nil {
    newb := h.newoverflow(t, b)
    inserti = &newb.tophash[0]
    insertk = add(unsafe.Pointer(newb), dataOffset)
    elem = add(insertk, bucketCnt*uintptr(t.keysize))
  }

  // 储存key、tophash的位置。h.count +1
  if t.indirectkey() {
    kmem := newobject(t.key)
    *(*unsafe.Pointer)(insertk) = kmem
    insertk = kmem
  }
  if t.indirectelem() {
    vmem := newobject(t.elem)
    *(*unsafe.Pointer)(elem) = vmem
  }
  typedmemmove(t.key, insertk, key)
  *inserti = top
  h.count++

  // 返回val的位置
done:
  if h.flags&hashWriting == 0 {
    throw("concurrent map writes")
  }
  h.flags &^= hashWriting
  if t.indirectelem() {
    elem = *((*unsafe.Pointer)(elem))
  }
  return elem
}
```
**总结**

> 1.每个 bucket 有 8 个空位，在没有溢出，且所有的桶都装满了的情况下，装载因子算出来的结果是 8。因此当装载因子超过 6.5 时，表明很多 bucket 都快要装满了，查找效率和插入效率都变低了。在这个时候进行扩容是有必要的。
> 2.在装载因子比较小的情况下，这时候 map 的查找和插入效率也很低，而第 1 点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，即 map 里元素总数少，但是 bucket 数量多（真实分配的 bucket 数量多，包括大量的 overflow bucket）。​原因：不停地插入、删除元素。先插入很多元素，导致创建了很多 bucket，但是装载因子达不到第 1 点的临界值，未触发扩容来缓解这种情况。之后，删除元素降低元素总数量，再插入很多元素，导致创建很多的 overflow bucket，overflow bucket 数量太多，导致 key 会很分散，查找插入效率低得吓人。
> 对于命中条件 1，2 的限制，都会发生扩容。但是扩容的策略并不相同，毕竟两种条件应对的场景不同。​
对于 1，元素太多，而 bucket 数量太少，很简单：将 B 加 1，bucket 最大数量(2^B)直接变成原来 bucket 数量的 2 倍。于是，就有新老 bucket 了。注意，这时候元素都在老 bucket 里，还没迁移到新的 bucket 来。新 bucket 只是最大数量变为原来最大数量的 2 倍(2^B*2) 。​
对于条件 2，其实元素没那么多，但是 overflow bucket 数特别多，说明很多 bucket 都没装满。解决办法就是开辟一个新 bucket 空间，将老 bucket 中的元素移动到新 bucket，使得同一个 bucket 中的 key 排列地更紧密。这样，原来，在 overflow bucket 中的 key 可以移动到 bucket 中来。结果是节省空间，提高 bucket 利用率，map 的查找和插入效率自然就会提升。​
由于 map 扩容需要将原有的 key/value 重新搬迁到新的内存地址，如果有大量的 key/value 需要搬迁，会非常影响性能。因此 Go map 的扩容采取了一种称为“渐进式”的方式，原有的 key 并不会一次性搬迁完毕，每次最多只会搬迁 2 个 bucket。

map库容分为**等量迁移和加倍扩容**：等量迁移是为了让稀疏的数据分布更加紧凑（由于删除操作，map可能会很稀疏），加倍扩容是由于插入数据过多，申请一个加倍的空间来存储kv，同时加倍扩容也会删除空的槽位，让数据分布紧凑；
扩容后，B 增加了 1，意味着 buckets 总数是原来的 2 倍，原来 1 号的桶“裂变”到两个桶，某个 key 在搬迁前后 bucket 序号可能和原来相等，也可能是相比原来加上 2^B（原来的 B 值），取决于 hash 值 第 6 bit 位是 0 还是 1。原理看下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/ef20e9a978d64323b740b1fd21e59584.png)



### 删除key value
**语法**

```go
delete(map, key)
```
**删除流程**
1 校验map是否为空，map是否正在写，如果是，则直接报"concurrent map writes"错误
2 设置写标志，计算key的哈希值
3 计算桶链首地址和tophash
4 找到桶链下的所有桶的元素，如果找到key，处理指针相关。
5 设置该位置的tophash值，这个逻辑比较复杂，详细会在tophash详解里面介绍
6 map总元素个数减1
7 清除写标志

![在这里插入图片描述](https://img-blog.csdnimg.cn/a81eef2e87ab4373b1fb7ef3e0f94b4e.png)
**源码分析**

```go
// 删除key、val
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
  // 1.判断map是否处于写状态
  if h.flags&hashWriting != 0 {
    throw("concurrent map writes")
  }

//2 设置写标志，计算key的哈希值
  hash := t.hasher(key, uintptr(h.hash0))
  h.flags ^= hashWriting

  //3 找出对应的桶位置
  //  如果需要迁移，继续迁移
  bucket := hash & bucketMask(h.B)
  if h.growing() {
    growWork(t, h, bucket)
  }
  b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
  bOrig := b
  top := tophash(hash)

  //2
  //  2-1 找出tophash key val位置
  //  2-2 把tophash置为 emptyOne（当前位置为空，但后面还有元素）
  //  2-3 当前bucket后面没有元素，则置为emptyRest（当前位置为空，且后面没有元素）
search:
  for ; b != nil; b = b.overflow(t) {
    for i := uintptr(0); i < bucketCnt; i++ {
      if b.tophash[i] != top {
        if b.tophash[i] == emptyRest { //如果发现对应的tophash已经是emptyRest状态，则标识后面没有数据了
          break search
        }
        continue
      }
      k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
      k2 := k
      if !t.key.equal(key, k2) {
        continue
      }
      e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
      //清除val
      if t.indirectelem() {
        *(*unsafe.Pointer)(e) = nil
      } else if t.elem.ptrdata != 0 {
        memclrHasPointers(e, t.elem.size)
      } else {
        memclrNoHeapPointers(e, t.elem.size)
      }
      // 把tophash置为 emptyOne（当前位置为空，但后面还有元素）
      b.tophash[i] = emptyOne
      //3 判断下一个位置（可能是溢出桶第一个）tophash是不是emptyRest
      //  3-1 如果不是，直接到notLast结束流程
      //  3-2 如果是，往前搜索，把所有emptyOne置为emptyRest
      if i == bucketCnt-1 {
        if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
          goto notLast
        }
      } else {
        if b.tophash[i+1] != emptyRest {
          goto notLast
        }
      }
      for {
        b.tophash[i] = emptyRest
        if i == 0 {
          if b == bOrig {
            break 
          }
          c := b
          for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
          }
          i = bucketCnt - 1
        } else {
          i--
        }
        if b.tophash[i] != emptyOne {
          break
        }
      }
      
    notLast:
      //数量减1
      h.count--
      break search
    }
  }
}
```


### map查找
 **语法**
-  第一种用法，只返回value值，当key在map中没有找到时，返回value对应类型的默认值。

```go
m := make(map[string]int)
print(m["1"]) -- 结果为: 0 (返回int对应的默认值)
```
- 第二种用法，返回value值以及key是否存在标志
```go
m := make(map[string]int)
val, ok := m["1"]
print(val, ok)  -- 结果为：0, false 
```

- 第三种用法，返回key和value，只用于range迭代场景
```go
m := make(map[string]int)
for k, v := range m {
}
```
**整体思路**
![在这里插入图片描述](https://img-blog.csdnimg.cn/7bd94f3fe75e49e0bf691f24c0d3a03c.png)

校验map是否正在写，如果是，则直接报"concurrent map read and map write"错误
计算key哈希值，并根据哈希值计算出key所在桶链位置
如果map正在扩（等量）容且该桶还未被迁移，定位到旧桶的位置
计算tophash值，便于快速查找
遍历桶链上的每一个桶，并依次遍历桶内的元素
先比较tophash，如果tophash不一致，再比较下一个，如果tophash一致，再比较key值是否相等
如果key值相等，计算出value的地址，并取出value值，并直接返回
如果key值不相等，则继续比较下一个元素，如果所有元素都不匹配，则直接返回value类型的默认值

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	...
	// 判断是否正在写，如果是直接报错
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	
	// 计算key哈希值
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
	
	// 比如 B=5，那 m 就是31，二进制是全 1
    // 求 bucket num 时，将 hash 与 m 相与，
    // 达到 bucket num 由 hash 的低 8 位决定的效果
    //假如 B=5 低5的值就是通的位置，即10。
    // 10010111 | 000011110110110010001111001010100010010110010101010 │ 01010
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	// 计算tophash值  高8位就是对应的tophash值 10010111-->151,ophash 值（HOB hash）为 151 的 key

  // 判断有没有旧桶（正在迁移）
  //  如果旧桶数据没有被迁移，定位到当前key在旧桶的位置
  //  如果当前旧桶没有被迁移，则迁移桶
  if c := h.oldbuckets; c != nil {
    if !h.sameSizeGrow() { 
    // 新 bucket 数量是老的 2 倍
      m >>= 1
    }
    //  求出 key 在老的 map 中的 bucket 位置
    oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
    if !evacuated(oldb) {
      b = oldb
    }
  }

	top := tophash(hash)
bucketloop:
	// 遍历桶链每个桶  从bucket中找出key对应的val，循环遍历top hash
	for ; b != nil; b = b.overflow(t) { 
	    // 遍历桶的元素
		for i := uintptr(0); i < bucketCnt; i++ {
			// 如果tophash不相等，continue
			if b.tophash[i] != top {
			// 如果top hash为emptyRest，则表示后面没数据了
				if b.tophash[i] == emptyRest { 
					break bucketloop
				}
				continue
			}
			// tophash相等
			// 获取当前位置对应的key值
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			// 如果与key匹配，表明找到了，直接返回value值
			if alg.equal(key, k) {
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				if t.indirectvalue() {
					v = *((*unsafe.Pointer)(v))
				}
				return v
			}
		}
	}
	
	// 一直没有找到，返回value类型的默认值
	return unsafe.Pointer(&zeroVal[0])
}
```


### 遍历  mapiterinit
**为什么map是无序的？​**
遍历的过程，就是按顺序遍历 bucket，同时按顺序遍历 bucket 中的 key。搬迁后，key 的位置发生了重大的变化，有些 key 飞上高枝，有些 key 则原地不动。这样，遍历 map 的结果就不可能按原来的顺序了。
hash表中数据每次插入的位置是变化的（其实是因为实现的原因，一方面hash种子是随机的，这导致相同的数据在不同的map变量内的hash值不同；另一方面即使同一个map变量内，数据删除再添加的位置也有可能变化，因为在同一个桶及溢出链表中数据的位置不分先后），所以为了防止用户错误的依赖于每次迭代的顺序。

**整体流程**
- 从hash数组中第it.startBucket个桶开始，先遍历hash桶，然后是这个桶的溢出链表。
之后hash数组偏移量+1，继续前一步动作。
- 遍历每一个桶，无论是hash桶还是溢出桶，都从it.offset偏移量开始。（如果只是随机一个开始的桶，range结果还是有序的；但每个桶都加it.offset偏移，这个输出结果就有点扑朔迷离，大家可以亲手试下，对同一个map多次range）
- 当迭代器经过一轮循环回到it.startBucket的位置，结束遍历。
![在这里插入图片描述](https://img-blog.csdnimg.cn/1e7f667a323f4427897344b1004f2574.png)

```go
// 初始化迭代器
func mapiterinit(t *maptype, h *hmap, it *hiter) {
  //1 初始化迭代器
  it.t = t
  if h == nil || h.count == 0 {
    return
  }
  it.h = h
  it.B = h.B
  it.buckets = h.buckets

  //2 给一个随机数，决定迭代桶的开始位置和桶内开始迭代的顺序
  r := uintptr(fastrand())
  if h.B > 31-bucketCntBits {
    r += uintptr(fastrand()) << 31
  }
  //it.startBucket：这个是hash数组的偏移量，表示遍历从这个桶开始
   //it.offset：这个是桶内的偏移量，表示每个桶的遍历都从这个偏移量开始
  it.startBucket = r & bucketMask(h.B)          //桶的偏移量
  it.offset = uint8(r >> h.B & (bucketCnt - 1)) //桶内偏移量

  it.bucket = it.startBucket
  ....
   mapiternext(it) // 初始化迭代器的同时也返回第1对key/value

}

//迭代器迭代
//  1 从随机偏移桶循环迭代每个桶数据
//  2 桶内从随机偏移量开始遍历
//  3 列出该方法隐掉了当正在进行扩容时怎么迭代，需要了解参考源码
func mapiternext(it *hiter) {
  if h.flags&hashWriting != 0 { //有协程正在写
    throw("concurrent map iteration and map write")
  }
  t := it.t
  bucket := it.bucket
  b := it.bptr
  i := it.i
  checkBucket := it.checkBucket

next:
  if b == nil { //当前桶指针为nil，标识桶内数据已经遍历完成，需要遍历下一个桶
    if bucket == it.startBucket && it.wrapped { //已经遍历到开始的桶
      it.key = nil
      it.elem = nil
      return
    }
    bucket++
    if bucket == bucketShift(it.B) {
      bucket = 0
      it.wrapped = true
    }
    i = 0
  }
  for ; i < bucketCnt; i++ {
    offi := (i + it.offset) & (bucketCnt - 1) //从桶内哪个位置开始遍历
    if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {//没数据
      continue
    }
    k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
    e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.elemsize))
  b = b.overflow(t)
  i = 0
  goto next
}
```

迭代还需要关注扩容缩容的情况：
如果是在迭代开始后才growing，这种情况当前的逻辑没处理，迭代有可能异常。呃，go map不支持并发。
如果是先growing，再开始迭代，这是有可能的。这种情况下，会先到旧hash表中检查key对应的桶有没有被疏散，未疏散则遍历旧桶，已疏散则遍历新hash表里对应的桶。


### go map的扩容缩容
```go
// overLoadFactor()返回true则触发扩容，即map的count大于hash桶数量(2^B)*6.5
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}

// tooManyOverflowBuckets()，顾名思义，溢出桶太多了触发缩容
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	if B > 15 {
		B = 15
	}
	return noverflow >= uint16(1)<<(B&15)
}
```
map只在插入元素即mapassign()函数中对是否扩容缩容进行触发，条件即是上面这段代码：

- 条件1：当前不处在growing状态
- 条件2-1：触发扩容：map的数据量count大于hash桶数量(2B)*6.5。注意这里的(2B)只是hash数组大小，不包括溢出的桶
- 条件2-2：触发缩容：溢出的桶数量noverflow>=32768(1<<15)或者>=hash数组大小。
仔细观察触发的代码，扩容和缩容是同一个函数，这是怎么做到的呢？在hashGrow()开始，会先判断是否满足扩容条件，如果满足就表明这次是扩容，不满足就一定是缩容条件触发了。扩容和缩容剩下的逻辑，主要区别就在于容量变化，就是hmap.B参数，扩容时B+1则hash表容量扩大1倍，缩容时hash表容量不变。
- h.oldbuckets：指向旧的hash数组，即当前的h.buckets
- h.buckets：指向新创建的hash数组

触发的主要工作已经完成，接下来就是怎么把元素搬迁到新hash表里了。如果现在就一次全量搬迁过去，显然接下来会有比较长的一段时间map被占用（不支持并发）。所以搬迁的工作是异步增量搬迁的。
在插入和删除的函数内都有下面一段代码用于在每次插入和删除操作时，执行一次搬迁工作.

```go
 if h.growing() { // 当前处于搬迁状态
		growWork(t, h, bucket) // 调用搬迁函数
	}
	
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// 将当前需要处理的桶搬迁
	evacuate(t, h, bucket&h.oldbucketmask())

	if h.growing() { // 再多搬迁一个桶
		evacuate(t, h, h.nevacuate)
	}
}
```
- 每执行一次插入或删除，都会调用growWork搬迁0~2个hash桶（有可能这次需要搬迁的2个桶在此之前都被搬过了）
- 搬迁是以hash桶为单位的，包含对应的hash桶和这个桶的溢出链表
- 被delete掉的元素(emptyone标志)会被舍弃（这是缩容的关键）

**为什么叫“伪缩容”？如何实现“真缩容”？**
因为缩容仅仅针对**溢出桶太多**的情况，触发缩容时hash数组的大小不变，即hash数组所占用的空间只增不减。也就是说，如果我们把一个已经增长到很大的map的元素挨个全部删除掉，hash表所占用的内存空间也不会被释放。所以如果要实现“真缩容”，需自己实现缩容搬迁，即创建一个较小的map，将需要缩容的map的元素挨个搬迁过来。

```go
// go map缩容代码示例
myMap := make(map[int]int, 1000000)

// 假设这里我们对bigMap做了很多次插入，之后又做了很多次删除，此时bigMap的元素数量远小于hash表大小
// 接下来我们开始缩容
smallMap := make(map[int]int, len(myMap))
for k, v := range myMap {
    smallMap[k] = v
}
myMap = smallMap // 缩容完成，原来的map被我们丢弃，交给gc去清理

```
**关键知识点**

5.1 基本原理
- 底层是hash实现，数据结构为hash数组 + 桶 + 溢出的桶链表，每个桶存储最多8个key-value对。
- 查找和插入的原理：key的hash值（低阶位）与桶数量相与，得到key所在的hash桶，再用key的高8位与桶中的tophash[i]对比，相同则进一步对比key值，key值相等则找到
- go map不支持并发。插入、删除、搬迁等操作会置writing标志，检测到并发直接panic
- 每次扩容hash表增大1倍，hash表只增不减
- 支持有限缩容，delete操作只置删除标志位，释放溢出桶的空间依靠触发缩容来实现。
- map在使用前必须初始化，否则panic：已初始化的map是make(map[key]value)或make(map[key]value, hint)这两种形式。而new或var xxx map[key]value这两种形式是未初始化的，直接使用会panic。

5.2 时间复杂度和空间复杂度分析
时间复杂度，go map是hash实现：
- 正常情况，且**不考虑扩容状态**，复杂度O(1)：通过hash值定位桶是O(1)，一个桶最多8个元素，合理的hash算法应该能把元素相对均匀散列，所以溢出链表（如果有）也不会太长，所以虽然在桶和溢出链表上定位key是遍历，考虑到数量小也可以认为是O(1)
- 正常情况，处于**扩容状态时**，复杂度也是O(1)：相比于上一种状态，扩容会增加搬迁最多2个桶和溢出链表的时间消耗，当溢出链表不太长时，复杂度也可以认为是O(1)
- 极端情况，散列极不均匀，大部分数据被集中在一条散列链表上，复杂度退化为O(n)。
所以综合情况下go map的时间复杂度应为O(1)

空间复杂度分析：
首先我们不考虑因删除大量元素导致的空间浪费情况（这种情况现在go是留给程序员自己解决），只考虑一个持续增长状态的map的一个空间使用率：
由于溢出桶数量超过hash桶数量时会触发缩容，所以最坏的情况是数据被集中在一条链上，hash表基本是空的，这时空间浪费O(n)。
最好的情况下，数据均匀散列在hash表上，没有元素溢出，这时最好的空间复杂度就是扩散因子决定了，当前go的扩散因子由全局变量决定，即loadFactorNum/loadFactorDen = 6.5。即平均每个hash桶被分配到6.5个元素以上时，开始扩容。所以最小的空间浪费是(8-6.5)/8 = 0.1875，即O(0.1875n)
结论：go map的空间复杂度（指除去正常存储元素所需空间之外的空间浪费）是O(0.1875n) ~ O(n)之间。