---
title: zookeeper学习（三）go-zookeeper
tags:
  - zookeeper 
  - 分布式
categories:
	- zookeeper
description: 
toc: true
toc_number: true
cover:  /img/post/98bf9a3c223248f2aa6ca0e66ec0df38.png
---

## 一、Zookeeper入门
### 1.1. Zookeeper简介
Zookeeper是一个分布式数据库(程序协调服务)，Hadoop子项目；以树状方式维护节点方数据的增、删、改、查；通过监听可以获取相应消息事件；
节点类型：
- 持久节点：一直存储在服务器上(0)
- 临时节点：会话失效、节点自动清理(FlagEphemeral)
- 顺序节点：节点创建时自动分配序列号(FlagSequence)


## 二.启动zookeeper
![在这里插入图片描述](https://img-blog.csdnimg.cn/4a2cf8d005f2420a94bc5a627febad64.png)
## 三.核心包
go-zookeeper

```bash
go get github.com/samuel/go-zookeeper/zk
文档地址: http://godoc.org/github.com/samuel/go-zookeeper/zk
```
## 四.Golang实现Zookeeper核心功能
### 4.1 建立连接

```go
func GetConnect(zkList []string) (conn *zk.Conn) {
	// 创建监听的option，用于初始化zk
	conn, _, err := zk.Connect(zkList, 10*time.Second)
	if err != nil {
		fmt.Println(err)
	}
	return
}
```
### 4.2创建节点

```go
func Create(conn *zk.Conn, path string, data []byte, flags int32, acl int32) (val string, err error) {
	//flags有4种取值：
	//0:永久，除非手动删除
	//zk.FlagEphemeral = 1:短暂，session断开则改节点也被删除
	//zk.FlagSequence  = 2:会自动在节点后面添加序号
	//3:Ephemeral和Sequence，即，短暂且自动添加序号
	// 获取访问控制权限
	var acls []zk.ACL
	if acl == 0 {
		/**
		PermRead = 1 << iota   1
		PermWrite              2
		PermCreate             4
		PermDelete             8
		PermAdmin             16
		PermAll = 0x1f        31
		*/
		acls = zk.WorldACL(zk.PermAll)
	} else {
		acls = zk.WorldACL(acl)
	}

	val, err = conn.Create(path, data, flags, acls)
	return
}

var (
	zkList = []string{"127.0.0.1:2181"}
	path   = "/root"
)

func TestCreate(t *testing.T) {
	conn := GetConnect(zkList)

	defer conn.Close()
	//创建节点
	val, err := Create(conn, path, []byte("root value"), 0, 31)
	if err != nil {
		fmt.Printf("创建失败: %v\n", err)
		return
	}
	fmt.Printf("创建: %s 成功", val)
}
```
### 4.3查询节点

```go
// 查
func Get(conn *zk.Conn, path string) (dataStr string, stat *zk.Stat, err error) {
	data, stat, err := conn.Get(path)
	if err != nil {
		return "", nil, err
	}
	return string(data), stat, err
}

func TestGet(t *testing.T) {
	conn := GetConnect(zkList)

	defer conn.Close()
	//查询节点
	val, _, err := Get(conn, path)
	if err != nil {
		fmt.Printf("查询%s失败, err: %v\n", path, err)
		return
	}
	fmt.Printf("%s 的值为 %s\n", path, val)
}

```

 ### 4.4 节点是否存在
 

```go
func Exists(conn *zk.Conn, path string) (exist bool, err error) {
	exist, _, err = conn.Exists(path)
	return
}

func TestExist(t *testing.T) {
	conn := GetConnect(zkList)

	defer conn.Close()
	//是否存在
	val, err := Exists(conn, path)
	if err != nil {
		fmt.Printf("查询%s失败, err: %v\n", path, err)
		return
	}
	if val {
		fmt.Printf("%s 存在\n", path)
	} else {
		fmt.Printf("%s 不存在\n", path)
	}
}

```
### 4.5删除节点

```go
//删除 cas支持
func Del(conn *zk.Conn, path string) (err error) {
	_, sate, _ := Get(conn, path)
	fmt.Println(sate)
	err = conn.Delete(path, sate.Version)
	return err
}

func TestDel(t *testing.T) {
	conn := GetConnect(zkList)

	defer conn.Close()
	// 删除
	err := Del(conn, path)
	if err != nil {
		fmt.Printf("数据删除失败: %v\n", err)
		return
	}
	fmt.Println("数据删除成功")
}

```

### 4.6 修改节点内容

```go
// 改 CAS支持
// 可以通过此种方式保证原子性
func Modify(conn *zk.Conn, path string, newData []byte) (sate *zk.Stat, err error) {
	_, sate, _ = conn.Get(path)
	fmt.Println(sate)
	sate, err = conn.Set(path, newData, sate.Version)
	return
}


func TestModify(t *testing.T) {
	conn := GetConnect(zkList)

	defer conn.Close()
	newData := []byte("hello zookeeper")
	stat, err := Modify(conn, path, newData)
	if err != nil {
		fmt.Printf("数据修改失败: %v\n", err)
		return
	}
	fmt.Printf("数据修改成功,stat %v\n", stat)
}

```
### 4.7获取目录信息

```go
func Children(conn *zk.Conn, path string) (data []string, err error) {
	data, _, err = conn.Children(path)
	return
}

func TestChildren(t *testing.T) {
	conn := GetConnect(zkList)

	defer conn.Close()
	data, err := Children(conn, "/")
	if err != nil {
		fmt.Printf("获取数据失败: %v\n", err)
		return
	}
	fmt.Printf("获取数据成功,data %v\n", data)
}

```

## 五.watch

```go
func callback(event zk.Event) {
    fmt.Println(">>>>>>>>>>>>>>>>>>>")
    fmt.Println("path:", event.Path)
    fmt.Println("type:", event.Type.String())
    fmt.Println("state:", event.State.String())
    fmt.Println("<<<<<<<<<<<<<<<<<<<")
}
 
func ZKOperateWatchTest() {
    fmt.Printf("ZKOperateWatchTest\n")
 
    option := zk.WithEventCallback(callback)
    var hosts = []string{"localhost:2181"}
    conn, _, err := zk.Connect(hosts, time.Second*5, option)
    if err != nil {
        fmt.Println(err)
        return
    }
    defer conn.Close()
 
    var path1 = "/zk_test_go1"
    var data1 = []byte("zk_test_go1_data1")
    exist, s, _, err := conn.ExistsW(path1)
    if err != nil {
        fmt.Println(err)
        return
    }
 
    fmt.Printf("path[%s] exist[%t]\n", path1, exist)
    fmt.Printf("state:\n")
    fmt.Printf("%s\n", ZkStateStringFormat(s))
 
    // try create
    var acls = zk.WorldACL(zk.PermAll)
    p, err_create := conn.Create(path1, data1, zk.FlagEphemeral, acls)
    if err_create != nil {
        fmt.Println(err_create)
        return
    }
    fmt.Printf("created path[%s]\n", p)
 
    time.Sleep(time.Second * 2)
 
    exist, s, _, err = conn.ExistsW(path1)
    if err != nil {
        fmt.Println(err)
        return
    }
 
    fmt.Printf("path[%s] exist[%t] after create\n", path1, exist)
    fmt.Printf("state:\n")
    fmt.Printf("%s\n", ZkStateStringFormat(s))
 
    // delete
    conn.Delete(path1, s.Version)
 
    exist, s, _, err = conn.ExistsW(path1)
    if err != nil {
        fmt.Println(err)
        return
    }
    fmt.Printf("path[%s] exist[%t] after delete\n", path1, exist)
    fmt.Printf("state:\n")
    fmt.Printf("%s\n", ZkStateStringFormat(s))
}
```
不止会打印watch监听的节点信息，还有打印session会话的状态
