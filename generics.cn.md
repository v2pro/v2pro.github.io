# 泛型展开

泛型展开不是简单的类型替换。在C++中有模板偏特化，以及由此发展出来一系列实现编译期计算的奇技淫巧，直到最后以constexpr变成语言的一部分。D语言的static if也是类似的，在编译期实现了D语言的一个子集。在 Go 2.0 中即便支持了泛型，要达到D语言的高度，可能还需要很长的路要走。所以目前最佳的方案还是用代码生成的方案。但是纯手写的代码生成没有办法做到很复杂的泛型代码的组合，比如一个泛型函数调用另外一个泛型函数之类的。所以 wombat 的实现目标是设计一个能够支撑大规模代码生成的机制，使得复杂的utility能够被广泛复用。这些utility可能简单的如compare，max，复杂得如json编解码。

# 最简单的例子

定义一个泛型的函数

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

测试一个泛型的函数

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

注意，在init的时候，我们开启了动态编译。这样在测试的时候，实际上是直接在执行的时候生成代码，并用plugin的方式加载的。这样测试泛型代码就能随写随测，仿佛和用反射写的代码是一样的。实现的原理是 go 1.8 的 plugin。

# 静态代码生成

wombat 虽然支持动态编译，但是不推荐上生产环境，只是用于加速泛型函数开发效率的一种手段。泛型函数的用户，还是应该用静态代码生成的方式来使用。需要静态生成，就需要在使用一个泛型的函数前，先进行声明。声明在 init() 里定义哪些模板函数的哪些类型展开会被用到

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

用 `go install github.com/v2pro/wombat/cmd/codegen` 编译出代码生成器。然后执行

```
codegen -pkg path-to-your-pkg
```

然后会在你的包下面生成 generated.go 文件。这样 `generic.Expand` 就会使用生成的代码了。如果使用之前少了对应的`generic.Declare`，同时又没有开启动态编译，在Expand的时候就会报错。

# 泛型展开时计算

如果需求不仅仅是支持int，还要支持int的指针。前面实现的函数模板是无法支持的。所以我们需要能够，在泛型展开的时候进行类型判断，选择不同的实现。

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

其中dispatch就是一个go语言实现的函数，可以在展开模板的时候被调用，用于选择具体的实现。然后调用expand来把对应的模板再展开，然后调用。

# 递归展开

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

# 泛型容器

除了支持模板函数之外，struct也可以加模板。写法如下：

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

其中固定了一个模板参数叫，I。这个是指模板struct需要实现的interface。比如，如果用`<int,string>`来展开struct，对应的interface应该是：

```golang
type IntStringPair interface {
	First() int
	SetFirst(val int)
	Second() string
	SetSecond(val string)
}
```

使用的代码需要用这个interface来创建pair的实例：

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
