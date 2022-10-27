---
title: 别问我内存满了怎么办
tags:
  - redis
categories:
	- redis
toc: true
toc_number: true
cover:  https://img-blog.csdnimg.cn/e642cbd4bb8b4182a58da81cf8636d87.png
---

# 前言
在我们平时开发过程中，会有一些 `bool `型数据需要存取，比如用户一年的签到记录，签了是 `1`，没签是 `0`，要记录 `365` 天。如果使用普通的` key/value`，每个用户要记录` 365`个，当用户上亿的时候，需要的存储空间是惊人的。为了解决这个问题，Redis 提供了**位图**数据结构，这样每天的签到记录只占据一个位，`365` 天就是` 365` 个位，`46`个字节 (一个稍长一点的字符串) 就可以完全容纳下，这就大大节约了存储空间。
# BitMap
即位图，是一串连续的`二进制数组`（0和1），可以通过`偏移量（offset）`定位元素。`BitMap`通过最小的单位`bit`来进行`0|1`的设置，表示某个元素的值或者状态，时间复杂度为O(1)。由于 `bit `是计算机中最小的单位，使用它进行储存将非常节省空间，特别适合一些`数据量大`且使用`二值统计`的场景。
![在这里插入图片描述](https://img-blog.csdnimg.cn/3ee6b236b5c547b7950a6a860e66880e.png)
**`bitmap `并不是一种数据结构，实际上它就是字符串，但是可以对字符串的位进行操作。**

从以下结果可以看出 Bitmaps实际上存的就是String

```
127.0.0.1:6379> set hello big
OK
127.0.0.1:6379> getbit hello 0
(integer) 0
127.0.0.1:6379> getbit hello 1
(integer) 1
127.0.0.1:6379> getbit hello 2
(integer) 1
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/e642cbd4bb8b4182a58da81cf8636d87.png)

# 常用命令
## setbit
setbit key offset value：  //给对应的位设置值

比如今天有用户`3、8、`登录了网站，
```
setbit user:login:200517 3 1
setbit user:login:200517 8 1

```
**开发提示**：很多应用`id`都不是从`1`开始，有许多是从指定数字开始的，比如`1001`、`10001`开始。对于这些，我们在设置的时候可以先减去初始值，防止浪费空间。

## getbit

```
getbit key offset    //用于获取Redis中指定key对应的值，中对应offset的bit
```

如果我想知道今天`8`号用户和`10`号用户是否登录过，则

```
127.0.0.1:6379> getbit user:login:200517 8
(integer) 1
127.0.0.1:6379> getbit user:view:200517 10
(integer) 0
```

## bitcount 

```
bitcount key [start end]   //用于统计字符串被设置为1的bit数
```

我想知道今天有多少用户登陆过了，则

```
127.0.0.1:6379> bitcount user:login:200517
(integer) 2
```

## Bitmaps

```
bitop and/or/xor/not  destkey   key  [key …]    用于对多个key求逻辑与/逻辑或/逻辑异或/逻辑非
```
`bitop`命令可以对多个`bitmaps`做交集（`and`)、并集（`or`）、非（`not`）、异或（`xor`），并将操作结果存放在destkey中。

如果想知道连续三天都登陆过的用户。

```
127.0.0.1:6379> bitop and three:and user:login:200517 user:login:200518 user:view:200519
127.0.0.1:6379> bitcount three:and
(integer) 2
```
# 应用场景
通过` bitcount`可以很快速的统计，比传统的关系型数据库效率高很多。
- **统计年活跃用户数量**
  - 用户的`ID`作为`offset`，当用户在一年内访问过网站，就将对应`offset`的`bit`值设置为“1”；
  - 通过`bitcount `来统计一年内访问过网站的用户数量
- **统计三天内活跃用户数量**
  - 时间字符串作为`key`，比如 `“190108：active“ “190109：active”“190110：active” `；
  - 用户的`ID`就可以作为`offset`，当用户访问过网站，就将对应`offset`的`bit`值设置为“1”；
 - **统计在线人数** 设置在线`key`：“online：active”，当用户登录时，通过`setbit`设置
   - ` bitmap`的优势，以统计活跃用户为例
   - 每个用户id占用空间为`1bit`，消耗内存非常少，存储`1亿`用户量只需要`12.5M`。
  - **签到统计**
    -  在签到打卡的场景中，我们只用记录签到,就将对应`offset`的`bit`值设置为“1”；
    - 每个用户一天的签到用 `1 `个 `bit `位就能表示，一个月的签到情况用` 30` 个 `bit `位就可以，而一年的签到也只需要用` 365 `个` bit` 位，根本不用太复杂的集合类型。
- **布隆过滤器**

**注意：最好给` Bitmap `设置过期时间，让` Redis` 删除过期的打卡数据，节省内存。**