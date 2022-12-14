---
title: redis数据结构及对应的场景
tags:
  - redis
  - 并发
categories:
	- redis
toc: true
toc_number: true
cover:  https://img-blog.csdnimg.cn/c926524dd2e242a48cebf016bc30cbc6.png
---

**Redis 为什么那么快？**
除了它是**内存数据库**，使得所有的操作都在内存上进行之外，还有一个重要因素，它实现的数据结构，使得我们对数据进行增删查改操作时，Redis 能高效的处理。

# 一.redisObject对象头
Redis底层就是一个**大map**，key是字符串，value可能是字符串，哈希，列表等。
![在这里插入图片描述](https://img-blog.csdnimg.cn/6187d00384934a64bcc258d2fe9f4c73.png)
`编码方式type`和`对象头redisObject`有关，Redis中的每个对象底层的数据结构都是**redisObject结构体**，**不同的场景同一个对象可能使用不同的底层编码**，这样一个 `RedisObject `对象头需要占据 `16 `字节的存储空间。

> encoding：对应于object encoding命令，记录了对象使用的底层数据结构。
> 指令：object encoding key

每种对象类型，有可能使用了多种**编码类型(encoding)**，具体的对应关系如下
|对象类型  type   | 	编码类型 encoding |
|--|--|
|string	  | raw int embstr |
|list	| quicklist(快速链表)|
|hash	| dict（哈希表） ziplist（压缩列表）|
|set	| intset(整数集合)    dict （哈希表）|
|zset	| ziplist（压缩列表）  skiplist+dict（跳表）|


# 二.字符串 string
## 2.1.简单动态字符串(SDS)
![SDS结构](https://img-blog.csdnimg.cn/0db5e6f2880449c888fce24c6449cf95.png)          

Redis 中的字符串是可以**修改的字符串**，在内存中它是以**字节数组**的形式存在的。
相比较于C语言中的字符串:
- 头部信息里面包含了字符串的实际长度 `len`，可以通过`O（1）`的时间复杂度得到。
- C语言字符串的末尾之外，字符串里面不能含有 `“\0”` 字符，否则最先被程序读入的 `“\0” `字符将被误认为是字符串结尾，这个限制使得 C 语言的字符串只能保存**文本数据**，不能保存像图片、音频、视频文化这样的二进制数据。` SDS` 不需要用 `“\0” `字符来标识字符串结尾了，而是有个专门的` len `成员变量来记录长度，所以可存储包含 `“\0” `的数据。但是 `SDS `为了兼容部分 `C `语言标准库的函数， SDS 字符串结尾还是会加上` “\0” `字符。
-  `capacity- len` 计算出剩余的空间大小，可以用来判断空间是否满足修改需求，如果不满足的话，就会自动将 SDS 的空间扩展至执行修改所需的大小**（小于 1MB 翻倍扩容，大于 1MB 按 1MB 扩容）**。

**问题：上面的 SDS 结构使用了范型 T，为什么不直接用 int 呢**
因为当字符串比较短时，len 和 capacity 可以使用 byte 和 short 来表示，Redis 为了对内存做极致的优化，不同长度的字符串使用不同的结构体来表示。

## 2.2.为什么要使用SDS？
- **缓冲区溢出**
Redis提供的SDS内置的空间分配策略则可以完全杜绝这种事情的发生。当API需要对SDS进行修改时, API会首先会检查SDS的空间是否满足条件, 如果不满足, API会自动对它动态扩展, 然后再进行修改。
- **空间预分配**
字符串之所以采用**预分配**的方式是防止修改操作需要不断重分配内存和字节数据拷贝。但同样也会造成内存的浪费。字符串预分配每次并不都是翻倍扩容，空间预分配规则如下:
   - 第一次创建`len`属性等于数据实际大小，free等于0，不做预分配。
   - 修改后如果已有`free`空间不够且数据小于1M，每次预分配一倍容量。如原有`len=10byte，free=0`，再追加`10byte`，预分配`20byte`，总占用空间:10byte+10byte+20byte+1byte。
   -  修改后如果已有`free`空间不够且数据大于`1MB`，每次预分配`1MB`数据。如原有`len=30MB，free=0`，当再追加`100byte` ,预分配`1MB`，总占用空间:`30MB+100byte+1MB+1byte`。
   
**开发提示:尽量减少字符串频繁修改操作如append，setrange, 改为直接使用set修改字符串，降低预分配带来的内存浪费和内存碎片化。**
![在这里插入图片描述](https://img-blog.csdnimg.cn/4b2dc8d3eefd4deb99778fe2845321cf.png)
- **惰性空间释放**
  惰性空间释放用于字符串缩短的操作。当字符串缩短是，程序并不是立即使用内存重分配来回收缩短出来的字节，而是使用`free`属性记录起来，并等待将来使用。Redis通过**空间预分配和惰性空间释放策略**在字符串操作中一定程度上减少了内存重分配的次数。
- 二进制安全     可用于保存字节数组，支持安全的二进制数据存储。
## 2.3.时间复杂度
- 获取`SDS`长度: 由于`SDS`中提供了`len`属性，因此我们可以直接获取时间复杂度为`O(1)`。
- 获取`SDS`未使用空间长度: 时间复杂度为`0(1)`。
- 清除`SDS`保存的内容:由于惰性空间分配策略，复杂度为`O(1)`。
- 创建一个长度为`N`的字符串:时间复杂度为`O(n)`。
- 拼接一个长度为`N`的`C`字符串:时间复杂度为`O(n)`。
- 拼接一个长度为`N`的`SDS`字符串:时间复杂度为`O(n)`。
- `Redis`在获取字符串长度上的时间复杂度为常数级`O(1)`。

## 2.4 编码方式（redisObject.encoding）
- **int**
当存储的字符串`全是数字`的时候，此时使用`int`方式来存储
- **embstr**
当存储的字符串长度**小于等于44**字符的时候，使用`embstr`方式来存储
-  **raw**
当存储的字符串长度**大于等于44**字符的时候，使用`raw`方式来存储
![在这里插入图片描述](https://img-blog.csdnimg.cn/f9326e04f80f412bb85edd62835c158e.png)

**为什么上面的阈值是44呢？**
- 一个字符串包含RedisObject和SDS的数据结构，至少会占用19（16 + 3）个字节的空间大小。
- C语言中的内存分配器分配内存大小的单位都是2的n次方，为了容纳一个完整的embstr对象，最少要分配32字节的空间。稍微长一些就是64字节了。所以定义大于64字节就属于大字符串。
- 64-19 = 45.剩余可防存放字符串的空间45字节，而字符串又是以NULL结尾，占据了1个字节，所以阈值为44


## 2.5.应用场景：
- 图片得二进制存储；
- 分布式锁；
- 统计微博数；
- 统计粉丝数。.

# 三. list
在Redis3.2版本以前列表类型的内部编码有两种，由`压缩列表 zipList`和`双向链表 LinkedList`实现的；在Redis3.2版本开始对列表数据结构进行了改造，使用 `quicklist` 代替了 `ziplist` 和`linkedlist`。
## 3.1.双向链表linkedlist
![在这里插入图片描述](https://img-blog.csdnimg.cn/94631b3be05341cba91ff845e0fd8de0.png#pic_center)
```c
typedef struct listNode {
    struct listNode *prev;// 前置节点
    struct listNode *next;// 后置节点
    void *value;// 存储的数据
} listNode;
typedef struct list {
    listNode *head;// 链表头节点
    listNode *tail; // 链表尾节点
    void *(*dup)(void *ptr);// 节点值复制函数
    void (*free)(void *ptr);  // 节点值释放函数
    int (*match)(void *ptr, void *key);  // 节点值对比函数
    unsigned long len; // 链表所有节点数量
} list;
```
**链表的优势：**
- **双向**：链表节点带有前驱、后继指针获取某个节点的前驱、后继节点的时间复杂度为`0(1)`。
- **无环**: 链表为非循环链表表头节点的前驱指针和表尾节点的后继指针都指向NULL，对链表的访问以NULL为终点。
- **带表头指针和表尾指针**：通过`list`结构中的head和tail指针，获取**表头和表尾节点**的时间复杂度都为`O(1)`。
- **带链表长度计数器**:通过`list`结构的`len`属性获取节点数量的时间复杂度为`O(1)`。
- **多态**：链表节点使用void*指针保存节点的值，并且可以通过list结构的`dup、free、match`三个属性为**节点值设置类型特定函数，所以链表可以用来保存各种不同类型的值**。

**链表的缺点：**
- 链表每个节点之间的内存都是不连续的，意味着无法很好利用 CPU 缓存;
- 保存一个链表节点的值都需要一个链表节点结构头的分配，内存开销较大。

`Redis 3.0 `的` List `对象在数据量比较少的情况下，会采用**压缩列表**作为底层数据结构的实现，它的优势是**节省内存空间，并且是内存紧凑型的数据结构**。
## 3.2.压缩列表zipList
设计初衷是为了**节约内存**。使用一块**连续**的内存空间存储。**每个元素长度不同**，采用**变长编码**。
- 适用于长度较小的值，因为他是由连续的空间实现的;
- 存取的效率高，内存占用小，但由于内存是连续的，在修改的时候要重新分配内存;

当`list`对象同时满足以下两个条件是，使用的`ziplist`编码;
> 1.list对象保存的所有字符串元素长度都小于64字节
>  2.list对象保存的元素数量小于512个


### 3.2.1压缩列表结构设计
它是由**连续内存块组成的顺序型数据结构**，有点类似于数组。
![在这里插入图片描述](https://img-blog.csdnimg.cn/5b09d32837c9401a80005d2b181e50e4.png)
变长编码体现在`prevrawlensize`属性，它记录的是`prerawlen`的大小，分为两种。	
- 若前一个结点的长度`小于254`字节，那么则使用`1`字节来存储`prerawlen`;
- 若前一个结点的长度`大于等于254`字节，那么将第一个字节设置为`254`，然后接下来的`4`个字节保存实际的长度。也就是用`5个字节来表示prerawlen`的长度。

**注意：存在连锁更新问题**
假设现在存在一组压缩列表（ e1,e2,e3,e4,e5），长度都在250字节至253字节之间，采用第一种方式**保存变长编码（prerawlen为1字节）**。突然新增一新节点new在e1之前， 长度大于等于254字节，会出现：此时记录e1的前一个节点的编码方式就需要修改（多出4个字节）。e1整体长度变化，就引起之后所有的节点的prerawlen发送变化。删除节点，同理。
如下图:
![在这里插入图片描述](https://img-blog.csdnimg.cn/69be06faa9a84d88aa053084db3367a0.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/740ba80b990f4694b245ab506fe88626.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/e93f1b667e8e48dbb041d8e4fbbbaffa.png)


### 3.2.2.压缩列表的缺陷
- 空间扩展操作也就是重新分配内存，因此连**锁更新**一旦发生，就会导致压缩列表占用的**内存空间要多次重新分配，这就会直接影响到压缩列表的访问性能**;
- **不能保存太大的元素**的场景，只要节点数量足够小，即使发生连锁更新，也是能接受的;
- 不能保存太多的元素，不然访问性能会降低;

应用场景： 发布订阅，消息队列，文章列表,记录帖子的相关文章 ID，根据内容推荐相关帖子 (list)。

## 3.3.快速列表 quicklist
**双向链表**的附加空间相对太高，`prev` 和 `next` 指针就要占去 `16 个字节` ，另外每个节点的内存都是`单独分配`，会加剧`内存的碎片化`，影响内存管理效率。因此Redis3.2版本开始对列表数据结构进行了改造，使用` quicklist` 代替了` ziplist `和 `linkedlist`。
quicklist 解决办法，**通过控制每个链表节点中的压缩列表的大小或者元素个数，来规避连锁更新的问题。因为压缩列表元素越少或越小，连锁更新带来的影响就越小，从而提供了更好的访问性能。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/83fdaa1bf4b3470c9909ec659a27f689.png)

```c
struct quicklist {
	quicklistNode* head;
	quicklistNode* tail;
	long count; // 元素总数
	int nodes; // ziplist 节点的个数
	int compressDepth; // LZF 算法压缩深度
	int fill //ziplist大小设置，存放list-max-ziplist-size参数的值。
...
}

struct quicklistNode {
	quicklistNode* prev;
	quicklistNode* next;
	ziplist* zl; // 指向压缩列表
	int32 size; // ziplist 的字节总数
	int16 count; // ziplist 中的元素数量
	int2 encoding; // 存储形式 2bit，原生字节数组还是 LZF 压缩存储
...
}

```
**插入**
quicklist可以选择在**头部或者尾部**进行插入，而不管是在头部还是尾部插入数据，都包含两种情况：
- 如果头节点（或尾节点）上ziplist大小没有超过限制，那么新数据被直接插入到ziplist中。
- 如果头节点（或尾节点）上ziplist太大了，那么新创建一个quicklistNode节点，然后把这个新创建的节点插入到quicklist双向链表中。

也可以从**任意指定**的位置插入。这种在任意指定位置插入数据的操作，要比在头部和尾部的进行插入要复杂一些。
- 当插入位置所在的ziplist大小没有超过限制时，直接插入到`ziplist`中就好了；
- 当插入位置所在的ziplist大小超过了限制，但插入的位置位于`ziplist`两端，并且相邻的`quicklist`链表节点的`ziplist`大小没有超过限制，那么就转而插入到相邻的那个`quicklist`链表节点的`ziplist`中；
- 当插入位置所在的`ziplist`大小超过了限制，但插入的位置位于`ziplist`两端，并且相邻的`quicklist`链表节点的`ziplist`大小也超过限制，这时需要新创建一个`quicklist`链表节点插入。
- 对于插入位置所在的ziplist大小超过了限制的其它情况（主要对应于在ziplist中间插入数据的情况），则需要把当前ziplist分裂为两个节点，然后再其中一个节点上插入数据。
**每个 ziplist 存多少元素 ？**
`quicklist `内部默认单个 `ziplist `长度为 `8k`字节，超出了这个字节数，就会新起一个`ziplist`。`ziplist `的长度由配置参数` list-max-ziplist-size `决定。


**quicklist为什么要这样设计呢？**
**大概是基于空间和效率的一个折中**
**双向链表**:**内存开销比较大**，除了要保存数据，还要保存前后节点的指针。并且每个节点是单独的内存块，容易造成内存碎片;
**ziplist**是一块连续的内存，**节省内存**。但是当进行修改操作时，会发生**级联更新**，降低性能;

**于是结合两者优点的quicklist诞生了，但这又会带来新的问题，每个ziplist存多少元素比较合适呢？**

`ziplist`越短，内存碎片增多，影响存储效率。当一个`ziplist`只存一个元素时，`quicklist`又退化成双向链表了;
`ziplist`越长，为`ziplist`分配大的连续的内存空间难度也就越大，会造成很多小块的内存空间被浪费，当`quicklist`只有一个节点，元素都存在一个`ziplist`上时，`quicklist`又退化成`ziplist`了。

# 四.hashmap
## 4.1.散列冲突
散列函数具有确定性和不确定性。
确定性:哈希的散列值不同，那么哈希的原始输入也就不同。即:key1=key2,那hash(key1)=hash(key2)。
不确定性:同一个散列值很有可能对应多个不同的原始输入。即:key1≠key2，hash(key1)=hash(key2)。
关于散列冲突也有很多解决办法，这里简单复习两种：开放寻址法和链表法。
- **开放寻址法**:开放寻址法的核心思想是，如果出现了**散列冲突，我们就重新探测一一个空闲位置**，将其插入。比如，我们可以使用线性探测法。当我们往散列表中插入数据时，如果某个数据经过散列函数散列之后，存储位置已经被占用了，我们就从当前位置开始，依次往后查找，看是否有空闲位置，如果遍历到尾部都没有找到空闲的位置，那么我们就再从表头开始找，直到找到为止。
-  **链表法**:  链表法是一种比较常用的散列冲突解决办法**,Redis使用的就是链表法来解决散列冲突**。链表法的原理是:如果遇到冲突，他就会在原地址新建一个空间，然后以链表结点的形式插入到该空间。当插入的时候，我们只需要通过散列函数计算出对应的散列槽位，将其插入到对应链表中即可。

   

## 4.2.hashtable 
![在这里插入图片描述](https://img-blog.csdnimg.cn/974cce46736742909888310f351d4714.png)
```c
// 字典 数据结构
typedef struct dict {//管理两个dictht，主要用于动态扩容。
    //类型特定函数
    dictType *type;
    //私有数据
    void *privdata;
    // 哈希表
    dictht ht[2]; //其中一个ht[0]保存哈希表，ht[1]只会在对ht[0]进行rehash时使用
    long rehashidx; // rehash索引，当rehash不再进行是，值为 -1
    unsigned long iterators; /* number of iterators currently running */
} dict;
 
//定义一个hash桶，用来管理hashtable
typedef struct dictht {
    // hash表数组，所谓的桶
    dictEntry **table;
    // hash表大小，元素个数
    unsigned long size;
    // hash表大小掩码，用于计算索引值，值总是size -1 ,决定了一个键应该放tabl数组的哪个索引上
    unsigned long sizemask;
    // 该hash表已有的节点数量
    unsigned long used;
} dictht;
 
//hash节点
typedef struct dictEntry {
    //键
    void *key;
    // 值
    union { 
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    // 指向下一个节点，解决碰撞冲突
    struct dictEntry *next;
} dictEntry;
 
```


## 4.3.Redis rehash
rehash指的是重新**计算键的哈希值和索引值**，然后将键值对重排的过程。

**加载因子（load factor） = ht[0].used / ht[0].size。**

**扩容：** 第一个大于等于ht[0].used * 2的2^n(2的n次方幂)。
-   没有执行`BGSAVE`和`BGREWRITEAOF`指令的情况下，哈希表的`加载因子`大于等于`1`。
- 正在执行`BGSAVE`和`BGREWRITEAOF`指令的情况下，哈希表的`加载因子`大于等于`5`。

Redis这么做的目的是基于操作系统创建子进程后写时复制技术，避免不必要的写入操作。

 **收缩:** 第一个大于等于ht[0].used的2^n(2的n次方幂)。
- 加载因子小于0.1时，程序自动开始对哈希表进行收缩操作。

**实现过程**
![在这里插入图片描述](https://img-blog.csdnimg.cn/8a78db000c35404a97ccf971676ae784.png)
1.为字典的`ht[1]`散列表分配空间，这个空间的大小取决于要执行的操作以及ht[0]当前包含的键值对数量 **(即:ht[0].used的属性值)**
- 扩展操作：`ht[1]`的大小为 第一个大于等于`ht[0].used*2`的`2`的`n`次方幂。如:`ht[0].used=3`则`ht[1]`的大小为`8`;
- 收缩操作: `ht[1]`的大小为 第一个大于等于`ht[0].used`的`2`的`n`次方幂。

2.将保存在`ht[0]`中的键值对**重新计算键的散列值和索引值**，然后放到`ht[1]`指定的位置上。在字典中有一个参数`rehashidx`的值设置为`0`，表示开始**rehash**。
- 在 rehash 进行期间， 每次对字典执行**添加、删除、查找或者更新**操作时， 程序除了执行指定的操作以外， 还会顺带将 `ht[0] `哈希表在 `rehashidx` 索引上的所有键值对 `rehash `到` ht[1]` ， 当 `rehash `工作完成之后， 程序将 `rehashidx `属性的值增一。
3.将`ht[0]`包含的所有键值对都迁移到了`ht[1]`之后，这时程序将 `rehashidx `属性的值设为 `-1` ， 表示 `rehash `操作已完成，释放`ht[0]`,将`ht[1]`设置为`ht[0],`并创建一个新的`ht[1]`，哈希表为下一次`rehash`做准备。
### 4.3.1.渐进式 rehash
`rehash`到`ht[1]`，并不是一步完成的，而是分成N多步，循序渐进的完成的**（有可能存放几千万甚至上亿个key，一次性将这些键值rehash的话，可能会导致服务器在一段时间内停止服务）**。

**说明:**
- 因为在进行渐进式` rehash `的过程中，字典会同时使用 `ht[0] `和 `ht[1] `两个哈希表，所以在渐进式 `rehash` 进行期间，**字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行**。
-  在渐进式 `rehash` 执行期间，**新添加到字典的键值对一律会被保存到 ht[1] 里面**，而` ht[0] `则不再进行任何**添加操作**：这一措施保证了 **ht[0] 包含的键值对数量会只减不增**，并随着 rehash 操作的执行而最终变成空表。

## 4.4.应用场景：
- 购物车
- 计数器-
- 实时记录当天的在线的人数。
- 记录帖子的点赞数、评论数和点击数 
- 缓存近期热帖内容 (帖子内容空间占用比较大)，减少数据库压力
- 缓存用户行为历史，进行恶意行为过滤 (zset,hash)

# 五.set
set可以用`intset`或者`字典`实现。
- 当存储的元素都是整数值，且数据量不大（小于512）时使用intset。
- 不满足`intset`使用条件的情况下都使用`字典`，使用字典时把value设置为`null`。
## 5.1.intset
整数集合（intset）是Redis用于保存整数值的集合抽象数据类型，它可以保存类型为**int16_t、int32_t 或者int64_t 的整数值**，并且保证集合中不会出现重复元素。
```c
typedef struct intset{
     //编码方式
     uint32_t encoding;
     //集合包含的元素数量
     uint32_t length;
     //保存元素的数组
     int8_t contents[];
}intset;
```
　　整数集合的每个元素都是` contents `数组的一个数据项，它们按照从**小到大**的顺序排列，并且不包含任何重复项。
　　`length 属性`记录了 `contents` 数组的大小。
　　`contents` 数组声明为 int8_t 类型，但是实际上contents 数组并不保存任何 int8_t 类型的值，其真正类型有` encoding `来决定。
- `encoding `=`INTSET_ENC_INT_16`、那么contents就是一个int16_t的数组;
- `encoding` =`INTSET_ENC_INT_32`、那么contents就是一个int32_t的数组;
- ` encoding `=`INTSET_ENC_INT_64`、那么contents就是一个int64_t的数组。

**整数集合的升级操作**
　　当我们**新增的元素类型**比**原集合元素类型的长度要大**时，需要对整数集合进行升级，才能将新元素放入整数集合中。**整数集合升级的过程不会重新分配一个新类型的数组，而是在原本的数组上扩展空间，然后在将每个元素按间隔类型大小分割**，如果 `encoding `属性值为 `INTSET_ENC_INT16`，则每个元素的间隔就是 **16 位**。
　　![在这里插入图片描述](https://img-blog.csdnimg.cn/5c377fff13554ba383c42191b75fb8ca.png)

具体步骤：
　　1、根据新元素类型，**扩展整数集合底层数组的大小**，**并为新元素分配空间**。
　　2、将**底层数组现有的所有元素都转成与新元素相同类型的元素**，并将**转换后的元素放到正确的位置**，放置过程中，维持整个元素顺序都是有序的。
　　3、将**新元素添加到整数集合**中（保证有序）。
　　![在这里插入图片描述](https://img-blog.csdnimg.cn/0c49f75ded2d4f2a963406349312e7d3.png)
**整数集合升级有什么好处呢？**
- **提升灵活性**: C语言是静态类型语言,为了避免类型错误,我们通常不**会将两种不同类型的值放在同一个数据结构里面**。
 - **节约内存**:要让一个数组可以同时保存`int16_t、int32_t、int64_t`三种类型的值,最简单的做法就是直接使用`int64t`类型的数组作为**整数集合的底层实现**。不过这样一来,即使添加到整数集合里面的都是`int16_t`类型或者`int32_t`类型的值,数组都需要使用`int64_t`类型的空间去保存它们**,从而出现浪费内存的情况。**
**整数集合升级缺点：**
-  **升级过程中消耗资源**
-  不支持降级

**应该场景**
-  用户标签
- 抽奖系统
-  社交需求

# 六.zset
`Redis` 只有` Zset `对象的底层实现用到了跳表，跳表的优势是能支持平均 `O(logN) `复杂度的节点查找。
![在这里插入图片描述](https://img-blog.csdnimg.cn/ca1bf61b70624d6b9910716d54267767.png)

**zset底层实现原理：**
**元素数量小于128个 所有元素的长度都小于64字节的时候 使用ziplist数据结构**。`ziplist`占用连续内存，每项元素都是`（数据+score）`的方式连续存储，按照`score`从小到大排序。`ziplist`为了节省内存，每个元素占用的空间可以不同，对于大的数据`（long long）`，就多用一些字节来存储，而对于小的数据`（short）`，就少用一些字节来存储。因此查找的时候需要按顺序遍历。**ziplist省内存但是查找效率低**。


**为什么元素数量比较多或者成员是比较长的字符串的时候Redis要使用跳跃表来实现？**
  跳跃表在链表的基础上增加了**多级索引**以提升查找的效率，但其是一个`空间换时间`的方案，必然会带来一个问题——`索引是占内存的`。原始链表中存储的有可能是很大的对象，而索引结点只需要存储**关键值和几个指针**，并不需要存储对象，因此当节点本身比较大或者元素数量比较多的时候，其优势必然会被放大，而缺点则可以忽略。
  
## 6.1.Redis中跳跃表的实现
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/c926524dd2e242a48cebf016bc30cbc6.png)
上图展示了一个跳跃表示例,其中最左边的是` skiplist`结构,该结构包含以下属性。
- header:指向跳跃表的表头节点，表头节点的时间复杂度就为`O(1)`;
- tail:指向跳跃表的表尾节点,表尾节点的时间复杂度就为`O(1)`;
- level:记录目前跳跃表内,**层数最大的那个节点的层数**，通过这个属性可以再`O(1)`的时间复杂度内获取层高最好的节点的层数。
- length:**记录跳跃表的长度**,也即是,跳跃表目前包含节点的数量，通过这个属性，程序可以再`O(1)`的时间复杂度内返回跳跃表的长度。
- 表头：是链表的哨兵节点，不记录主体数据。

结构右方的是四个` zskiplistNode`结构,该结构包含以下属性
- **层(level):**:​ 节点中用`L1、L2、L3`等字样标记节点的各个层。​ 每个层都带有两个属性:**前进指针和跨度**。前进指针用于访问位于**表尾方向的其他节点**,而跨度则记录了**前进指针所指向节点和当前节点的距离**(跨度越大、距离越远)。​ 每次创建一个新跳跃表节点的时候,程序都根据幂次定律**随机生成一个介于1和32之间的值**作为**level**数组的大小,这个大小就是层的“高度”。
- **后退(backward)指针：**,它指向位于当前节点的前一个节点。后退指针在程序从表尾向表头遍历时使用。
- **分值(score):**:​各个节点中的`1.0、2.0`和`3.0`是节点所保存的分值。在跳跃表中,节点按各自所保存的分值从小到大排列。
- **成员对象(oj):**​ 各个节点中的`o1、o2`和`o3`是节点所保存的成员对象。在**同一个跳跃表中,各个节点保存的成员对象必须是唯一的**。

## 6.2.操作

![在这里插入图片描述](https://img-blog.csdnimg.cn/074e2f547caa4320928c45953fe55ade.png)
**查询过程**
查找一个跳表节点的过程时，跳表会从头节点的最高层开始，逐一遍历每一层。在遍历某一层的跳表节点时，会用跳表节点中的 SDS 类型的元素和元素的权重来进行判断，共有两个判断条件：
- **如果当前节点的权重「小于」要查找的权重时，跳表就会访问该层上的下一个节点。**
- **如果当前节点的权重「等于」要查找的权重时，并且当前节点的 SDS 类型数据「小于」要查找的数据时，跳表就会访问该层上的下一个节点。**

如果上面两个条件都不满足，或者下一个节点为空时，跳表就会使用目前遍历到的节点的 level 数组里的下一层指针，然后沿着下一层指针继续查找，这就相当于跳到了下一层接着查找。
![在这里插入图片描述](https://img-blog.csdnimg.cn/06e5f9225b914d3590872bcb8d51b456.png)查找元素：**abcd，权重：4**的节点，查找的过程是这样的：

- 先从头节点的最高层开始，L2 指向了**元素：abc，权重：3**节点，这个节点的权重比要查找节点的小，所以要访问该层上的下一个节点；但是该层的下一个节点是**空节点**，于是就会跳到下一层去找，也就是` leve[1]`;
- 元素：**abc，权重：3**节点的 leve[1] 的下一个指针指向了**元素：abcde，权重：4**的节点，然后将其和要查找的节点比较。虽然**元素：abcde，权重：4**的节点的权重和要查找的权重相同，但是当前节点的 **SDS 类型数据大于要查找的数据**，所以会继续跳到**元素：abc，权重：3**节点的下一层去找，也就是 **leve[0]**；
- **元素：abc，权重：3**节点的 **leve[0]** 的下一个指针指向了**元素：abcd，权重：4**的节点，该节点正是要查找的节点，查询结束。

## 6.3.跳表节点层数设置
跳表的相邻两层的节点数量的比例会影响跳表的查询性能。
![在这里插入图片描述](https://img-blog.csdnimg.cn/8c5284ee0b024aa6a70c965d7b68e587.png)
这时，如果想要查询`节点 6`，那基本就跟链表的查询复杂度一样，就需要在第一层的节点中依次顺序查找，复杂度就是 `O(N)` 了。所以，为了降低查询复杂度，我们就需要维持相邻层结点数间的关系。
**跳表的相邻两层的节点数量最理想的比例是 2:1，查找复杂度可以降低到 O(logN)。**
Redis 则采用一种巧妙的方法是，**跳表在创建节点的时候，随机生成每个节点的层数**，并没有严格维持相邻两层的节点数量比例为 2 : 1 的情况。
    具体的做法是，跳表在创建节点时候，会生成范围为[0-1]的一个随机数，**如果这个随机数小于 0.25，那么层数就增加 1 层，然后继续生成下一个随机数，直到随机数的结果大于 0.25 结束，最终确定该节点的层数。**

## 6.4.参考REDIS中为什么使用跳表
理论上来讲，查找、插入、删除以及迭代输出有序序列这几个操作，红黑树也可以完成，时间复杂度和跳表是一样的。
**redis使用跳表而不是红黑树的原因**
- **在做范围查找的时候**:按照区间查找数据这个操作，红黑树的效率没有跳表高。跳表可以在 O(logn)时间复杂度定位区间的起点，然后在原始链表中顺序向后查询就可以了。
- **从算法实现难度上来比较** : 相比于红黑树，跳表还具有代码更容易实现、可读性好、不容易出错、更加灵活等优点。
- **从性能上来比较:**插入、删除时跳表只需要调整少数几个节点，红黑树需要颜色重涂和旋转，开销较大。
- **从内存占用上来比较**，跳表比平衡树更灵活一些。
## 6.5.应用场景
- 排行榜系统例如学生成绩的排名。
- 某视频(博客等)网站的用户点赞、播放排名。
- 电商系统中商品的销量排名等。
