---
layout: default
title: plz service
---

Service：服务接口

* TOC
{:toc}

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
