---
title: 你是不是认为Singleflight是单例模式
tags:
  - golang 
  - 并发
categories:
	- golang

toc: true
toc_number: true
cover:  /img/post/6833939bly1giclhfehz7j20zk0m8u0x.jpg
---

### 一.为什么需要Singleflight?

一般情况下我们在写对外的服务的时候都会有一层 cache 作为缓存，用来减少底层数据库的压力，但是在遇到例如 redis 抖动或者其他情况可能会导致大量的 cache 失效出现，导致大量请求打入数据库，造成数据库压力。

### 二.使用场景

这个库的主要作用就是: **将一组相同的请求合并成一个请求，实际上只会去请求一次，然后对所有的请求返回相同的结果。**

主要场景：用在对指定资源频繁操作(高并发下的缓存击穿)

### 三.原理解析

#### 1.内部结构介绍

```go
//数据结构是用来记录key相关数据，即请求当前key的个数和请求结果
type call struct {
	wg sync.WaitGroup
    // 请求返回结果，会在sync.WaitGroup为Done的时候执行
	val interface{}
	err error
    // 可以理解为请求key的个数，每增加一个请求，则值加1 
	dups  int
    // 存储所有key对应的 Result{}, 一个请求对应一个 Result
	chans []chan<- Result
}


// SingleFlight 的主结构体 
type Group struct {
	mu sync.Mutex       // 并发锁
	m  map[string]*call // 懒加载 用来存储key与请求的关系
}
    // Do 执行函数, 对同一个 key 多次调用的时候，在第一次调用没有执行完的时候
	// 只会执行一次 fn 其他的调用会阻塞住等待这次调用返回
	// v, err 是传入的 fn 的返回值
	// shared 表示当前返回结果是否为多个请求结果
    func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool)
	// DoChan 和 Do 类似，只是 DoChan 返回一个 channel，也就是同步与异步的区别
	func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result
 
    // Forget 用于通知 Group 删除某个key这样后面继续这个key的调用的时候就不会在阻塞等待了
	func (g *Group) Forget(key string)
```

#### 2.常用Do方法解析（Dochan逻辑类似）

```go
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
    g.mu.Lock()
    if g.m == nil {
      g.m = make(map[string]*call)
    }
    if c, ok := g.m[key]; ok {
      // 存在相同的key, 增加计数
      c.dups++
      g.mu.Unlock()
      c.wg.Wait() //等待这个key对应的fn调用完成
      return c.val, c.err, true // 返回fn调用的结果
    }
    c := new(call) // 不存在key, 是第一个请求, 创建一个call结构体
    c.wg.Add(1)
    g.m[key] = c //加入到映射表中
    g.mu.Unlock()
    g.doCall(c, key, fn) // 调用方法
    return c.val, c.err, c.dups > 0
  }


```

SingleFlight 定义一个call结构体，每个结构体都保存了fn调用对应的信息。

1.每次调用Do方法都会先去获取互斥锁，随后判断在映射表里是否已经有Key对应的`fn`函数调用信息的`call`结构体。

2.当key不存在时，证明是这个Key的第一次请求，那么会初始化一个`call`结构体指针，增加`SingleFlight`内部持有的`sync.WaitGroup`计数器到1。释放互斥锁，然后阻塞的等待doCall方法执行`fn`函数的返回结果

3.当key存在时，增加`call`结构体内代表`fn`重复调用次数的计数器`dups`，释放互斥锁，然后使用`WaitGroup`等待`fn`函数执完成。

4.卡在第2步的方法得到执行，返回结果

5.`call`结构体的`val` 和 `err` 两个字段只会在 `doCall`方法中执行`fn`有返回结果后才赋值。

### 四.简单使用

#### 1.示例介绍

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"

	"golang.org/x/sync/singleflight"
)

var count int32

//模拟调用1000次 实际1000次
func getArticle(id int) (article string, err error) {
	// 假设这里会对数据库进行调用, 模拟不同并发下耗时不同
	atomic.AddInt32(&count, 1)
	fmt.Println(count)
	time.Sleep(time.Duration(count) * time.Millisecond)

	return fmt.Sprintf("article: %d", id), nil
}

//模拟调用1000次，实际接近1次
func singleFlightGetArticle(sg *singleflight.Group, id int) (string, error) {
	v, err, _ := sg.Do(fmt.Sprintf("%d", id), func() (interface{}, error) {
		return getArticle(id)
	})
	return v.(string), err
}
 // 模拟1000请求访问
func main() {
	var (
		wg  sync.WaitGroup
		now = time.Now()
		n   = 1000
		sg  = &singleflight.Group{}
	)

	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			res, _ := singleFlightGetArticle(sg, 1)
			//res, _ := getArticle(1)
			if res != "article: 1" {
				panic("err")
			}
			wg.Done()
		}()
	}

	wg.Wait()
	fmt.Printf("同时发起 %d 次请求，耗时: %s", n, time.Since(now))
}
-----------------------------
//getArticle 同时发起 1000 次请求，耗时: 1.0786589s 相当于1000个相同请求打入数据库
//singleFlightGetArticle方法 同时发起 1000 次请求，耗时: 4.2138ms 
```

#### 2.泛型版本

Do 方法返回的 v 是 interface{}，我们做了类型断言：`v.(string)`。

 Go1.18 有了泛型，可以有泛型版本的 singleflight，不需要做类型断言了。GitHub 已经有人实现并开源：https://github.com/marwan-at-work/singleflight。

可以使用泛型的singleflight不需要做类型断言

### 五.注意事项

#### 1.一个阻塞，全员等待

```go
//修改上面的singleFlightGetArticle方法，新增select阻塞
func singleFlightGetArticle2(sg *singleflight.Group, id int) (string, error) {
	v, err, _ := sg.Do(fmt.Sprintf("%d", id), func() (interface{}, error) {
		// 模拟阻塞
		select {}
		return getArticle2(id)
	})
 
	return v.(string), err
}

```

//结果：导致整个程序阻塞

**解决方法: context + DoChan + select**

```go
// 调用一个含超时限制的context
func singleflightGetArticle(ctx context.Context, sg *singleflight.Group, id int) (string, error) {
	result := sg.DoChan(fmt.Sprintf("%d", id), func() (interface{}, error) {
		// 模拟出现问题，hang 住
		select {}
		return getArticle(id)
	})
 
	select {
	case r := <-result:
		return r.Val.(string), r.Err
	case <-ctx.Done():
		return "", ctx.Err()
	}
}
```

#### 2.一个出错，全部出错

因为singleflight就是这么设计的。如果1s尝试10次，但是用了singleflight之后只能尝试一次。只要出错这段时间内的所有请求都会收到影响。

### 六.总结

1.Do方法和DoChan方法，分别提供了同步和异步的调用方式，这让我们使用起来也更加灵活；

2.如果调用Forget方法 可以通知 Group 在持有的映射表中删除某个键，接下来对该键的调用就不会等待前面的函数返回了；

3.一旦调用的函数返回了错误，所有在等待的 Goroutine 也都会接收到同样的错误；

