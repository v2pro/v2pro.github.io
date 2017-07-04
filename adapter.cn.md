---
layout: default
title: 标准化 RPC
---

标准化 RPC

* TOC
{:toc}

一个 RPC 框架基本上都需要包含以下四个方面的功能。现有的 RPC 框架的模式存在以下的问题：

* 因为和具体实现紧耦合。像日志库这样的东西很多地方都要用，但是不想引入那么重的依赖。提供 interface 的模式可以方便用户组合模块的时候，不会碰到用了四五种不同日志库的尴尬。
* 和具体的协议耦合，HTTP和THRIFT等不同接入方式无法复用代码。
* 各种 RPC 没有统一的切入点，不同的 RPC 调用要重复写代码接入服务发现，熔断，metrics上报等代码。

一个 rpc server，调用一堆的 rpc client，同时写很多的日志。这样就基本概括了大部分的日常工作了。而标准化 RPC 的作用就是让你的代码，和具体的与外界交互的实现用接口进行隔离。一方面让业务逻辑自身更清晰，同时可以让中间件能够做更多的事情与处理非功能需求。

# Server

HTTP/THRIFT/MQ 三种服务接入方式，适配成统一的接口。

# Client

HTTP/THRIFT等RPC服务，统一适配成 client 的抽象。

# Logging

提供标准化的 API/SPI 接口，适配各种现有的日志库。给用户的接口是

```golang
func LoggerOf(loggerKv ...interface{}) Logger

type Logger interface {
	Log(level Level, msg string, kv ...interface{})
	Error(msg string, kv ...interface{})
	Info(msg string, kv ...interface{})
	Debug(msg string, kv ...interface{})
	ShouldLog(level Level) bool
}
```

LoggerOf 的参数是 logger 自身的属性。对于 SPI 来说，根据这些属性可以区别对待不同的 logger，比如打印到不同的文件。

```golang
var Providers = []func(loggerKv []interface{}) Logger{}
```

注册到这个 Providers 列表里，提供具体的 logger 实现。
