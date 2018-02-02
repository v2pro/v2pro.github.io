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
countlog.Trace(
  "%(a)s + %(b)s = %(c)s", 
  "a", a, 
  "b", b, 
  "c", c)
``

输出的日志就是 `1 + 1 = 2` 了。这里和 `fmt.Println` 还是稍有不同，fmt 用的是顺序来标识参数，而 countlog 使用的是名字来标识参数。

