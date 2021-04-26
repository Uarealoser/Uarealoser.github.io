---
layout:     post
title:      golang net/rpc
subtitle:   golang net/rpc
date:       2021-04-26
author:     Uarealoser
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - rpc
---


# 基础实现

Go语言的RPC包的路径为net/rpc，也就是放在了net包目录下面

## provider
```go
type HelloService struct {}

func (p *HelloService) Hello(request string, reply *string) error {
    *reply = "hello:" + request
    return nil
}
```
其中GolangRPC规则：方法只能有2个可序列化的参数，其中第二个参数是指针类型，并且返回一个error类型。同时必须是公开方法。

```go
func main() {
    rpc.RegisterName("HelloService", new(HelloService))

    listener, err := net.Listen("tcp", ":1234")
    if err != nil {
        log.Fatal("ListenTCP error:", err)
    }

    conn, err := listener.Accept()
    if err != nil {
        log.Fatal("Accept error:", err)
    }

    rpc.ServeConn(conn)
}
```

其中rpc.Register函数调用会将对象类型中所有满足RPC规则的对象方法注册为RPC函数，所有注册的方法会放在“HelloService”服务空间之下。然后我们建立一个唯一的TCP链接，并且通过rpc.ServeConn函数在该TCP链接上为对方提供RPC服务。

## consumer

```go
func main() {
    client, err := rpc.Dial("tcp", "localhost:1234")
    if err != nil {
        log.Fatal("dialing:", err)
    }

    var reply string
    err = client.Call("HelloService.Hello", "hello", &reply)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Println(reply)
}
```
首先是通过rpc.Dial拨号RPC服务，然后通过client.Call调用具体的RPC方法。在调用client.Call时，第一个参数是用点号链接的RPC服务名字和方法名字，第二和第三个参数分别我们定义RPC方法的两个参数。

## 更安全的实现

在前面的例子中我们为了简化将以上几种角色的工作全部放到了一起，虽然看似实现简单，但是不利于后期的维护和工作的切割。

如果要重构HelloService服务，第一步需要明确服务的名字和接口：

```go
const HelloServiceName = "path/to/pkg.HelloService"

type HelloServiceInterface = interface {
    Hello(request string, reply *string) error
}

func RegisterHelloService(svc HelloServiceInterface) error {
    return rpc.RegisterName(HelloServiceName, svc)
}
```

我们将RPC服务的接口规范分为三个部分:

- 服务的名字
- 服务要要实现的详细方法列表
- 注册该类型服务的函数

```go
func main() {
    client, err := rpc.Dial("tcp", "localhost:1234")
    if err != nil {
        log.Fatal("dialing:", err)
    }

    var reply string
    err = client.Call(HelloServiceName+".Hello", "hello", &reply)
    if err != nil {
        log.Fatal(err)
    }
}
```
中唯一的变化是client.Call的第一个参数用HelloServiceName+".Hello"代替了"HelloService.Hello"。然而通过client.Call函数调用RPC方法依然比较繁琐，同时参数的类型依然无法得到编译器提供的安全保障。

为了简化客户端用户调用RPC函数，我们在可以在接口规范部分增加对客户端的简单包装：

```go
type HelloServiceClient struct {
    *rpc.Client
}

var _ HelloServiceInterface = (*HelloServiceClient)(nil)

func DialHelloService(network, address string) (*HelloServiceClient, error) {
    c, err := rpc.Dial(network, address)
    if err != nil {
        return nil, err
    }
    return &HelloServiceClient{Client: c}, nil
}

func (p *HelloServiceClient) Hello(request string, reply *string) error {
    return p.Client.Call(HelloServiceName+".Hello", request, reply)
}
```

我们在接口规范中针对客户端新增加了HelloServiceClient类型，该类型也必须满足HelloServiceInterface接口，这样客户端用户就可以直接通过接口对应的方法调用RPC函数。同时提供了一个DialHelloService方法，直接拨号HelloService服务。

基于新的客户端接口，我们可以简化客户端用户的代码：

```go
func main() {
    client, err := DialHelloService("tcp", "localhost:1234")
    if err != nil {
        log.Fatal("dialing:", err)
    }

    var reply string
    err = client.Hello("hello", &reply)
    if err != nil {
        log.Fatal(err)
    }
}
```
现在客户端用户不用再担心RPC方法名字或参数类型不匹配等低级错误的发生。

最后是基于RPC接口规范编写真实的服务端代码：

```go
type HelloService struct {}

func (p *HelloService) Hello(request string, reply *string) error {
    *reply = "hello:" + request
    return nil
}

func main() {
    RegisterHelloService(new(HelloService))

    listener, err := net.Listen("tcp", ":1234")
    if err != nil {
        log.Fatal("ListenTCP error:", err)
    }

    for {
        conn, err := listener.Accept()
        if err != nil {
            log.Fatal("Accept error:", err)
        }

        go rpc.ServeConn(conn)
    }
}
```

在新的RPC服务端实现中，我们用RegisterHelloService函数来注册函数，这样不仅可以避免命名服务名称的工作，同时也保证了传入的服务对象满足了RPC接口的定义。最后我们新的服务改为支持多个TCP链接，然后为每个TCP链接提供RPC服务。

