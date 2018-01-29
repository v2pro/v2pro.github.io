---
layout: default
title: plz config
---

config：配置读取和 A/B 测试

* TOC
{:toc}

# 接口设计原则

配置是一个对象，而不就是一个key/value。这个对象可以任意复杂，由业务自己决定。简单的情况下，配置就是一个string作为key的map可以表达的。复杂的配置可能是一个规则列表，需要用规则引擎来解读。配置的 API 提供通用的接口来获得配置对象。配置提供 SPI 来解析不同的格式的配置文件。

配置是一个高可用的接口。意味着使用方不需要为配置读取进行兜底。读取配置不需要返回error，然后每次读取配置的时候都要处理这个error。配置应该永远都在那里，始终是可用的状态。虽然 plz 不是具体的实现，但是我们假设配置实际上是缓存在内存里的。

配置不是一成不变的。对于不同的请求，我们可能需要使用不同的配置。所以配置不是静态的，而是看人下菜碟的，也就是支持 A/B 测试。

# 配置 API

```go
func GetObject(namespace string, objectName string, targetKV ...string) interface{} 
```

配置的 API 只有一个函数。就是用 namespace/objectName 拿一个配置对象。这个配置对象是什么类型，由业务代码自己决定。传入的 targetKV 用于给不同的请求返回不同的配置使用。比如

```
cfg := GetObject("ui", "style", "userCity", "beijing").(*UIStyleConfig)
```

这里传入的 userCity 就可以用于内部进行 A/B 测试的判断。具体的 A/B 规则本身也是一个配置文件。

```go
func ShouldUse(namespace string, objectName string, targetKV ...string) bool
```

当配置的值是一个bool类型的时候，可以用 `ShouldUse` 这个快捷方式。实际上等价于

```
shouldDropOrder := GetObject("toggle", "shouldDropOrder", "sessionId", "xxxx").(bool)
```

# 配置 SPI

```
type Parser interface {
	Parse(data []byte) (interface{}, error)
}

func RegisterParser(dataFormat string, parser Parser)
func RegisterParserByFunc(dataFormat string, f func(data []byte) (interface{}, error))
```

注册自定义的数据格式的 parser

# A/B 规则引擎的语法

```go
type rawToggle struct {
	PreprocessorFn   string
	PreprocessorArgs map[string]interface{}
	Variants         []*variant
	DefaultVariant   itemName
}

type variant struct {
	ItemName itemName
	RuleFn   string
	RuleArgs map[string]interface{}
}
```
