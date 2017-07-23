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

`ComparePtr` can not be sure, it will always call `CompareSimpleValue`, given there are cases like `**int` or `***int`. So, `ComparePtr` dereferences the pointer, and use `CompareByItself` to do recursive template expansion.

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

Generic function used need to be imported by `ImportFunc`. In init() `ByItself.ImportFunc(comparePtr)` is to avoid compilation error due to cyclic dependency. With support of recursive template expansion, complex utility like JSON encoder/decoder can be written in this style.

# Generic Container

Besides generic function, generic struct is supported as well. The syntax is:

{% raw %}
```golang
var Pair = generic.DefineStruct("Pair").
	Source(`
{{ $T1 := .I | method "First" | returnType }}
{{ $T2 := .I | method "Second" | returnType }}

type {{.structName}} struct {
    first {{$T1|name}}
    second {{$T2|name}}
}

func (pair *{{.structName}}) SetFirst(val {{$T1|name}}) {
    pair.first = val
}

func (pair *{{.structName}}) First() {{$T1|name}} {
    return pair.first
}

func (pair *{{.structName}}) SetSecond(val {{$T2|name}}) {
    pair.second = val
}

func (pair *{{.structName}}) Second() {{$T2|name}} {
    return pair.second
}`)
```
{% endraw %}

There is a fixed template parameter called I, which means the interface of the expanded struct. For example, if we expand the pair with `<int,string>`, the interface should be defined as:

```golang
type IntStringPair interface {
	First() int
	SetFirst(val int)
	Second() string
	SetSecond(val string)
}
```

Then, we can use `IntStringPair` to create a instance of pair:

```golang
func init() {
	generic.DynamicCompilationEnabled = true
}

func Test_pair(t *testing.T) {
	type IntStringPair interface {
		First() int
		SetFirst(val int)
		Second() string
		SetSecond(val string)
	}
	should := require.New(t)
	intStringPairType := reflect.TypeOf(new(IntStringPair)).Elem()
	pair := generic.New(Pair, intStringPairType).(IntStringPair)
	should.Equal(0, pair.First())
	pair.SetFirst(1)
	should.Equal(1, pair.First())
}
```

# Type Inference

In previous example, we have two lines of code:

{% raw %}
```golang
{{ $T1 := .I | method "First" | returnType }}
{{ $T2 := .I | method "Second" | returnType }}
```
{% endraw %}

to get element types from its container type.

Template parameters support default value, for example:

{% raw %}
```golang
var ByItself = generic.DefineFunc("MaxByItself(vals T) E").
	Param("T", "array type").
	Param("E", "array element type", func(argMap generic.ArgMap) interface{} {
	return argMap["T"].(reflect.Type).Elem()
}).
	ImportFunc(compare.ByItself).
	Source(`
{{ $compare := expand "CompareByItself" "T" .E }}
currentMax := vals[0]
for i := 1; i < len(vals); i++ {
	if {{$compare}}(vals[i], currentMax) > 0 {
		currentMax = vals[i]
	}
}
return currentMax`)
```
{% endraw %}

By getting `int` from `[]int`,the user of the generic function, only need to specify its container type, the element type is "infered".
