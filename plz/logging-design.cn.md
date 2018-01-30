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

# 低成本的 tracing

tracing 在没有打开的情况下，应该是低成本的。debug 也是一样。但是如果所有的地方都要写 if/else 的判断就太麻烦了。

## 低开销的级别判断：build tags

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

## 低开销的级别判断：if else

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

## 参数传递怎么办？

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

## interface{} 是不是一个好主意呢？

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

## 耗时的参数准备

```go

var MinLevel = 20

func Trace(kv ...interface{}) {
	if 10 < MinLevel {
		return
	}
	ptr := unsafe.Pointer(&kv)
	ptrAsValue := uintptr(ptr)
	args := castEmptyInterfaces(ptrAsValue)
	for i := 0; i < len(args); i += 2 {
		key := args[i].(string)
		value := args[i+1].(func() interface{})()
		blackHole(key)
		blackHole(value)
	}
}

func blackHole(interface{}) {
}

func castEmptyInterfaces(ptr uintptr) []interface{} {
	return *(*[]interface{})(unsafe.Pointer(ptr))
}

var v1 = 1024 * 1024
var v2 = 4096 * 4096

func Benchmark_expandable_args(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		Trace("k1", func() interface{} {
			return make([]byte, v1)
		}, "v2", func() interface{} {
			return make([]byte, v2)
		})
	}
}
```

通过把更耗时的参数准备放到 `func() interface{}` 的回掉里，我们可以只在 level 满足要求的时候，才对这些参数进行求值。这个循环跑下来，单次只需要8ns的时间。这个验证了构造closure的实例并不会消耗特别多的资源。

测试代码 [https://github.com/v2pro/logging-design/tree/master/expandable-args](https://github.com/v2pro/logging-design/tree/master/expandable-args)

## 去掉变长参数

通过前面的实验，我们可以看到 Go 的编译器还没有办法把变长参数的构造给完全优化掉。虽然传递过去的参数实际上并不会被使用，inline 并不会把 if 的判断往前提。zerolog 和 zap 的做法是不用变长参数进行传参。

```go
var MinLevel = 20

func Trace() *Event {
	if 10 < MinLevel {
		return nil
	}
	return &Event{}
}
```

如果 log 级别不满足，返回的是空的 event 对象

```go
type Event struct {
	keys []string
	values []interface{}
}

func (event *Event) Float64(key string, value float64) *Event {
	if event == nil {
		return event
	}
	if event.keys != nil {
		event.keys = append(event.keys, key)
	}
	event.values = append(event.values, value)
	return event
}

func (event *Event) Int(key string, value int) *Event {
	if event == nil {
		return event
	}
	if event.keys != nil {
		event.keys = append(event.keys, key)
	}
	event.values = append(event.values, value)
	return event
}
```

如果 event 为空，则不做任何操作返回

```go
var k1 = "k1"
var v1 = 100
var k2 = "k2"
var v2 = 10.24

func Benchmark_zerolog(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		e := Trace()
		e = e.Int(k1, v1)
		e = e.Float64(k2, v2)
		e.Log()
	}
}
```

benchmark 跑下来，一次需要 1.5 ns。相比之前的 `...interface{}`，速度要快很多。可以看见创建 `[]interface{}` 的开销被省掉了。当然这里用的是有类型的接口和 `interface{}` 比，不是很公平。

```go
func (event *Event) Object(key string, value interface{}) *Event {
	if event == nil {
		return event
	}
	if event.keys != nil {
		event.keys = append(event.keys, key)
	}
	ptr := unsafe.Pointer(&value)
	event.values = append(event.values, castEmptyInterface(uintptr(ptr)))
	return event
}

func castEmptyInterface(ptr uintptr) interface{} {
	return *(*interface{})(unsafe.Pointer(ptr))
}
```

对比一下 `interface{}` 的情况

```go
func Benchmark_zerolog(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		e := Trace()
		e = e.Object(k1, v1)
		e = e.Object(k2, v2)
		e.Log()
	}
}
```

这个单次的速度是 3ns，仍然要快很多。

## unsafe.Pointer 是危险的

通过 `unsafe.Pointer` 我们接管了逃逸分析。如果使用不当，就会导致引用已经释放了的栈上的值。下面演示一个滥用之后导致出错的例子：

```go
type SomeValue struct{
	string
	float64
}

func A() {
	ctx := map[string]interface{}{}
	B(ctx)
	C(ctx)
	if ctx["hello"].(SomeValue).float64 != 10.24 {
		panic("invalid")
	}
}

//go:noinline
func B(ctx map[string]interface{}) {
	var obj interface{} = SomeValue{"hello", 10.24}
	ptr := unsafe.Pointer(&obj)
	ctx["hello"] = castEmptyInterface(uintptr(ptr))
}

//go:noinline
func C(ctx map[string]interface{}) interface{} {
	a := ctx["hello"]
	b := ctx["hello"]
	if a == nil {
		return a
	}
	return b
}


func castEmptyInterface(ptr uintptr) interface{} {
	return *(*interface{})(unsafe.Pointer(ptr))
}

func Benchmark_A(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		A()
	}
}
```

运行后会发现 panic 被触发了。因为 ctx 里保存的 `interface{}` 引用了B的栈上分配的对象，而 B 退出之后这个对象已经失效了。

## 传参方式的结论

虽然 zerolog 的传参方式看起来很有吸引力。但是我们仍然选择 `...interface{}`。因为从使用的角度来说

```
Trace().Int64(k1, v1).Float64(k2, v2).Msg("xxxx")
```

看起来实在是太奇怪了。如果忘记了调用 Msg 则不会有日志输出。

```
Trace("xxxx", k1, v1, k2, v2)
```

这种写法明显更容易让人接受。在性能非常敏感的场景下， 还是用 `if ShouldLog(DEBUG) {}` 这样的写法吧。毕竟 zerolog 的写法虽然开销小，也不是没有开销的。而且在命中了 log level 之后，zerolog 还是要分配空间来保存参数的，虽然用 sync.Pool 做了全局的对象池共享。引入 sync.Pool 又增加了多线程之间的同步开销。

对于 `func() interface{}`，我们选择不使用这种写法。如果有昂贵的计算需要，应该用 `if ShouldLog(DEBUG) {}` 这个写法。

# 错误处理的写法

Go 的错误处理很难写得很漂亮。

```
input := []byte(`1,2,3]`)
var value interface{}
err := json.Unmarshal(input, &val)
if err != nil {
   return err
}
```

用户得到错误消息是 `invalid character ',' after top-level value`。如果没有加任何日志话，这个错误就很难找原因了。

