扩展 Go 语言自身

# App

```golang
plz.RunApp(func() int {
	// os.Exit(0)
	return 0
})
```

把main函数包装起来。返回值就是进程的exit code。提供以下两个 SPI

## BeforeFinish

```golang
app.BeforeFinish = append(app.BeforeFinish, func(kv []interface{}) {
	ioutil.WriteFile("/tmp/hello", []byte("world"), os.ModeAppend|0666)
})
```

在退出之前回调。一般用于处理一些日志刷盘的事情。

## AfterPanic

```golang
app.AfterPanic = append(app.AfterPanic, func(recovered interface{}, kv []interface{}) int {
	fmt.Println("panic", recovered)
	return 2
})
```

在main函数panic的时候被回调。可以在这里做一些打日志，上报指标的事情。

# Routine

# Tagging

# Accessor 
