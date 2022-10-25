---
title: gf_lua脚本
tags:
  - golang 
  - redis
  - goframe
categories:
	- golang
  
toc: true
toc_number: true
cover:  /img/post/6833939bly1gicitht3xtj20zk0m8k5v.jpg
---


### 前言
Lua是一个高效的轻量级脚本语言。

**Redis中使用 Lua 的好处**
- 减少网络开销，在 Lua脚本 中可以把多个命令放在同一个脚本中运行；
- 原子操作，Redis 会将整个脚本作为一个整体执行，中间不会被其他命令插入（编写脚本过程中无需担心会出现竞态条件）
- 复用性，客户端发送的脚本会永远存储在 Redis中，意味着其他客户端可以复用这一脚本。

**Redis Lua脚本与事务**
从定义上来说， Redis 中的脚本本身就是一种事务， 所以任何在事务里可以完成的事， 在脚本里面也能完成。 并且一般来说， 使用脚本要来得更简单，并且速度更快。但是lua支持缓存，可以复用脚本，这个是原来的事务所没有的。

使用事务时可能会遇上以下两种错误：

- 事务在执行 EXEC 之前，入队的命令可能会出错。比如说，命令可能会产生语法错误；
- 命令可能在 EXEC 调用之后失败。举个例子，事务中的命令可能处理了错误类型的键，比如将列表命令用在了字符串键上面，诸如此类。
对于发生在 EXEC 执行之前的错误，客户端以前的做法是检查命令入队所得的返回值：如果命令入队时返回 QUEUED ，那么入队成功；否则，就是入队失败。如果有命令在入队时失败，那么大部分客户端都会停止并取消这个事务。


Redis 相关命令
**1.EVAL**

```lua
---   脚本   key个数    key      附件参数
EVAL script numkeys key [key …] arg [arg …]
--- key [key ...] ： 从 EVAL 的第三个参数开始算起，表示在脚本中所用到的那些 Redis键(key)，这些键名参数可在 Lua 中通过全局变量 KEYS 数组，用 1 为基址的形式访问( KEYS[1] ， KEYS[2] ，以此类推)。
--- arg [arg ...] ： 附加参数，在 Lua 中通过全局变量 ARGV 数组访问，访问的形式 ARGV[1]、ARGV[2] 
```
每次执行 Eval命令时 Redis 都会将脚本的 SHA1 摘要加入到脚本缓存中，以便下次客户端可以使用 EVALSHA 命令调用该脚本。（EVALSHA+script load)

**2.EVALSHA**
EVALSHA 命令的表现如下：
```lua
---     加密后的   key个数    key      附件参数
evalsha sha1 numkeys key [key …] arg [arg …]
```

  EVAL 命令要求在每次执行脚本的时候都发送一次脚本主体。Redis 有一个内部的缓存机制，因此它不会每次都重新编译脚本，不过在很多场合，付出无谓的带宽来传送脚本主体并不是最佳选择。为了减少带宽的消耗， Redis 实现了 EVALSHA 命令，它的作用和 EVAL 一样，都用于对脚本求值，但它接受的第一个参数不是脚本，而是脚本的 SHA1 校验和(sum)。

**3.SCRIPT FLUSH**

```lua
script flush
```
清除Redis服务端所有 Lua 脚本缓存


**4.SCRIPT EXISTS**
```lua
SCRIPT EXISTS sha1 [sha1 …] 
```
给定一个或多个脚本的 SHA1 校验和，返回一个包含 0或 1 的列表，表示校验和所指定的脚本是否已经被保存在缓存当中

**5.SCRIPT LOAD**
将脚本 script 添加到Redis服务器的脚本缓存中，并不立即执行这个脚本，而是会立即对输入的脚本进行求值。并返回给定脚本的 SHA1 校验和。如果给定的脚本已经在缓存里面了，那么不执行任何操作。

```lua
      ------脚本
SCRIPT LOAD script 

```

**6.SCRIPT KILL**
杀死当前正在运行的 Lua 脚本，当且仅当这个脚本没有执行过任何写操作时，这个命令才生效。 这个命令主要用于终止运行时间过长的脚本，比如一个因为 BUG 而发生无限 loop 的脚本，诸如此类。
假如当前正在运行的脚本已经执行过写操作，那么即使执行SCRIPT KILL，也无法将它杀死，因为这是违反 Lua 脚本的原子性执行原则的。在这种情况下，唯一可行的办法是使用SHUTDOWN NOSAVE命令，通过停止整个 Redis 进程来停止脚本的运行，并防止不完整(half-written)的信息被写入数据库中。

### Lua中执行redis命令
在 Lua 脚本中，可以使用两个不同函数来执行 Redis 命令，它们分别是：

- redis.call()
- redis.pcall()

这两个函数的唯一区别在于它们使用不同的方式处理执行命令所产生的错误。



### go使用lua脚本
以goframe框架中gredis为例。


