---
layout: default
title: 核心竞争优势
---

* TOC
{:toc}

# 总体思路

没有必要重新造重复的轮子，那是在浪费生命。

* 基础组件：大部分轮子都要把用户绑死在一个实现上。我们需要一个中间层，通过 interface 提供 API 给直接用户，提供 SPI 做为扩展点给其他的库作者。让业务和具体的轮子解耦，使得业务逻辑可以方便迁移。通过 SPI 让轮子之间的互相组合成为可能。
* 业务平台：在线业务的微服务可以分成“存储”，“流程”和“智能”三类。提供平台使得三类微服务共性的部分可以下沉，使得写业务的同学可以专注于业务逻辑本身。
* 进程外流量监控和回放：对于遗留应用，改代码的成本是非常高的。通过在网络层分析流量，可以快速给遗留应用提供质量保证手段。

# 基础组件

## Routine

### go()

所有 fork 出去的 goroutine 都需要进行包装，从而统一进行 panic 的 recover。对于常驻后台的 goroutine，提供 supervisor 的功能在 panic 之后重启。

### main()

主 goroutine 也需要进行包装。提供 SPI 在退出之前做一些额外操作。

## Logger

logger 接口提供抽象地进程对外输出“非业务” event 的能力。日志和指标是常见的两种类型的event。“非业务”主要体现在非功能性需求上：

* 写event不参与事务。不能因为写event失败，使得业务操作回滚。
* 量比较大，只保证最大努力地可靠性，在高负载下可丢弃。
* 只写不读。业务逻辑本身不依赖于这些 event。

### 日志

把 logger 接口适配到现有的日志库上

### 指标

把 logger 接口适配到现有的metrics库上

## 反射

### Accessor

把对象分为以下几类：

* 简单值
* Struct
* Map
* Array
* Variant

提供抽象的反射接口对这些对象进行读和写。这些对象底层可以是

* go 语言的对象
* http request/response
* json []byte
* thrift []byte
* mysql protocol []byte
* redis protocol []byte

Accessor提供的能力可分为（类似 STL 的 iterator）：

* 顺序读
* 随机读
* 顺序写
* 随机写

### Copy

所有的协议编解码都可以抽象为以下几类对象的通过Accessor实现互相拷贝。当然代码里如果有对象互相拷贝的需求，深拷贝一个对象的需求，也可以使用 Copy：

* map 顺序读 => struct 随机写（比如json绑定到struct）
* struct 顺序读 => struct 随机写 （mysql协议等有schema的协议解析过程，原始的[]byte读的时候，无法做基于offset的随机定位）
* struct 顺序读 => map 顺序写 （struct的json序列化）
* 其他同类型对象的拷贝 （简单赋值）

在拷贝的过程中，如果有类型不符等场景，可以提供 SPI 去做 Accessor 的包装。比如把一个 int 类型的 accessor 转换为 string 类型的 accessor。这样，无论是什么协议的编解码，都可以用统一的方式来做外部扩展。

### Validate

所有的参数验证都可以抽象为对象图的遍历。通过 Accessor 接口，我们可以很容易遍历任意输入。

### 函数式 Util

基于Accessor接口，我们可以提供一些常见的函数式编程的便利性。

* map
* fold
* filter
* sort / sorted
* to_map
* ...
* linq：语言内的sql

## Server 

所有的RPC服务，都可以抽象为 `map[string]Handler`。key是方法名，value是方法的handler。Hnandler就是一个方法

```
func Handle(ctx context.Context, request MyRequestType) (response MyResponseType, err error)
```

提供 SPI 来处理：
* request/response的对象拷贝问题
* 服务注册
* 限流
* 超时控制
* 指标监控

### HTTP

把url等输入映射到方法名上。适配HTTP框架到抽象的Server模型。

### THRIFT

适配THRIFT框架到抽象的Server模型。

### MQ

适配消息队列的topic和消息体到抽象的Server模型。

## Client

所有的对外RPC调用可以分为两类：

* 封闭型的服务：比如特定业务的HTTP/THRIFT接口
* 开放型服务：Redis/Mysql等。常见的做法是封装DAO，把开放型的接口封闭起来。所以最后也是变成封闭型的接口。

所有的PRC调用，于是都可以抽象为

```
func Call(ctx context.Context, serviceName string, methodName string, request MyRequestType) (response MyResponseType, err error)
```

提供SPI来处理：
* request/response的对象拷贝问题
* 服务发现/负载均衡/故障节点摘除
* 熔断/降级
* 指标监控

### HTTP

HTTP适配到抽象的Client模型

### THRIFT

THRIFT适配到抽象的Client模型

### SQL

SQL+DAO适配到抽象的Client模型

### Redis

Redis+DAO适配到抽象的Client模型

# 业务平台

# 进程外流量监控和回放
