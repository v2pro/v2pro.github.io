---
layout: default
title: plz countlog
---

countlog 使用指南

* TOC
{:toc}

# 基本使用

```go
a := 1
b := 1
c := a + b
// countlog.Trace 是 countlog 包提供的静态函数，打印 TRACE 级别的日志
countlog.Trace(
   // 第一个参数是发生的 event
   "result of a+b", 
   // 后面的参数是 key, value 的格式
   "a", a,
   "b", b,
   "a+b", c,
)
```

countlog的使用是非常简单的，仅仅需要使用几个静态函数。无需提前初始化 logger 对象。基本上用起来和 `fmt.Println` 差不多。但是 countlog 和很多日志库又不同，它提供的是结构化日志打印的 API。也就是 countlog 预期的输入不是一个字符串，而是一个结构化的结构体。

```go
// 错误的用法！
countlog.Trace(fmt.Sprintf("result of a+b=%v", a+b))
```

countlog 不希望被这样使用。用户不需要格式化好字符串再来调用 countlog，而是让 countlog 来负责日志的格式化。默认的格式化方式会把所有提供的参数都给打印出来。类似这样的格式

```
=== result of a+b ===
a: 1
b: 1
a+b: 2
```

输出格式是可以自由定制的。我们可以把格式改成 Printf 的风格的

```go
import "github.com/v2pro/plz/countlog/output"
import "github.com/v2pro/plz/countlog/output/printf"
import "github.com/v2pro/plz/countlog"
import "os"

countlog.EventWriter = output.NewEventWriter(output.EventWriterConfig{
  Format: &printf.Format{},
  Writer: os.Stdout,
})
```

然后就可以使用

```go
a = 1
b = 1
c = a + b
// countlog.EventWriter 已经修改成用 &printf.Format{} 定义的格式输出
countlog.Trace(
  "%(a)s + %(b)s = %(c)s", 
  "a", a, 
  "b", b, 
  "c", c)
```

输出的日志就是 `1 + 1 = 2` 了。这里和 `fmt.Println` 还是稍有不同，fmt 用的是顺序来标识参数，而 countlog 使用的是名字来标识参数。顺便打一个广告，如果你想使用这个写法来调用 sprintf，可以 `import github.com/v2pro/plz/nfmt`。nfmt 这个包提供了 fmt 包一样的接口，但是用的是命名参数的传参方式。

# 错误处理

日志和错误处理是紧密关联的两个主题。而 Go 语言的错误处理从诞生以来就一直为人诟病。促使我们创造 countlog 这个新的轮子的原始动力之一，就是我们希望让 Go 的错误处理能够更加简洁。

```go
input := `[1,2,3`
var gameScores []int
err := json.Unmarshal(input, &gameScores)
if err != nil {
  return err
}
```

传统的 Go 的错误处理方式就是返回一个 error 对象，然后用 `if err != nil` 的方式来判断是否出错。这个本来已经是相对来说有点繁琐的写法了。但是在实践中我们发现仅仅判断 err != nil 是不完善的错误处理代码。完善的错误处理要同时保证三个方面的需求：

* 对于用户：我们需要在出错的时候给一个相对清晰的指引。让他们知道错误大概是什么，需要去找谁来处理。但是在前面这个例子里，`[1,2,3` 因为缺少了半边中括号，给出的默认错误消息是非常难以理解的。我们希望告诉用户的错误是 gameScores 读取失败。这意味着需要对原始的错误进行二次的包装。
* 对于同事：我们需要给出相对齐全的指标体系。比如接口的错误率，以及平均调用延迟。通过这些指标的监控告警，同事们才能在出问题的时候，找出哪个模块是根本原因。从而能让你来跟进后续的处理。
* 对于你自己：最终我们要为自己写的代码负责。如果出了问题，没有足够的信息来还原故障现场。那必然会浪费大量的时间在各种猜测上。同时在开发的过程中，频频需要定位一些低级错误。如果有完善的调用跟踪（tracing），可以很容易地知道问题是怎么发生的。

如果要把这三方面的需求都满足到位了。实际的错误处理代码远比 `if err != nil` 更复杂。大概会写成这个样子：

```go
err := json.Unmarshal(input, &gameScores)
if ShouldLog(LevelTrace) {
  fmt.Println("json.Unmarshal", string(input))
}
metrics.Send("unmarshal.stats", err)
if err != nil {
   fmt.Println("json.Unmarshal failed", err, string(input))
   return fmt.Errorf("failed to read game scores: %v", err.Error())
}
```

如果每个地方都要这么写。很快这些代码会把正常的业务逻辑给淹没掉。所以实际的情况是，大部分情况下我们都直接 `if err != nil { return err }` 就搞定了。然后在出了问题之后，再回代码里加上一些必要的日志和埋点。countlog 的目标是让你可以在第一遍写代码的时候就把必要的埋点都给低阅读成本，低执行成本地给添加上了。从而给开发，测试，故障定位节省大量的时间，从而提高整体的开发效率。

```go
input := `[1,2,3`
var gameScores []int
err := json.Unmarshal(input, &gameScores)
err = countlog.TraceCall("read game scores", err, "input", input)
if err != nil {
  return err
}
```

这里会同时做四件事情：

* 包装 error：countlog.TraceCall 返回的 error 是添加了 read game scores 这个前缀的。
* 错误日志：在出错的情况下，虽然这是一个 trace 级别的日志，仍然会按照 warn 的级别打印出日志。
* 指标统计：在没有出错的情况下，如果日志级别是 trace，会进行错误率的统计，然后发给对接的监控系统。
* 调用跟踪：如果日志级别设置得比 trace 还低半级（也就是 countlog.LevelTraceCall 这个级别），会把原始的调用详情也逐条打印出来。这个就相当于调试代码时添加的 fmt.Println 的作用了。

我们的建议是给所有会返回 error 的函数调用添加上 countlog.TraceCall （或者 DebugCall 和 InfoCall）。给所有的 RPC 远程函数调用加上 countlog.InfoCall。这样，我们对程序就有一个最基础的调用埋点了。那么问题是，所有地方都添加日志会不会很消耗性能？传统的日志打印的最佳实践是这样的：

```go
input := `[1,2,3`
var gameScores []int
err := json.Unmarshal(input, &gameScores)
if ShouldLog(LevelTrace) {
  log(LevelTrace, fmt.Sprintf("read game scores: %s", input))
}
if err != nil {
  return err
}
```

因为传统的日志库，虽然可以在内部通过级别判断是否打印日志。但是调用日志库本身的开销已经足够大了。所以要求用户在性能关键的路径调用 trace 或者 debug 的时候都需要添加 if 判断。但是这样搞，代码的可读性就更差了。我们来看一下实际上 countlog.TraceCall 在级别不满足的时候的开销。

```go
func Benchmark_trace(b *testing.B) {
	SetMinLevel(LevelDebug)
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		TraceCall("event!hello", nil, "a", "b")
	}
}

// 1000000000	         7.24 ns/op	       0 B/op	       0 allocs/op
```

实际跑出来的结果是，单次调用成本在 10ns 以内。在大部分的业务代码里，这点开销是完全可以忽略不计的。
