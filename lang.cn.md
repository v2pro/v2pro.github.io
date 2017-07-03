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

## AfterPanic

```golang
import "github.com/v2pro/plz/lang/app"

app.AfterPanic = append(app.AfterPanic, func(recovered interface{}, kv []interface{}) int {
	fmt.Println("panic", recovered)
	return 2
})
```

在main函数panic的时候被回调。可以在这里做一些打日志，上报指标的事情。

## BeforeFinish

```golang
import "github.com/v2pro/plz/lang/app"

app.BeforeFinish = append(app.BeforeFinish, func(kv []interface{}) {
	ioutil.WriteFile("/tmp/hello", []byte("world"), os.ModeAppend|0666)
})
```

在退出之前回调。一般用于处理一些日志刷盘的事情。如果有panic，会先调用 `AfterPanic` 再调用 `BeforeFinish`。

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

## BeforeStart

```golang
import "github.com/v2pro/plz/lang/routine"

routine.BeforeStart = append(routine.BeforeStart, func(kv []interface{}) error {
	return errors.New("can not start more goroutine")
})
```

在goroutine被启动之前调用。如果返回错误，则goroutine不会被启动。错误会作为 `plz.Go` 或者 `plz.GoLongRunning` 的返回值返回。调用者在goroutine外部。

## AfterStart

```golang
import "github.com/v2pro/plz/lang/routine"

routine.AfterStart = append(routine.AfterStart, func(kv []interface{}) {
	fmt.Println("started")
})
```

在goroutine启动之后被调用。与BeforeStart不同，调用者在goroutine内部。

## AfterPanic

```golang
import "github.com/v2pro/plz/lang/routine"

routine.AfterPanic = append(routine.AfterPanic, func(recovered interface{}, kv []interface{}) {
	fmt.Println("panic", recovered)
})
```

在任何goroutine panic之后被调用。如果是`GoLongRunning`，则可能一个goroutine多次调用AfterPanic。调用者在goroutine内部。

## BeforeFinish

```golang
import "github.com/v2pro/plz/lang/routine"

routine.BeforeFinish = append(routine.BeforeFinish, func(kv []interface{}) {
	fmt.Println("finished")
})
```

在goroutine退出之前被调用。每个goroutine仅调用一次。调用者在goroutine内部。如果有panic，会先调用 `AfterPanic` 再调用 `BeforeFinish`。

# Tagging

## 通过 struct 自身定义 tag

```golang
type Order struct {
	OrderId   int `json:"order_id"`
	ProductId int `json:"product_id"`
}
```

这种写法是go原生支持的写法。缺点是

* 都在一个字符串上，写得比较长
* 无法支持不同上下文，提供不同的tags。比如一个字段是requird，取决于输入的来源方。
* 不支持string之外的数据类型
* 只能定义在 field 上，无法定义在 struct 自身，或者非 struct 的类型

扩展的 `github.com/v2pro/plz/tagging` 就是为了扩展打tag的能力。给任意的 go 类型都可以打 tag。

## 通过 struct method 定义 tag

```golang
import "github.com/v2pro/plz/tagging"

type Order struct {
	OrderId   int `json:"order_id"`
	ProductId int `json:"product_id"`
}

func (order *Order) DefineTags() tagging.Tags {
	return tagging.D(
		tagging.S("comment", "some more info about the struct itself"),
		tagging.F(&order.OrderId, "validation", "required", "tag_is_not_only_string", 100),
		tagging.F(&order.ProductId, "validation", "required"),
	)
}
```

## 通过 tagging.Define 定义 tag

```golang

import "github.com/v2pro/plz/tagging"

type Product struct {
	ProductId int `json:"product_id"`
}

tagging.Define(func(p *Product) tagging.Tags {
	return tagging.D(
		tagging.S("comment", "some more info about the struct itself"),
		tagging.F(&p.ProductId, "validation", "required", "tag_is_not_only_string", 100),
	)
})
```

# Accessor 

## Accessor 抽象

把对象的遍历问题抽象成以下四类Accessor

* 顺序读
* 随机读
* 顺序写
* 随机写

主要解决的问题是

* 提供类似 Java 反射的 API，避免 Go 的 reflect.Value 在运行时做过多的判断。
* 把 JSON/THRIFT 等序列化的形式也封装成 Accessor，抽象处理所有遍历对象图的需求。
* 给对象拷贝，验证等需求提供简化的语言抽象（屏蔽了 array/slice 等的区别）。

