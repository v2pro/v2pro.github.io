---
layout: default
title: 扩展 Go 语言自身
---

扩展 Go 语言自身

* TOC
{:toc}

# App

```golang
import "github.com/v2pro/plz"

plz.RunApp(func() int {
	// os.Exit(0)
	return 0
})
```

把main函数包装起来。返回值就是进程的exit code。提供以下两个 SPI

## BeforeFinish

```golang
import "github.com/v2pro/plz/lang/app"

app.BeforeFinish = append(app.BeforeFinish, func(kv []interface{}) {
	ioutil.WriteFile("/tmp/hello", []byte("world"), os.ModeAppend|0666)
})
```

在退出之前回调。一般用于处理一些日志刷盘的事情。

## AfterPanic

```golang
import "github.com/v2pro/plz/lang/app"

app.AfterPanic = append(app.AfterPanic, func(recovered interface{}, kv []interface{}) int {
	fmt.Println("panic", recovered)
	return 2
})
```

在main函数panic的时候被回调。可以在这里做一些打日志，上报指标的事情。

# Routine

```golang
import "github.com/v2pro/plz"

plz.Go(func() {
	fmt.Println("hello from one off goroutine")
	panic("should not crash the whole program")
})
```

通过 `plz.Go` 调用比直接 `go` 可以自动实现 panic 的捕捉。

```golang
import "github.com/v2pro/plz"

plz.GoLongRunning(func() {
	timer := time.NewTimer(time.Second).C
	for {
		select {
		case <-timer:
			fmt.Println("hello from running goroutine")
			return
		}
	}
})
```

对于常驻的后台goroutine，我们往往需要保证它不要挂掉。简单的panic recover会导致goroutine退出。使用 `plz.GoLongRunning` 启动的goroutine可以在异常之后自动重启。

包装goroutine的生命周期，除了给一些默认的行为之外，更多是为了把一些 SPI 暴露出去，从而使得其他的库能够进行一些扩展（比如限制同时并发数，实现goroutine local storage）。

## AfterPanic

```golang
routine.AfterPanic = append(routine.AfterPanic, func(recovered interface{}, kv []interface{}) {
	fmt.Println("panic", recovered)
})
```

在任何goroutine panic之后被调用。如果是`GoLongRunning`，则可能一个goroutine多次调用AfterPanic。调用者在goroutine内部。

## BeforeFinish

```golang
routine.BeforeFinish = append(routine.BeforeFinish, func(kv []interface{}) {
	fmt.Println("finished")
})
```

在goroutine退出之前被调用。每个goroutine仅调用一次。调用者在goroutine内部。

## BeforeStart

```golang
routine.BeforeStart = append(routine.BeforeStart, func(kv []interface{}) error {
	return errors.New("can not start more goroutine")
})
```

在goroutine被启动之前调用。如果返回错误，则goroutine不会被启动。错误会作为 `plz.Go` 或者 `plz.GoLongRunning` 的返回值返回。调用者在goroutine外部。

## AfterStart

```golang
routine.AfterStart = append(routine.AfterStart, func(kv []interface{}) {
	fmt.Println("started")
})
```

在goroutine启动之后被调用。与BeforeStart不同，调用者在goroutine内部。

# Tagging

# Accessor 
