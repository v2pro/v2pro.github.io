---
layout: default
title: plz pnp
---

pnp：进程的即插即用

* TOC
{:toc}

# 接口设计原则

进程应该能够实现即插即用。只要启动之后，就可以自动加入所提供服务的集群。这里包括服务自发现，以及服务注册两个动作。

# 服务自发现 API

服务自发现如何做，取决于具体的实现。标准的 API 仅仅是要求自发现完成之后，把信息填入到 plz 全局变量上

```go
// who am i, set externally
var PrimaryIP string
var PrimaryPort int
var Service string
var Cluster string

// to be set externally, additional info about this process
var ProcessInfo = map[string]interface{}{}
```

# 服务注册 API

```go
var PingUrl = ""

// PlugAndPlay will register the process into the grid
func PlugAndPlay()
```

如果设置了 PingUrl，则调用了 PlugAndPlay 之后就会把当前进程和 PingUrl 建立一个定期的长连心跳。心跳采用 http 协议，用保持tcp连接不断的形式，实现5分钟级别的长连接。断开之后自动重连。

# 反向隧道 SPI

通过 PlugAnPlay 注册之后，管理端可以通过反向的隧道的接口来调用一些  SPI。主要用于问题定位。这个 SPI 仍未定义。
