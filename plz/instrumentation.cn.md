---
layout: default
title: plz instrumentation
---

Instrumentation：问题发现和定位的效率源头

* TOC
{:toc}

# 日志 API

所有队外暴露事件类型的埋点，统一走日志API。这里包括错误日志，业务事件，以及metrics指标统计。日志 API 定义为静态的函数，主要的考虑是为了效率，需要能够在编译器关掉部分的 trace/debug 日志输出。

```go
func ShouldLog(level int) bool
func Trace(event string, properties ...interface{})
func TraceCall(callee string, err error, properties ...interface{})
func Debug(event string, properties ...interface{})
func DebugCall(callee string, err error, properties ...interface{})
func Info(event string, properties ...interface{})
func InfoCall(callee string, err error, properties ...interface{})
func Warn(event string, properties ...interface{})
func Error(event string, properties ...interface{})
func Fatal(event string, properties ...interface{})
func Log(level int, event string, properties ...interface{})
```

因为经常需要把context作为日志参数。所以在 context 上直接定义了日志输出的快捷方式，默认把 context 加入到日志里。

```go
type Context struct {
	context.Context
}
func (ctx *Context) Trace(event string, properties ...interface{})
func (ctx *Context) TraceCall(callee string, err error, properties ...interface{})
func (ctx *Context) Debug(event string, properties ...interface{})
func (ctx *Context) DebugCall(callee string, err error, properties ...interface{})
func (ctx Context) Info(event string, properties ...interface{})
func (ctx *Context) InfoCall(callee string, err error, properties ...interface{})
func (ctx *Context) Warn(event string, properties ...interface{})
func (ctx *Context) Error(event string, properties ...interface{})
func (ctx *Context) Fatal(event string, properties ...interface{})
func (ctx *Context) Log(level int, event string, properties ...interface{})
```

`ctx.Info("event!hello")` 和 `countlog.Info("event!hello", "ctx", ctx)` 是等价的，就是为了写起来更简便一些。日志输出的第一个参数是event名字，必须是用`event!`开头。这么做的目的是为了方便把日志输出和代码里出现的位置关联起来。通过正则可以快速地在代码里找到所有输出的事件。

# expvar


