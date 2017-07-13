# Generics implemented with Go 1.8 plugin

A framework to implement generic functions

# Goal

We want to use a simple interface, such as 

```golang
func Compare(
  obj1 interface{},
  obj2 interface{}) int {
  // ...
}
```

So that the code can be simple, just as if we have function overloading like Java. But the actual implementation should be concrete, without reflection:

```golang
func Compare_int(
  obj1 interface{},
  obj2 interface{}) int {
  // end of signature
  return typed_Compare_int(
    obj1.(int),
    obj2.(int))
}
func typed_Compare_int(
  obj1 int,
  obj2 int) int {
  // end of signature
  if (obj1 < obj2) {
    return -1
  } else if (obj1 == obj2) {
    return 0
  } else {
    return 1
  }
}
```

To bind `Compare` to `Compare_int` we need a func cache `map[reflect.Type]func(interface{}, interface{})int`. 
Then the next question is how to get a instance of `Compare_int`.
There are two choices:

* Static codegen, and include the codegen in compliation
* Static codegen, and compile a plugin. Load plugin to fill the func cache.
* Dynamic codegen and dynamic compilation and load plugin dynamically

Wombat has implemented both static codegen and dynamic codegen. When using static codegen, the code looks like:

```golang
func init() {
	wombat.CompilePlugin("/tmp/fp_test.so", func() {
		plz.Max(1, 3, 2)
	}, func() {
		plz.Max(
			User{1}, User{3}, User{2},
			"Score")
	})
	wombat.LoadPlugin("/tmp/fp_test.so")
	wombat.DisableDynamicCompilation()
}

type User struct {
	Score int
}

func Test_max_min(t *testing.T) {
	should := require.New(t)
	should.Equal(3, plz.Max(1, 3, 2))
	should.Equal(User{3}, plz.Max(
		User{1}, User{3}, User{2},
		"Score"))
}
```

# Generics

The generics definition of a function

{% raw %}
```golang
func init() {
	F.Dependencies["cmpSimpleValue"] = F
}

var F = &gen.FuncTemplate{
	Dependencies: map[string]*gen.FuncTemplate{
	// set in init()
	// "cmpSimpleValue": F,
	},
	Variables: map[string]string{
		"T": "the type to compare",
	},
	FuncName: `Compare_{{ .T|symbol }}`,
	Source: `
{{ if .T|isPtr }}
	{{ $compareElem := gen "cmpSimpleValue" "T" (.T|elem) }}
	{{ $compareElem.Source }}
	func {{ .funcName }}(
		obj1 interface{},
		obj2 interface{}) int {
		// end of signature
		return typed_{{ .funcName }}(
			obj1.({{ .T|name }}),
			obj2.({{ .T|name }}))
	}
	func typed_{{ .funcName }}(
		obj1 {{ .T|name }},
		obj2 {{ .T|name }}) int {
		// end of signature
		return typed_{{ $compareElem.FuncName }}(*obj1, *obj2)
	}
{{ else }}
	func {{ .funcName }}(
		obj1 interface{},
		obj2 interface{}) int {
		// end of signature
		return typed_{{ .funcName }}(
			obj1.({{ .T|name }}),
			obj2.({{ .T|name }}))
	}
	func typed_{{ .funcName }}(
		obj1 {{ .T|name }},
		obj2 {{ .T|name }}) int {
		// end of signature
		if (obj1 < obj2) {
			return -1
		} else if (obj1 == obj2) {
			return 0
		} else {
			return 1
		}
	}
{{ end }}
`,
}
```
{% endraw %}

The syntax is golang `text/template`.
The function is expanded into concrete instance by `gen.Compile`:

```golang
func Gen(typ reflect.Type) func(interface{}, interface{}) int {
	switch typ.Kind() {
	case reflect.Ptr, reflect.Int, reflect.Int8, reflect.Int16:
		funcObj := gen.Compile(F, `T`, typ)
		return funcObj.(func(interface{}, interface{}) int)
	}
	return nil
}
```

If dynamic compilation is not disabled, `gen.Compile` will generate go source code, compile a plugin and load it.
If dynamic compilation is disabled, `wombat.CompilePlugin` should be used at build time. `wombat.LoadPlugin` should be manually called in the runtime `init()`.

# Dependency

Generic function can call other generic function. Such as `Max` will dependend on `Compare`.

{% raw %}
```golang
var F = &gen.FuncTemplate{
	Dependencies: map[string]*gen.FuncTemplate{
		"cmpSimpleValue": cmpSimpleValue.F,
	},
	Variables: map[string]string{
		"T": "the type to max",
	},
	FuncName: `Max_{{ .T|symbol }}`,
	Source: `
{{ $compare := gen "cmpSimpleValue" "T" .T }}
{{ $compare.Source }}
func {{ .funcName }}(objs []interface{}) interface{} {
	currentMax := objs[0].({{ .T|name }})
	for i := 1; i < len(objs); i++ {
		typedObj := objs[i].({{ .T|name }})
		if typed_{{ $compare.FuncName }}(typedObj, currentMax) > 0 {
			currentMax = typedObj
		}
	}
	return currentMax
}
func typed_{{ .funcName }}(objs []{{ .T|name }}) {{ .T|name }} {
	currentMax := objs[0]
	for i := 1; i < len(objs); i++ {
		if typed_{{ $compare.FuncName }}(objs[i], currentMax) > 0 {
			currentMax = objs[i]
		}
	}
	return currentMax
}`,
}
```
{% endraw %}

when we compile `Max`, `Compare` will be also compiled into same plugin.

```golang
func Gen(typ reflect.Type) func([]interface{}) interface{} {
	switch typ.Kind() {
	case reflect.Int, reflect.Int8:
		funcObj := gen.Compile(F, `T`, typ)
		return funcObj.(func([]interface{}) interface{})
	}
	return nil
}
```

The actual generated source is:

```golang
func Compare_int(
	obj1 interface{},
	obj2 interface{}) int {
	// end of signature
	return typed_Compare_int(
		obj1.(int),
		obj2.(int))
}
func typed_Compare_int(
	obj1 int,
	obj2 int) int {
	// end of signature
	if (obj1 < obj2) {
		return -1
	} else if (obj1 == obj2) {
		return 0
	} else {
		return 1
	}
}
func Max_int(objs []interface{}) interface{} {
	currentMax := objs[0].(int)
	for i := 1; i < len(objs); i++ {
		typedObj := objs[i].(int)
		if typed_Compare_int(typedObj, currentMax) > 0 {
			currentMax = typedObj
		}
	}
	return currentMax
}
func typed_Max_int(objs []int) int {
	currentMax := objs[0]
	for i := 1; i < len(objs); i++ {
		if typed_Compare_int(objs[i], currentMax) > 0 {
			currentMax = objs[i]
		}
	}
	return currentMax
}
```

You can notice `typed_Max_int` calls `typed_Compare_int` directly, without going through `Compare_int`, this saves the cost of `interface{}`. 

# Potentials

With a generic function generation framework, we can build all kinds of utilities. Such as

* Max/min
* Map
* Filter
* Sorted

More importantly, we can build a generic implementation of `Copy`:

```golang
func Copy(dst, src interface{}) error
```

It can be used to copy go objects, copy json into go object (like json.Unmarshal), copy go object into json (like json.Marshal), copy http request into your request, copy sql rows into your model. Which binding to use is a "generic function compile" time calculation, depending on the incoming dstination type and source type, we use different copy implementation, such as: https://github.com/v2pro/wombat/blob/master/cp/cpStatically/init.go 
