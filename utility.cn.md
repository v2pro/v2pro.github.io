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

```golang
func Copy(dst, src interface{}) error
```

先把不同的协议用 Accessor 接口适配成对象图的遍历。然后所有的对象绑定问题都可以用 Copy 解决。具体的 Copy 实现，由 SPI 提供

```golang
type Copier interface {
	Copy(dst interface{}, src interface{}) error
}

var CopierProviders = []func(dstAccessor, srcAccessor lang.Accessor) (Copier, error){}
```

## HTTP Query 编解码

## JSON 编解码

## THRIFT 编解码

# 参数验证

统一的一个api支持所有类型的参数验证需求

```golang
func Validate(obj interface{}) error
```

具体的 Validate 实现，由 SPI 提供

```golang
type ResultCollector interface {
	Enter(pathElement interface{}, ptr uintptr)
	Leave()
	IsVisited(ptr uintptr) bool
	CollectError(err error)
}

type Validator interface {
	Validate(collector ResultCollector, obj interface{}) error
}

var ValidatorProviders = []func(accessor lang.Accessor) (Validator, error){}
```

# 函数式编程

让 Go 在函数式编程方面，不输于 Python。

* map
* fold
* filter
* sort/sorted

# LINQ

用类 SQL 的语法来操作对象图，简化 Go 的泛型编程。
