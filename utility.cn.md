---
layout: default
title: 实用函数
---

实用函数

* TOC
{:toc}

以下所有的实现，都构建在对基础语言的两个扩展的基础上：

* [扩展的tagging](/lang.cn.html#tagging)
* [Accessor](/lang.cn.html#accessor)

# 编解码

统一的一个api支持各种类型的编解码和对象拷贝

```
func Copy(dst, src interface{}) error
```

## HTTP Query 编解码

## JSON 编解码

## THRIFT 编解码

# 参数验证

统一的一个api支持所有类型的参数验证需求

```golang
func Validate(obj interface{}) error
```

# 函数式编程

让 Go 在函数式编程方面，不输于 Python。

* map
* fold
* filter
* sort/sorted

# LINQ

用类 SQL 的语法来操作对象图，简化 Go 的泛型编程。