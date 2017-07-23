# Template Expansion

Wombat Project: [https://github.com/v2pro/wombat](https://github.com/v2pro/wombat)

Generics is more than simple type subsitution. In C++, template specialization evolved into a set of techniques to do compile time calculation, which leads to constexpr added to the language. In D, we have static if, which supports a subset of the D language in the compile time. Even if Go 2.0 starts to support Generics, it will take some time to same level of maturity. Code generation will continue to be a possible solution for a long time. Hand-rolled code generator will have scalability issue, we can support complex logic such as one generic function calling another one. The goal of [wombat](https://github.com/v2pro/wombat) is to build a large-scale code generation framework, which makes complex utility reusable. The utility can be as simple as compare, max, or as complex as json encoding/decoding.

# Simple Example

To define a generic function

{% raw %}
```golang
var compareSimpleValue = generic.DefineFunc("CompareSimpleValue(val1 T, val2 T) int").
	Param("T", "the type of value to compare").
	Source(`
if val1 < val2 {
	return -1
} else if val1 == val2 {
	return 0
} else {
	return 1
}`)
```
{% endraw %}

To test it

{% raw %}
```golang
func init() {
	generic.DynamicCompilationEnabled = true
}

func Test_compare_int(t *testing.T) {
	should := require.New(t)
	f := generic.Expand(compareSimpleValue, "T", generic.Int).
	(func(int, int) int)
	should.Equal(-1, f(3, 4))
	should.Equal(0, f(3, 3))
	should.Equal(1, f(4, 3))
}
```
{% endraw %}

Notice that, in init() we turned on dynamic compilation. When the test runs, the code is generated & compiled then loaded as plugin. So this makes writing and testing generic function quite easy.

# Static Code Generation

Although wombat supports dynamic compliation, it is not recommended in production environment. It should be used a development tools. For generic function user, they need to do static code generation. Every `generic.Expand` need a corresponding `generic.Declare` defined in init().

```golang
func init() {
	generic.Declare(compareSimpleValue, "T", generic.Int)
}

func xxx() {
	f := generic.Expand(compareSimpleValue, "T", generic.Int).
	(func(int, int) int)
	f(3, 4)
}
```

Then use `go install github.com/v2pro/wombat/cmd/codegen` to compile codegen. Then use codegen

```
codegen -pkg path-to-your-pkg
```

The codegen will import your package, get declarations from side effect of init(), and produce a `generated.go` file back into your package. The functions defined in generated.go will be used when `generic.Expand`. If missing corresponding `generic.Declare` and without dynamic compilation, `generic.Expand` will fail.

# Expand-time calculation

If we need to compare more than simple value, for example supporting pointer to int. Then previous generic function can not support it, as `*int` can not be compared directly. So in the expand-time, we need to choose different implementation for different types.

{% raw %}
```golang
var ByItself = generic.DefineFunc("CompareByItself(val1 T, val2 T) int").
	Param("T", "the type of value to compare").
	Generators("dispatch", dispatch).
	Source(`
{{ $compare := expand (.T|dispatch) "T" .T }}
return {{$compare}}(val1, val2)`)

func dispatch(typ reflect.Type) string {
	switch typ.Kind() {
	case reflect.Int:
		return "CompareSimpleValue"
	case reflect.Ptr:
		return "ComparePtr"
	}
	panic("unsupported type: " + typ.String())
}
```
{% endraw %}

`dispatch` is just a Go function, but can be used in the template. It chooses implementation by types, then the chosen template will be expanded. `(.T|dispatch)` is syntax of `text/template`, which calls dispatch with T as argument.

# Recursion

ComparePtr其实无法确认自己一定是调用CompareSimpleValue。因为可能还有`**int`，以及`***int`这样的情况。所以，ComparePtr在对指针进行取消引用之后，再次调用CompareByItself进行递归展开模板。

{% raw %}
```golang
func init() {
	ByItself.ImportFunc(comparePtr)
}

var comparePtr = generic.DefineFunc("ComparePtr(val1 T, val2 T) int").
	Param("T", "the type of value to compare").
	ImportFunc(ByItself).
	Source(`
{{ $compare := expand "CompareByItself" "T" (.T|elem) }}
return {{$compare}}(*val1, *val2)`)
```
{% endraw %}

`ByItself.ImportFunc(comparePtr)` 是为了避免循环引用自身而引入的。否则两个函数就会循环引用，导致编译失败。具有了这样的函数模板化的能力，我们可以把JSON编解码这样的复杂的utility也用模板的方式写出来。
