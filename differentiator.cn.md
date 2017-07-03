---
layout: default
title: 核心竞争优势
---

核心竞争优势

* TOC
{:toc}

# 总体思路

重新造重复的轮子，就是浪费生命。

* 基础组件：大部分轮子都要把用户绑死在一个实现上。我们需要一个中间层，通过 interface 提供 API 给直接用户，提供 SPI 做为扩展点给其他的库作者。让业务和具体的轮子解耦，使得业务逻辑可以方便迁移。通过 SPI 让轮子之间的互相组合成为可能。
* 业务平台：在线业务的微服务可以分成“配置”，“存储”，“流程”和“智能”四类。提供平台使得四类微服务共性的部分可以下沉，使得写业务的同学可以专注于业务逻辑本身。
* 进程外流量监控和回放：对于遗留应用，改代码的成本是非常高的。通过在网络层分析流量，可以快速给遗留应用提供质量保证手段。

# 基础组件

基础组件就是一些Go的Library。调用的基础服务往往是开源的，比如 Mysql，Redis，Kafka这些。

## 扩展 Go 语言自身

### main()/go 的封装

所有 fork 出去的 goroutine 都需要进行包装，从而统一进行 panic 的 recover。对于常驻后台的 goroutine，提供 supervisor 的功能在 panic 之后重启。

主 goroutine 也需要进行包装。提供 SPI 在退出之前做一些额外操作。

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

## 实用函数


### Copy

所有的协议编解码都可以抽象为以下几类对象的通过Accessor实现互相拷贝。当然代码里如果有对象互相拷贝的需求，深拷贝一个对象的需求，也可以使用 Copy：

* map 顺序读 => struct 随机写（比如json绑定到struct）
* struct 顺序读 => struct 随机写 （mysql协议等有schema的协议解析过程，原始的[]byte读的时候，无法做基于offset的随机定位）
* struct 顺序读 => map 顺序写 （struct的json序列化）
* 其他同类型对象的拷贝 （简单赋值）

在拷贝的过程中，如果有类型不符等场景，可以提供 SPI 去做 Accessor 的包装。比如把一个 int 类型的 accessor 转换为 string 类型的 accessor。这样，无论是什么协议的编解码，都可以用统一的方式来做外部扩展。

### Validate

所有的参数验证都可以抽象为对象图的遍历。通过 Accessor 接口，我们可以很容易遍历任意输入。

### 函数式编程

基于Accessor接口，我们可以提供一些常见的函数式编程的便利性。

* map
* fold
* filter
* sort / sorted
* to_map
* ...
* linq：语言内的sql

## 标准化 RPC

### Logger

logger 接口提供抽象地进程对外输出“非业务” event 的能力。日志和指标是常见的两种类型的event。“非业务”主要体现在非功能性需求上：

* 写event不参与事务。不能因为写event失败，使得业务操作回滚。
* 量比较大，只保证最大努力地可靠性，在高负载下可丢弃。
* 只写不读。业务逻辑本身不依赖于这些 event。

### Server 

所有的RPC服务，都可以抽象为 `map[string]Handler`。key是方法名，value是方法的handler。Hnandler就是一个方法

```
func Handle(ctx context.Context, request MyRequestType) (response MyResponseType, err error)
```

提供 SPI 来处理：
* request/response的对象拷贝问题
* IDL，验证
* 服务注册
* 限流
* 超时控制
* 指标监控

### Client

所有的对外RPC调用可以分为两类：

* 封闭型的服务：比如特定业务的HTTP/THRIFT接口
* 开放型服务：Redis/Mysql等。常见的做法是封装DAO，把开放型的接口封闭起来。所以最后也是变成封闭型的接口。

所有的PRC调用，于是都可以抽象为

```
func Call(ctx context.Context, serviceName string, methodName string, request MyRequestType) (response MyResponseType, err error)
```

提供SPI来处理：
* request/response的对象拷贝问题
* IDL，验证
* 服务发现/负载均衡/故障节点摘除
* 熔断/降级
* 指标监控

MQ 本身也是一个同步的rpc服务。只是rpc调用的是一个通用的队列服务。从调用者角度来说，MQ其实是同步的rpc，而不是异步的。写MQ和用mysql存一个event到表里面，其实并没有本质区别。所以MQ也可以适配到抽象的Client模型。

# 业务平台

自身是提供RPC接口的网络服务。框架模式，业务逻辑以插件的方式托管到平台上。

## 配置平台

提供具体业务无关的配置能力。配置格式定义之后，自动生成配置界面。配置数据配送到机器，提供进程内缓存和更新的能力。常见的配置访问方式提供sdk

* 开关类配置的 sdk
* i18n配置的 sdk
* 规则类配置的 sdk

## 存储平台

提供具体业务无关的存储能力。存储分为两部分

* 主存储：带版本的文档型存储。实质上是一个event store
* 视图：所谓从库
  * 详情类视图： mysql/redis/elasticsearch，各类存储。按不同的主键重新hash
  * 统计类视图：业务指标统计

核心功能

* 幂等性：业务无需自己实现幂等性。通过把业务逻辑托管给存储平台，实现所有command处理的自动幂等。
* 并发下的数据完整性保证：通过乐观锁和业务逻辑托管，保证单个entity（aggregate root）的状态永远不会违背业务规则。
* 视图的同步：托管视图的同步逻辑。实现推拉结合的视图同步方式，保证更新永远不丢（通过拉），保证更新尽可能的及时（通过推）。

## 流程平台

提供具体业务无关的流程控制能力。有点类似 bpm。主要提供两个能力：

* 流程类逻辑的简化编程
* 最终一致性的框架性保证。

支持以下几类主要流程类型：

* 在线同步顺序更新多个主存储
* 在线+异步的业务逻辑
* saga pattern：带回滚的业务流程
* 审批类需求：把人融入到流程中

流程支持嵌套叠加。允许不同的团队负责流程的某一段。不要求全部集中在一起（无论是逻辑上，还是物理上的）。

## 智能平台

业务系统的core domain往往体现为通过机器代替人类进行智能决策。这种智能决策有两种实现方式：

* 业务规则：比如计费，比如税的计算
* 机器学习的模型

这类微服务比较难以抽象出共性的部分进行下沉。只能是把输入输出做一些简化，还有就是类map/reduce的计算模型的抽象。

# 进程外流量监控和回放

独立的工具，有界面有命令行。

## 进程外流量监控

通过内核抓包，知道哪个线程在收发哪些数据。通过数据的协议解码，知道这些数据是在做什么样的RPC调用。通过统计，实现指标监控。通过上下游tcp连接信息的串联，实现分布式trace。

## 流量回放

流量监控抓下来的包，可以用于在线下实现单接口的流量回放，从而部分替代单元测试，解决遗留应用不敢改的问题。结合IDL，实现流量编辑，用于测试新功能。
