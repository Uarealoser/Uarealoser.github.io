---
layout:     post
title:      golang net/rpc 2
subtitle:   golang net/rpc 2
date:       2021-04-26
author:     Uarealoser
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - rpc
---


# 客户端RPC的实现原理

Go语言的RPC库最简单的使用方式是通过Client.Call方法进行同步阻塞调用，该方法的实现如下：

```go
func (client *Client) Call(
    serviceMethod string, args interface{},
    reply interface{},
) error {
    call := <-client.Go(serviceMethod, args, reply, make(chan *Call, 1)).Done
    return call.Error
}
```

首先通过Client.Go方法进行一次异步调用，返回一个表示这次调用的Call结构体。然后等待Call结构体的Done管道返回调用结果。

我们也可以通过Client.Go方法异步调用前面的HelloService服务：

```go
func doClientWork(client *rpc.Client) {
    helloCall := client.Go("HelloService.Hello", "hello", new(string), nil)

    // do some thing

    helloCall = <-helloCall.Done
    if err := helloCall.Error; err != nil {
        log.Fatal(err)
    }

    args := helloCall.Args.(string)
    reply := helloCall.Reply.(string)
    fmt.Println(args, reply)
}
```

在异步调用命令发出后，一般会执行其他的任务，因此异步调用的输入参数和返回值可以通过返回的Call变量进行获取。

执行异步调用的Client.Go方法实现如下：

```go
func (client *Client) Go(
    serviceMethod string, args interface{},
    reply interface{},
    done chan *Call,
) *Call {
    call := new(Call)
    call.ServiceMethod = serviceMethod
    call.Args = args
    call.Reply = reply
    call.Done = make(chan *Call, 10) // buffered.

    client.send(call)
    return call
}
```

首先是构造一个表示当前调用的call变量，然后通过client.send将call的完整参数发送到RPC框架。**client.send方法调用是线程安全的，因此可以从多个Goroutine同时向同一个RPC链接发送调用指令。**

当调用完成或者发生错误时，将调用call.done方法通知完成：

```go
func (call *Call) done() {
    select {
    case call.Done <- call:
        // ok
    default:
        // We don't want to block here. It is the caller's responsibility to make
        // sure the channel has enough buffer space. See comment in Go().
    }
}
```

# 基于RPC实现Watch功能

这里我们可以尝试通过RPC框架实现一个基本的Watch功能。如前文所描述，因为client.send是线程安全的，我们也可以通过在不同的Goroutine中同时并发阻塞调用RPC方法。通过在一个独立的Goroutine中调用Watch函数进行监控。

为了便于演示，我们计划通过RPC构造一个简单的内存KV数据库。首先定义服务如下：

```go
type KVStoreService struct {
    m      map[string]string
    filter map[string]func(key string)
    mu     sync.Mutex
}

func NewKVStoreService() *KVStoreService {
    return &KVStoreService{
        m:      make(map[string]string),
        filter: make(map[string]func(key string)),
    }
}
```

其中m成员是一个map类型，用于存储KV数据。filter成员对应每个Watch调用时定义的过滤器函数列表。而mu成员为互斥锁，用于在多个Goroutine访问或修改时对其它成员提供保护。

然后就是Get和Set方法：

```go
func (kv *KVStoreService)Get(key string,value *string)error{
	kv.mu.RLock()
	defer kv.mu.RUnlock()

	if v,ok:= kv.m[key];ok{
		*value = v
		return nil
	}

	return fmt.Errorf("not found:%s",key)
}

func (kv *KVStoreService)Set(kventry [2]string,reply *struct{})error{
	kv.mu.Lock()
	defer kv.mu.Unlock()

	key,value:=kventry[0],kventry[1]

	if oldValue:=kv.m[key];oldValue!=value{
		for _,f:=range kv.filter{
			f(key)
		}
	}

	kv.m[key] = value
	return nil
}
```

在Set方法中，输入参数是key和value组成的数组，用一个匿名的空结构体表示忽略了输出参数。当修改某个key对应的值时会调用每一个过滤器函数。

而过滤器列表在Watch方法中提供：

```go
func (kv *KVStoreService)Watch(timeoutSecond int,keyChanged *string)error{
	id := fmt.Sprintf("watch-%s-%03d",time.Now(),rand.Int())
	ch:=make(chan string,10) //buffered

	kv.mu.Lock()
	kv.filter[id] = func(key string) {
		ch<-key
	}
	kv.mu.Unlock()

	select {
		case <-time.After(time.Duration(timeoutSecond)*time.Second):
			return fmt.Errorf("timeout")
		case key:=<-ch:
			*keyChanged = key
			return nil
	}
	return nil
}
```

watch方法的输入参数是超时的秒数，当key有变化时将Key作为返回值返回，如果超时时间后依然没有key被修改，则返回超时的错误。Watch的实现中，用唯一的id表示每个watch调用，然后根据id将自身对应的过滤器注册到kv.filter列表

客户端使用Watch方法：

```go
func doClientWork(client *rpc.Client){
	go func() {
		var keyChanged string
		err := client.Call("KVStoreService.Watch", 30, &keyChanged)
		if err != nil{
			fmt.Println("KVStoreService.Watch err:",err)
		}
		fmt.Println("watch:",keyChanged)
	}()

	err := client.Call("KVStoreService.Set", [2]string{"abc", "abc-alive"}, new(struct{}))
	if err!= nil{
		fmt.Println("err:",err)
	}
	time.Sleep(time.Second*3)
}
```

首先启动一个独立的Goroutine监控key的变化。同步的watch调用会阻塞，直到有key发生变化或者超时。然后在通过Set方法修改KV值时，服务器会将变化的key通过Watch方法返回。这样我们就可以实现对某些状态的监控。

#  上下文信息

基于上下文我们可以针对不同客户端提供定制化的RPC服务。我们可以通过为每个链接提供独立的RPC服务来实现对上下文特性的支持。

首先改造HelloService，里面增加了对应链接的conn成员：

```go
type HelloService struct {
    conn net.Conn
}
```

然后为每个链接启动独立的RPC服务：

```go
func main() {
    listener, err := net.Listen("tcp", ":1234")
    if err != nil {
        log.Fatal("ListenTCP error:", err)
    }

    for {
        conn, err := listener.Accept()
        if err != nil {
            log.Fatal("Accept error:", err)
        }

        go func() {
            defer conn.Close()

            p := rpc.NewServer()
            p.Register(&HelloService{conn: conn})
            p.ServeConn(conn)
        } ()
    }
}
```

Hello方法中就可以根据conn成员识别不同链接的RPC调用：

```go
func (p *HelloService) Hello(request string, reply *string) error {
    *reply = "hello:" + request + ", from" + p.conn.RemoteAddr().String()
    return nil
}
```

基于上下文信息，我们可以方便地为RPC服务增加简单的登陆状态的验证：

```go
type HelloService struct {
    conn    net.Conn
    isLogin bool
}

func (p *HelloService) Login(request string, reply *string) error {
    if request != "user:password" {
        return fmt.Errorf("auth failed")
    }
    log.Println("login ok")
    p.isLogin = true
    return nil
}

func (p *HelloService) Hello(request string, reply *string) error {
    if !p.isLogin {
        return fmt.Errorf("please login")
    }
    *reply = "hello:" + request + ", from" + p.conn.RemoteAddr().String()
    return nil
}
```

这样可以要求在客户端链接RPC服务时，首先要执行登陆操作，登陆成功后才能正常执行其他的服务。