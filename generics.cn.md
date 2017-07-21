# 泛型展开

泛型展开不是简单的类型替换。在C++中有模板偏特化，以及由此发展出来一系列实现编译期计算的奇技淫巧，直到最后以constexpr变成语言的一部分。D语言的static if也是类似的，在编译期实现了D语言的一个子集。在 Go 2.0 中即便支持了泛型，要达到D语言的高度，可能还需要很长的路要走。所以目前最佳的方案还是用代码生成的方案。但是纯手写的代码生成没有办法做到很复杂的泛型代码的组合，比如一个泛型函数调用另外一个泛型函数之类的。所以 wombat 的实现目标是设计一个能够支撑大规模代码生成的机制，使得复杂的utility能够被广泛复用。这些utility可能简单的如compare，max，复杂得如json编解码。

# 最简单的例子

定义一个泛型的函数

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

测试一个泛型的函数

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

注意，在init的时候，我们开启了动态编译。这样在测试的时候，实际上是直接在执行的时候生成代码，并用plugin的方式加载的。这样测试泛型代码就能达到和反射的实现一样的高效。

使用一个泛型的函数

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
因为没有开启动态编译，所以调用`generic.Expand`会失败。需要用 `go install github.com/v2pro/wombat/cmd/codegen` 编译出代码生成器。然后执行

```
codegen -pkg path-to-your-pkg
```

然后会在你的包下面生成 generated.go 文件。这样运行时`generic.Expand` 就不会报错了。