一个简单的例子

```golang
import (
"github.com/v2pro/plz"
_ "github.com/v2pro/plz/lang/nativeacc"
)

obj := []int{1, 2, 3}
accessor := plz.AccessorOf(reflect.TypeOf(obj))
elemAccessor := accessor.Elem()
accessor.IterateArray(obj, func(index int, elem interface{}) bool {
	fmt.Println(elemAccessor.Int(elem))
	return true
})
```

其中 `_ "github.com/v2pro/plz/lang/nativeacc"` 是为了注册原生对象的反射能力。

```golang
import "github.com/v2pro/plz/lang"

func init() {
	lang.Providers = append(lang.Providers, accessorOfNativeType)
}

func accessorOfNativeType(typ reflect.Type) lang.Accessor {
// ...
}
```

Accessor 的 interface 定义为

```golang
type Accessor interface {
	// === static ===
	fmt.GoStringer
	Kind() Kind
	// map
	Key() Accessor
	// array/map
	Elem() Accessor
	// struct
	NumField() int
	Field(index int) StructField
	// array/struct
	RandomAccessible() bool

	// === runtime ===
	// variant
	VariantElem(obj interface{}) (elem interface{}, elemAccessor Accessor)
	InitVariant(obj interface{}, template interface{}) (elem interface{}, elemAccessor Accessor)
	// map
	IterateMap(obj interface{}, cb func(key interface{}, elem interface{}) bool)
	FillMap(obj interface{}, cb func(filler MapFiller))
	// array/struct
	Index(obj interface{}, index int) (elem interface{}) // only when random accessible
	IterateArray(obj interface{}, cb func(index int, elem interface{}) bool)
	FillArray(obj interface{}, cb func(filler ArrayFiller))
	// primitives
	Skip(obj interface{}) // when the value is not needed
	String(obj interface{}) string
	SetString(obj interface{}, val string)
	Bool(obj interface{}) bool
	SetBool(obj interface{}, val bool)
	Int(obj interface{}) int
	SetInt(obj interface{}, val int)
	Int8(obj interface{}) int8
	SetInt8(obj interface{}, val int8)
	Int16(obj interface{}) int16
	SetInt16(obj interface{}, val int16)
	Int32(obj interface{}) int32
	SetInt32(obj interface{}, val int32)
	Int64(obj interface{}) int64
	SetInt64(obj interface{}, val int64)
	Uint(obj interface{}) uint
	SetUint(obj interface{}, val uint)
	Uint8(obj interface{}) uint8
	SetUint8(obj interface{}, val uint8)
	Uint16(obj interface{}) uint16
	SetUint16(obj interface{}, val uint16)
	Uint32(obj interface{}) uint32
	SetUint32(obj interface{}, val uint32)
	Uint64(obj interface{}) uint64
	SetUint64(obj interface{}, val uint64)
	Float32(obj interface{}) float32
	SetFloat32(obj interface{}, val float32)
	Float64(obj interface{}) float64
	SetFloat64(obj interface{}, val float64)
	// pointer to memory address
	AddressOf(obj interface{}) uintptr
}
```

## 顺序读写的 Array

* Kind() 返回 lang.Array
* Elem() 返回数组成员的 Accessor
* RandomAccessible() 返回 false
* IterateArray() 支持
* FillArray() 支持

Go 的 slice/array 均封装为 Array 类型。但是 array 在 Fill 的时候受长度限制。

## 随机读写的 Array

* RandomAccessible() 返回 true
* Index() 支持

## 顺序读写的 Map

* Kind() 返回 lang.Map
* Key() 返回 Map key 的 Accessor
* Elem() 返回 Map value 的 Accessor
* RandomAccessible() 返回 false，map无需支持随机读写
* IterateMap() 支持
* FillMap() 支持

## 顺序读写的 Struct

* Kind() 返回 lang.Struct
* NumField() 支持
* Field() 支持
* RandomAccessible() 返回 false
* IterateArray() 支持
* FillArray() 支持

Struct 和 Array 是非常类似的。区别在于 Array 只有一个 Elem Accessor。而 Struct 的每个 Field 都有独自的 Accessor。

## 随机读写的 Struct

* RandomAccessible() 返回 true
* Index() 支持
