# Goal

We want to use a simple interface, such as 

```golang
func Compare(
  obj1 interface{},
  obj2 interface{}) int {
  // ...
}
```

The actual implementation should be concrete (generics expanded):

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
