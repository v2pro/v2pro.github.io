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

# 对象拷贝

统一的一个api支持各种类型的编解码和对象拷贝。在 .net 里有一个库叫 AutoMapper，这个和它是类似的。

```golang
func Copy(dst, src interface{}) error
```

框架的很大一部分作用就是处理输入输出，这就少不了绑定对象。比如 HTTP 参数，比如 SQL 的返回。
所有的对象绑定类型的问题都可以用 Copy 解决。具体的 Copy 实现，由 SPI 提供

```golang
type Copier interface {
	Copy(dst interface{}, src interface{}) error
}

var CopierProviders = []func(dstAccessor, srcAccessor lang.Accessor) (Copier, error){}
```

## 普通 Go 对象拷贝

```golang
import (
	"fmt"
	_ "github.com/v2pro/wombat/cp"
	"github.com/v2pro/plz"
)

func Example_copy_struct() {
	type UserProperties struct {
		City string
		Age  int
	}
	type User struct {
		FirstName  string
		LastName   string
		Tags       []int
		Properties *UserProperties
	}
	type UserInfo struct {
		FirstName  *string
		LastName   *string
		Tags       []int
		Properties map[string]interface{}
	}
	userInfo := UserInfo{}
	plz.Copy(&userInfo, User{
		FirstName: "A",
		LastName:  "B",
		Tags:      []int{1, 2, 3},
		Properties: &UserProperties{
			"C",
			30,
		},
	})
	fmt.Println(*userInfo.FirstName)
	fmt.Println(*userInfo.LastName)
	fmt.Println(userInfo.Tags)
	fmt.Println(userInfo.Properties["City"])
	fmt.Println(userInfo.Properties["Age"])
	// Output:
	// A
	// B
	// [1 2 3]
	// C
	// 30
}
```

## HTTP Query 编解码

```golang
import (
	"strconv"
	"bytes"
	"net/http"
	"github.com/v2pro/plz"
	"fmt"
	_ "github.com/v2pro/wombat/cp_http"
)

func Example_bind_http_request() {
	body := "k1=v1&k2=v2"
	httpReq, _ := http.NewRequest("POST", "/url", bytes.NewBufferString(body))
	httpReq.Header.Add("Content-Type", "application/x-www-form-urlencoded")
	httpReq.Header.Add("Content-Length", strconv.Itoa(len(body)))
	httpReq.Header.Add("Traceid", "1000")
	httpReq.ParseForm()

	type MyRequest struct {
		Url     string `http:"Url"`
		TraceId string `header:"Traceid"`
		K1      string `form:"k1"`
		K2      string `form:"k2"`
	}

	myReq := MyRequest{}
	plz.Copy(&myReq, httpReq)
	fmt.Println(myReq.Url)
	fmt.Println(myReq.TraceId)
	fmt.Println(myReq.K1)
	fmt.Println(myReq.K2)

	// Output:
	// /url
	// 1000
	// v1
	// v2
}
```

## JSON 编解码

```golang
import (
	"github.com/v2pro/plz/lang/tagging"
	"github.com/v2pro/plz"
	"fmt"
	_ "github.com/v2pro/wombat/cp_json"
)

func Example_encode_json() {
	type User struct {
		FirstName string `json:"first_name"`
		LastName  string `json:"last_name"`
		Tags      []int `json:"tags"`
	}
	tagging.Define(new(User), "codec", "json")

	output := []byte{}
	plz.Copy(&output, User{"A", "B", []int{1, 2, 3}})
	fmt.Println(string(output))
	// Output:
	// {"first_name":"A","last_name":"B","tags":[1,2,3]}
}
```

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

## Max/Min

```golang
import (
	"testing"
	"github.com/stretchr/testify/require"
	"github.com/v2pro/plz"
)

func Test_max_min(t *testing.T) {
	should := require.New(t)
	should.Equal(3, plz.Max(1, 3, 2))
	should.Equal(1, plz.Min(1, 3, 2))

	type User struct {
		Score int
	}
	should.Equal(User{3}, plz.Max(
		User{1}, User{3}, User{2},
		"Score"))
}
```

# LINQ

用类 SQL 的语法来操作对象图，简化 Go 的泛型编程。
