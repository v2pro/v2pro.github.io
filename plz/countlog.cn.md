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
countlog.Trace(
   // 第一个参数是发生的 event
   "result of a+b", 
   // 后面的参数是 key, value 的格式
   "a", a,
   "b", b,
   "a+b", c,
)
```
