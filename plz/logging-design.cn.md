---
layout: default
title: plz logging
---

日志 API 设计之路

* TOC
{:toc}

# 愿景

1、使用简单。我们希望让日志埋点的写法变得非常非常简单，从而可以不影响代码可读性的前提下，大量地添加日志。比如说，基本上所有的对外操作之后，都要把 error 给记录下来。

2、一石三鸟。基本的日志仅仅是错误日志。高级的一点的日志还希望看一下统计数据，比如错误率，比如延迟的histogram。再高级一点，希望能开tracing，把所有的调用都给log下来。我们不希望给错误日志，指标统计和tracing，打印三份日志。要写一行打印日志的代码，把三个目的都满足了。另外业务上使用的一些事件输出，其实也可以走日志的接口。

3、level不满足情况下要是低开销的。最常简的做法是 `if ShouldLog(DEBUG) {}`。但是这个写法非常的繁琐，而且容易忘记。我们希望在不这么写的前提下，也可以低成本的打印 DEBUG 和 TRACE 级别的日志。

4、指标统计的低开销。指标统计的调用次数会比普通的日志多很多。它相当于一种开销更低的tracing，毕竟把详情信息给省略掉了。之所以需要统计指标，也是在开销小的情况下，看到更低一个层次的信息，否则全量tracing就好了。

# 日志级别设计

在假设 WEB 应用的场景下。日志级别对应的量级如下：

TRACE：每个 WEB 请求会打印 N 条 TRACE 日志。而且 N 可能还和请求的数据量有关系。

DEBUG：每个 WEB 请求打印 1 条或者 2 条日志。基本上就是 Access Log 这个级别。

INFO：每个 WEB 请求打印少于 1 条。也就是只有在状态发生变化的情况下输出一条，不是每个 WEB 请求都会触发的日志。

ERROR：出错了，但是错误在预期范围内。对应 error 返回值的级别。

FATAL：出错了，而且没有预料到。对应 panic 以及类似的情况。

# 低开销的级别判断：build tags

低成本的实现 tracing 的一种方式就是把 trace 的函数实现变成空的。

```go
//+build release

package log

func Trace(event string) {
}
```

```go
//+build !release

package log

import "fmt"

func Trace(event string) {
	fmt.Println(event)
}
```

调用 log.Trace 的地方会 inline 函数的实现。如果函数是空的话，则完全没有开销了。通过反汇编我们可以验证这一点：[https://github.com/v2pro/logging-design/tree/master/build-tag](https://github.com/v2pro/logging-design/tree/master/build-tag)

# 低开销的级别判断：if else

使用 build tag 虽然可以实现最低成本的埋点。但是也丧失了灵活性。要想把日志重新打开，还需要重新编译二进制。因为 CPU 对判断有很好的预测能力，所以用 if/else 来判断级别其实并不会有太大的开销。


```go
const LevelTrace = 10
const LevelDebug = 20

var MinLevel = LevelDebug

func Trace(event string) {
	if LevelTrace < MinLevel {
		return
	}
	fmt.Println(event)
}
```

通过简单的测试

```go
func Benchmark_trace_branch_predication(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		Trace("hello")
	}
}
```

我们可以看到 if/else 其实开销是很低的。[https://github.com/v2pro/logging-design/tree/master/branch-predication](https://github.com/v2pro/logging-design/tree/master/branch-predication)





