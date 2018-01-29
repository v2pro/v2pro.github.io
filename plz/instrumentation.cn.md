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

# expvar