```go
type scriptEVALSha struct {
	r *gredis.Redis
	m sync.Map
}

var ScriptEVALSha = &scriptEVALSha{}
var path = "resource/scripts/"

func new(r *gredis.Redis) *scriptEVALSha {
	if r == nil {
		return nil
	}
	return &scriptEVALSha{
		r: r,
		m: sync.Map{},
	}
}

func init() {
	ctx := context.Background()
	ScriptEVALSha = new(g.Redis())
	list, err := gfile.ScanDirFile(path, "*.lua", true)
	if err != nil {
		fmt.Println(err.Error())
	}
	for _, v := range list {
		sha, err := ScriptEVALSha.registerScriptFile(ctx, v)
		if err != nil {
			fmt.Println(err)
		} else {
			fmt.Println(sha)
		}
	}
}
```


2.注册脚本

```go

func getNameByFilePath(ctx context.Context, filePath string) (fileName string) {
	//获取文件名
	base := filepath.Base(filePath)
	if i := strings.LastIndexByte(base, '.'); i != -1 {
		fileName = base[:i]
	} else {
		fileName = base
	}
	return
}

// RegisterScriptFile 注册{scriptsPath}/{filename}脚本文件
func (dr *scriptEVALSha) registerScriptFile(ctx context.Context, filePath string) (sha string, err error) {
	fileName := getNameByFilePath(ctx, filePath)
	g.Log().Debugf(ctx, "script: fileName = %s ", fileName)
	file, err := os.Open(filePath)
	if err != nil {
		g.Log().Error(ctx, "open file failed:", err, filePath)
		return "", err
	}
	defer file.Close()
	fileBuffer, err := io.ReadAll(file)
	if err != nil {
		g.Log().Error(ctx, "read file failed:", err, filePath)
		return "", err
	}
	script := string(fileBuffer)
	// 缓存脚本,获取校验和
	if sha, err = dr.scriptLoad(ctx, script); err != nil && sha == "" {
		return "", errors.New(fmt.Sprintf("script load `%s` failed:", err))
	}
	g.Log().Debugf(ctx, "script: fileName = %s sha = %s", fileName, sha)

	// 注册脚本key和校验和，如果fileName已注册，则返回错误
	res, loaded := dr.m.LoadOrStore(fileName, sha)
	if loaded {
		err = errors.New(fmt.Sprintf("script fileName = %s has registered. exists sha1 = %s", fileName, res))
		return "", err
	}
	sha = res.(string)
	g.Log().Debugf(ctx, "register script fileName: key = %s, sha = %s", fileName, sha)
	return sha, err
}
//缓存脚本  sha1加密
func (dr *scriptEVALSha) scriptLoad(ctx context.Context, script string) (string, error) {
	var (
		doArgs = []interface{}{"LOAD", script}
	)
	if v, err := dr.r.Do(ctx, "SCRIPT", doArgs...); err != nil {
		return "", err
	} else {
		return v.String(), nil
	}
}
```

4.执行脚本

```go
//fileName 执行的脚本名
func (dr *scriptEVALSha) evalSha1CmdScriptKey(ctx context.Context, fileName string, keys []string, argv ...interface{}) (*gvar.Var, error) {
	var (
		//第一个参数是执行脚本 第二参数是key的个数
		doArgs = make([]interface{}, 2+len(keys), 2+len(keys)+len(argv))
	)
	g.Log().Debugf(ctx, "keys: %v, args: %v", keys, argv)
	// 读取脚本对应的SHA1校验和
	sha, ok := dr.m.Load(fileName)
	if !ok {
		return nil, errors.New(fmt.Sprintf("fileName = %s have not registered", fileName))
	}
	doArgs[0] = sha
	doArgs[1] = len(keys)

	for i, k := range keys {
		doArgs[2+i] = k
	}
	doArgs = append(doArgs, argv...)
	g.Log().Debugf(ctx, "eval script key: `evalsha %v`", doArgs)
	return dr.r.Do(ctx, "evalsha", doArgs...)
}

func (dr *scriptEVALSha) FlushScriptKey(ctx context.Context) (string, error) {
	var (
		doArgs = []interface{}{"FLUSH"}
		newMp  = sync.Map{}
	)
	if v, err := dr.r.Do(ctx, "SCRIPT", doArgs...); err != nil {
		return "", err
	} else {
		dr.m = newMp
		return v.String(), nil
	}
}

func (dr *scriptEVALSha) GetAndDel(ctx context.Context, key string) (*gvar.Var, error) {
	return dr.evalSha1CmdScriptKey(ctx, "get-and-del", []string{key})
}

func (dr *scriptEVALSha) GetAndSet(ctx context.Context, key string, v ...interface{}) (*gvar.Var, error) {
	return dr.evalSha1CmdScriptKey(ctx, "get-and-set", []string{key}, v)
}
```
其他

```go
// Map 获取所有键值对
func (dr *scriptEVALSha) Map() map[string]string {
	var m = make(map[string]string)
	dr.m.Range(func(key, value interface{}) bool {
		if k, ok := key.(string); ok {
			m[k] = value.(string)
		}
		return true
	})

	return m
}

// Keys 获取所有key
func (dr *scriptEVALSha) Keys() []string {
	var keys []string
	dr.m.Range(func(key, _ interface{}) bool {
		if k, ok := key.(string); ok {
			keys = append(keys, k)
		}
		return true
	})

	return keys
}

```
