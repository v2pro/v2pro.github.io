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

countlog的使用是非常简单的，仅仅需要使用几个静态函数。无需提前初始化 logger 对象。基本上用起来和 `fmt.Println` 差不多。
