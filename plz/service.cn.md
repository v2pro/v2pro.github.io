---
layout: default
title: plz service
---

Service：服务接口

* TOC
{:toc}

# 接口设计的原则

plz 定义的是 API/SPI，而不是具体的实现。非功能性需求有自己的演进步伐，比如从 http 升级到 thrift 传输和编解码。传统的服务框架和web框架的做法没法有效地定义功能性需求和非功能性需求地边界，比如一般地 http 服务的 controller 是这么定义的

```go
func myController(respWriter http.ResponseWriter, req *http.Request)
```

controller 里需要耦合具体的传输协议的接口，比如从 `*http.Request` 里获取参数。类似的当我们去调用 http 服务的时候，也是这样的情况。

plz service 的目标是定义一个边界，在边界以内，完全不出现任何和具体的 RPC 方式相关的代码。

关于服务依赖的问题，比如我们的代码依赖一个 passport 服务。在测试等场景下，我们有替换这个具体服务的实现的需求。一般来说，有三种做法

* 依赖注入：通过函数参数或者对象组装实现 IoC
* Service Locator：在调用的时候去获取具体的实现
* 全局依赖注入

我们选择的代码风格是全局依赖注入。也就是

```
var validatePassport = func(*countlog.Context, *passport.Ticket) (*passport.Result, error)

// use validatePassport directly as a function
```

在运行时，把不同的函数实现赋给 `validatePassport` 这个函数指针，从而切换实现。这种代码风格比依赖注入和Service Locator更简洁，Boilerplate code 更少。

# Service Provider Interface

plz service 不定义统一的 API（比如如何启动web server，如何解析服务名字）。但是提供统一的 SPI。因为不同的服务暴露方式千差万别，thrift/protobuf/http 等不同的传输协议需要自己的 API 来定义自己的行为。但是所有的服务最终都需要提供 SPI 插入功能性需求代码。比如 http 需要定义给定 URL 对应的 handler。这个统一的 SPI 接口定义如下。

```go
func(ctx *countlog.Context, request *mypkg.MyRequest) (response *mypkg.MyResponse, err error)
```

* ctx: 传递了请求的上下文。实现了 `context.Context` 接口，同时提供了埋点的 API。
* request：一定需要是一个struct的指针。具体的struct类型由使用方自己定义。含义是服务调用的输入。
* response：一定需要是一个struct的指针。具体的struct类型由使用方自己定义。含义是服务调用的正常返回。
* error：类型是error，可能实现了 ErrorNumber 接口，提供错误码。含义是服务调用的错误返回。

```go
type ErrorNumber interface {
   ErrorNumber() int
}
```

如果返回的 error 实现了 `ErrorNumber()` 则会尝试把错误码加入到具体协议的响应里。不同的传输和编解码协议对于错误码的编码方式是不同的。

# 服务器的推荐 API 风格

前面定义的 SPI 需要注册成为服务器的 handler。推荐的 API 风格是这样的：

```go
func sayHello(ctx countlog.Context, req *MyReqeust) (*MyResponse, error) {
	// ...
}
server := http.NewServer()
server.Handle("/sayHello", sayHello)
server.Start("127.0.0.1:9998")
```

# 客户端的推荐 API 风格

前面定义的 SPI 作为客户端使用的时候，需要获取一个有具体实现的 handler 来进行RPC调用。推荐的 API 风格是这样的：

```go
var sayHello = func (ctx countlog.Context, req *MyReqeust) (*MyResponse, error)
client := http.NewClient()
client.Handle("POST", "http://127.0.0.1:9998/sayHello", &sayHello)

// use sayHello(...) to call server
```

注意 `&sayHello` 传入了一个指针，从而给 `sayHello` 赋予了一个具体的实现。

# 具体实现

所有的服务提供和调用都可以用前面定义的 SPI 来定义非功能性需求的边界。比如

* 提供 http 服务
* 消费 http 服务
* 提供 thrift 服务
* 消费 thrift 服务
* 消费 sql 服务，比如访问 mysql

目前实现了两个example的服务器和客户端：[https://github.com/v2pro/plz.service](https://github.com/v2pro/plz.service)

不同的具体实现里可以自由发挥地去添加各种非功能性需求的可复用实现。比如熔断限流，服务发现等功能。这些功能都可以适配到前面定义的 SPI 上。
