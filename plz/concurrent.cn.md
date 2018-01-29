---
layout: default
title: plz concurrent
---

concurrent：goroutine的生命周期

* TOC
{:toc}

# API

```go
type Executor interface {
  Go(handler func(ctx *countlog.Context))
  Stop()
  StopAndWaitForever()
  StopAndWait(ctx context.Context)
}
```

启动 goroutine 通过 Go 函数替代 go 关键字。停止 executor 可以是立即终止，也可以 graceful shutdown。

# 具体实现

目前只有一个 UnboundedExecutor 的实现，没有限制并发的 goroutine 的数量。
