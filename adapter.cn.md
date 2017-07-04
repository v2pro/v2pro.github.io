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

开发者首先定义一个rpc方法的handler

```golang
type MyRequest struct {
	Field string
}

type MyResponse struct {
	Field string
}

func handleMyRequest(ctx context.Context, req MyRequest) (MyResponse, error) {
	return MyResponse{req.Field}, nil
}
```

然后把这个handler挂到一个server上，并启动

```golang
import "github.com/v2pro/plz"

signal, err := plz.BuildServer("http_address", "127.0.0.1:9000").
	Method("example", handleMyRequest).
	Start()
if err != nil {
	panic(err.Error())
}
signal.Wait()
```

具体这个server是用什么协议启动的，请求和响应是怎么编解码的，都由 SPI 来提供。

```golang
func init() {
	srv.Executors = append(srv.Executors, StartServer)
}

func StartServer(server *srv.Server) (srv.Notifier, error) {
//...
}
```

SPI 需要对传递进来的 Server 结构体进行翻译。这个结构体的定义是

```golang
type Server struct {
	Properties map[string]interface{}
	Methods    []map[string]interface{}
	// sub server will map to Service concept in thrift
	// will map to url in http
	SubServers []*Server
}
```

# Client

HTTP/THRIFT等RPC服务，统一适配成 client 的抽象。

开发者先获取一个client，然后再用这个client进行rpc：

```golang
import "github.com/v2pro/plz"

client := plz.ClientOf("douban", "index") 

type MyRequest struct{} // 省略
type MyResponse struct {} // 省略
req := MyRequest{}
resp := MyResponse{}
err := client.Call(context.TODO(), req, &resp)
```

如果有错误，则返回error。如果正常，则结果会用 `plz.Copy` 绑定到传入的MyResponse上。

具体这个 client 是什么，由 SPI 来提供。比如 http 调用，我们可以有一个简单的封装：

```golang
type httpClientAdapter struct {
	executor HttpExecutor
}

func (adapter *httpClientAdapter) Call(ctx context.Context, req interface{}, resp interface{}) error {
	httpReq := &http.Request{}
	httpReq = httpReq.WithContext(ctx)
	err := plz.Copy(httpReq, req)
	if err != nil {
		return err
	}
	httpResp, err := adapter.executor.Do(httpReq)
	if err != nil {
		return err
	}
	err = plz.Copy(resp, httpResp)
	if err != nil {
		return err
	}
	return nil
}
```

而 serviceName 和 methodName 是如何映射到不同的 ip:port 的 url 上的，则又交给了 executor 来实现。内部可能再有服务发现之类的 SPI 扩展点。

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

LoggerOf 的参数是 logger 自身的属性。对于 SPI 来说，根据这些属性可以区别对待不同的 logger，比如打印到不同的文件。也可以是同时发送一些指标给监控系统。对于业务逻辑代码来说，都是一样的接口。

```golang
var Providers = []func(loggerKv []interface{}) Logger{}
```

注册到这个 Providers 列表里，提供具体的 logger 实现。

开发者使用的时候就是先获得一个logger，然后用这个logger打日志。

```golang
var requestLogger = plz.LoggerOf("logger", "request")

func xxx() {
	if requestLogger.ShouldLog(logging.LEVEL_DEBUG) {
		requestLogger.Debug("handle request", "url", ctx.Request().URL.String())
	}
}
```
