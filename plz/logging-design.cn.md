---
layout: default
title: plz logging
---

日志 API 设计之路

* TOC
{:toc}

# 愿景

1、使用简单。我们希望让日志埋点的写法变得非常非常简单，从而可以不影响代码可读性的前提下，大量地添加日志。比如说，基本上所有的对外操作之后，都要把 error 给记录下来。

2、一石三鸟。基本的日志仅仅是错误日志。高级的一点的日志还希望看一下统计数据，比如错误率，比如延迟的histogram。再高级一点，希望能开tracing，把所有的调用都给log下来。我们不希望给错误日志，指标统计和tracing，打印三份日志。要写一行打印日志的代码，把三个目的都满足了。另外业务上使用的一些事件输出，其实也可以走日志的接口。这意味着一定是结构化日志的 API。

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

# 参数传递怎么办？

传统的日志打印的最佳实践是

```go
if ShouldLog(DEBUG) {
  Debug(xxx)
}
```

这里主要避免的其实不是 `Debug` 这个函数的调用开销。而是准备参数的成本。对于结构化日志来说，参数不是格式化成 string 再传递进去的，而是传递原始的参数。这样格式化的大头开销就避免了。但是参数传递是免费的吗？

```go
var MinLevel = 20

func Trace(kv ...string) {
	if 10 < MinLevel {
		return
	}
	fmt.Println(kv)
}

func Benchmark_arguments_passing(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		Trace("hello", "world", "hey", "there")
	}
}
```

我们可以看到，这个跑下来单次要 50ns，而且有一次内存分配。这就是因为变长参数的传递引起的内存分配。通过设置 `-gcflags="-m"`，我们可以观察到下面的信息：

```
./log_test.go:18:16: inlining call to testing.(*B).ReportAllocs
./log_test.go:14:13: kv escapes to heap
./log_test.go:10:18: leaking param: kv
./log_test.go:14:13: Trace ... argument does not escape
./log_test.go:20:8: ... argument escapes to heap
./log_test.go:17:37: Benchmark_arguments_passing b does not escape
```

之所以内存分配没有延迟进行，是因为 arguments 逃逸到了堆上，因为 `fmt.Println` 这个调用。解决办法是拷贝一份参数。

```go
func Trace(kv ...string) {
	if 10 < MinLevel {
		return
	}
	fmt.Println(append(([]string)(nil), kv...))
}
```

改成这个写法之后，单次调用的时间下降到了5ns，而且没有内存分配。通过 `-gcflags="-m"` 也可以验证确实是如此

```
./log_test.go:18:16: inlining call to testing.(*B).ReportAllocs
./log_test.go:10:18: leaking param content: kv
./log_test.go:14:20: append(([]string)(nil), kv...) escapes to heap
./log_test.go:14:13: Trace ... argument does not escape
./log_test.go:17:37: Benchmark_arguments_passing b does not escape
./log_test.go:20:8: Benchmark_arguments_passing ... argument does not escape
```

对应的测试代码 [https://github.com/v2pro/logging-design/tree/master/variable-args](https://github.com/v2pro/logging-design/tree/master/variable-args)

# interface{} 是不是一个好主意呢？

既然变长的 string 没问题，那么变长的 `interface{}` 是不是也可以呢？

```go
var MinLevel = 20

func Trace(kv ...interface{}) {
	if 10 < MinLevel {
		return
	}
	fmt.Println(append(([]interface{})(nil), kv...))
}

var k1 = "k1"
var v1 = 100
var k2 = "k2"
var v2 = 10.24

func Benchmark_empty_interface(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		Trace(k1, v1, k2, v2)
	}
}
```

跑一下发现，单次需要104ns，而且有4次内存分配。逃逸分析显示

```
./log_test.go:23:16: inlining call to testing.(*B).ReportAllocs
./log_test.go:10:18: leaking param content: kv
./log_test.go:14:20: append(([]interface {})(nil), kv...) escapes to heap
./log_test.go:14:13: Trace ... argument does not escape
./log_test.go:25:8: k1 escapes to heap
./log_test.go:25:8: v1 escapes to heap
./log_test.go:25:8: k2 escapes to heap
./log_test.go:25:8: v2 escapes to heap
./log_test.go:22:35: Benchmark_empty_interface b does not escape
./log_test.go:25:8: Benchmark_empty_interface ... argument does not escape
```

虽然 arguments 没有逃逸。但是编译器认为 k1, v1, k2, v2 这四个 `interface{}` 都逃逸了，所以需要在堆上进行分配。


```go
var MinLevel = 20

func Trace(kv ...interface{}) {
	if 10 < MinLevel {
		return
	}
	ptr := unsafe.Pointer(&kv)
	ptrAsValue := uintptr(ptr)
	args := castEmptyInterfaces(ptrAsValue)
	fmt.Println(args)
}

func castEmptyInterfaces(ptr uintptr) []interface{} {
	return *(*[]interface{})(unsafe.Pointer(ptr))
}
```

通过把 kv 的引用，强转成 `uintptr` 之后。我们成功地避免了堆上分配。调用时间降回了10ns。

测试代码 [https://github.com/v2pro/logging-design/tree/master/empty-interface](https://github.com/v2pro/logging-design/tree/master/empty-interface)






