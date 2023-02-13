---
title: Golang 单元测试极速入门
date: 2021-2-31 06:29:53
tags:
  - Golang
  - Unit Testing
categories: 学习笔记
---

> 编程之事，隔离是方向，起名是关键，测试是主角，调试是补充，版本控制是后悔药。

# 前言

一个庞大的项目，其环境的复杂度，逻辑的关联性是难以预估的。当你接手一个新项目的时候，你优化或者添加某些特性之后，是很难将所有的场景去人工测试的。而小小的几个测试却能帮你覆盖大部分场景。通过编写单元测试，它能够保证你的代码准确性，当持续迭代时，它将是你最可靠的伙伴。

本文将会通过使用 Go 语言带你极速入门单元测试，基准测试及 mock（gomock）模拟数据库。本人水平有限，有不足之处也请大家多多指点。

# Hello World

Go 语言推荐将测试代码和源代码放置于一处，测试源代码使用`_test.go`结尾。

```golang
// 源文件
package main

func Add(a number, b number) number {
    return a + b
}
```

```golang
// 测试文件
package main

import "testing"

func TestAdd(t *testing.T) {
    if res := Add(1, 1); res != 2 {
        t.Errorf("1 + 1 expected be 2, but result is %d", res)
    }
}

```

测试用例中需要引入 testing，并且每个测试函数需要传入唯一参数`*testing.T`，需要注意的是，基准测试是`*testing.B`,测试主函数是`*testing.M`。

然后我们运行`go test`，即可执行所有测试。

如果我们想要查看用例打印及结果，可以加上-v 参数，如果想要看测试覆盖率，可以加上-cover 参数。如果我们想要只运行一测试用例，我们可以使用-run 参数进行指定，如：`go test -run TestAdd -v`，支持通配符和部分正则。

# 表驱动测试

Go 语言支持一个测试中进行多个子测试，使用 t.Run 即可创建。

```golang
package main

import "testing"

func TestAdds(t *testing.T) {
    t.Run("first test", func(t *testing.T) {
        if res := Add(1, 1); res != 2 {
            t.Fatal("1 + 1 expected be 2, but result is %d", res)
        }
    })


    t.Run("second test", func(t *testing.T) {
        if res := Add(2, 1); res != 3 {
            t.Fatal("2 + 1 expected be 3, but result is %d", res)
        }
    })
}
```

注：`t.Error/t.Errorf`即使错误也会继续执行其他用例，而`t.Fatal/t.Fatalf`则会停止执行。借此我们可以使用表驱动测试的方法去执行多个测试。

```golang
func TestAdds(t *testing.T) {
        cases := []struct {
                Name           string
                A, B, Expected int
        }{
                {"first", 1, 1, 2},
                {"second", 2, 1, 3},
                {"third", 2, 0, 2},
        }

        for _, c := range cases {
                t.Run(c.Name, func(t *testing.T) {
                        if res := Add(c.A, c.B); res ！= c.Expected {
                                t.Fatalf("%d + %d expected %d, but result is %d ",c.A, c.B, c.Expected, res)
                        }
                })
        }
}
```

# t.Helper

由于我们的业务可能会有比较复杂的逻辑，我们往往会将次要逻辑抽离成一个个函数。

```golang
package main

import "testing"

type calcCase struct{ A, B, Expected int }

func createAddTestCase(t *testing.T, c *calcCase) {
        if ans := Mul(c.A, c.B); ans != c.Expected {
                t.Fatalf("%d + %d expected %d, but result is %d", c.A, c.B, c.Expected, ans)
        }

}

func TestAdds(t *testing.T) {
        createAddTestCase(t, &calcCase{1, 1, 2})
        createAddTestCase(t, &calcCase{2, 1, 3})
        createAddTestCase(t, &calcCase{2, 0, 0}) // wrong case
}
```

在这个测试中，Go 会告诉我们错误出现在第 9 行。但是这不是我们想要的，因为第 9 行被调用了 3 次，我们还需要人工去寻找那个用例的问题。

但是如果我们使用了`t.Helper`后，他将会清晰地告诉我们错误发生在第 18 行。他的作用就是报错时将输出函数调用者的信息，而不是函数的内部信息。

```golang
package main

import "testing"

type calcCase struct{ A, B, Expected int }

func createAddTestCase(t *testing.T, c *calcCase) {
        t.Helper()
        if ans := Mul(c.A, c.B); ans != c.Expected {
                t.Fatalf("%d + %d expected %d, but result is %d", c.A, c.B, c.Expected, ans)
        }

}

func TestAdds(t *testing.T) {
        createAddTestCase(t, &calcCase{1, 1, 2})
        createAddTestCase(t, &calcCase{2, 1, 3})
        createAddTestCase(t, &calcCase{2, 0, 0}) // wrong case
}
```

# setup 和 teardown

当我们的测试用例前后有相同逻辑时，我们可以使用 setup 和 teardown 来进行抽离。

```golang
func setup() {
        fmt.Println("Before all tests")
}

func teardown() {
        fmt.Println("After all tests")
}

func Test1(t *testing.T) {
        fmt.Println("I'm test1")
}

func Test2(t *testing.T) {
        fmt.Println("I'm test2")
}

func TestMain(m *testing.M) {
        setup()
        code := m.Run()
        teardown()
        os.Exit(code)
}
```

# 网络测试

对于我们互联网程序员来说，网络测试是必不可少的。假如我们有个接口需要进行测试。

```golang
func helloHandler(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("hello world"))
}
```

我们可以通过创建真实的网络进行测试。

```golang
import (
        "io/ioutil"
        "net"
        "net/http"
        "testing"
)

func handleError(t *testing.T, err error) {
        t.Helper()
        if err != nil {
                t.Fatal("failed", err)
        }
}

func TestConn(t *testing.T) {
        ln, err := net.Listen("tcp", "127.0.0.1:0")
        handleError(t, err)
        defer ln.Close()

        http.HandleFunc("/hello", helloHandler)
        go http.Serve(ln, nil)

        resp, err := http.Get("http://" + ln.Addr().String() + "/hello")
        handleError(t, err)

        defer resp.Body.Close()
        body, err := ioutil.ReadAll(resp.Body)
        handleError(t, err)

        if string(body) != "hello world" {
                t.Fatal("expected hello world, but got", string(body))
        }
}
```

但是还是推荐使用标准库 net/http/httptest 进行测试。

```golang
import (
        "io/ioutil"
        "net/http"
        "net/http/httptest"
        "testing"
)

func TestConn(t *testing.T) {
        req := httptest.NewRequest("GET", "http://example.com/foo", nil)
        w := httptest.NewRecorder()
        helloHandler(w, req)
        bytes, _ := ioutil.ReadAll(w.Result().Body)

        if string(bytes) != "hello world" {
                t.Fatal("expected hello world, but got", string(bytes))
        }
}
```

# 基准测试

基准测试会帮你测试你的函数性能。他必须以`Benchmark`开头，参数为`b *testing.B`，执行时需要携带`-bench`参数。

```golang
func BenchmarkHello(b *testing.B) {
    for i := 0; i < b.N; i++ {
        fmt.Sprintf("hello")
    }
}
```

基准测试的结果值含义如下:

```golang
type BenchmarkResult struct {
    N         int           // 迭代次数
    T         time.Duration // 基准测试花费的时间
    Bytes     int64         // 一次迭代处理的字节数
    MemAllocs uint64        // 总的分配内存的次数
    MemBytes  uint64        // 总的分配内存的字节数
}
```

如果我们只想测试某一部分的时间，我们可以使用`b.ResetTimer()` 先重置定时器

```golang
func BenchmarkHello(b *testing.B) {
    ... // dosomething...
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        fmt.Sprintf("hello")
    }
}
```

使用`RunParallel`测试并发性能。

```golang
func BenchmarkParallel(b *testing.B) {
        templ := template.Must(template.New("test").Parse("Hello, {{.}}!"))
        b.RunParallel(func(pb *testing.PB) {
                var buf bytes.Buffer
                for pb.Next() {
                        // 所有 goroutine 一起，循环一共执行 b.N 次
                        buf.Reset()
                        templ.Execute(&buf, "World")
                }
        })
}
```

# Gomock

当我们需要测试跟数据库相关的函数时，我们可以使用官方提供的 gomock。首先需要安装 gomock：

```bash
go get -u github.com/golang/mock/gomock
go get -u github.com/golang/mock/mockgen
```

假如我们有一个链接数据的待测函数：

```golang
// db.go
type DB interface {
        Get(key string) (int, error)
}

func GetFromDB(db DB, key string) int {
        if value, err := db.Get(key); err == nil {
                return value
        }

        return -1
}
```

现在我们需要测试`GetFromDB`这个函数，这时候就可以使用 gomock 了。首先我们需要使用 mockgen 生成 `db_mock.go`。一般传递三个参数。包含需要被 mock 的接口得到源文件 source，生成的目标文件 destination，包名 package。

```bash
$ mockgen -source=db.go -destination=db_mock.go -package=main
```

接下来我们编写测试用例：

```golang
// db_test.go
func TestGetFromDB(t *testing.T) {
        ctrl := gomock.NewController(t)
        defer ctrl.Finish() // 断言 DB.Get() 方法是否被调用

        m := NewMockDB(ctrl)
        m.EXPECT().Get(gomock.Eq("Tom")).Return(100, errors.New("not exist"))

        if v := GetFromDB(m, "Tom"); v != -1 {
                t.Fatal("expected -1, but got", v)
        }
}
```

其中 GetFromDB() 的逻辑是否正确(如果 DB.Get() 返回 error，那么 GetFromDB() 返回 -1)，而 NewMockDB() 的定义在 db_mock.go 中，由 mockgen 自动生成。

当然，除了刚才 Get 方法，gomock 还提供了调用次数，调用顺序和返回值动态设置等等方法。

常见断言方法如下：

- Eq(val)等于 val。
- Any() 任意入参。
- Not(val) 不等于 value 的值。
- Nil() None 值

```golang
m.EXPECT().Get(gomock.Eq("akita")).Return(0, errors.New("not exist"))
m.EXPECT().Get(gomock.Any()).Return(4396, nil)
m.EXPECT().Get(gomock.Not("akt")).Return(0, nil)
m.EXPECT().Get(gomock.Nil()).Return(0, errors.New("nil"))
```

并且我们不光可以使用 Return 返回确定的值，还可以使用 Do 执行某一操作，和 DoAndReturn 进行动态控制返回值。

```golang
m.EXPECT().Get(gomock.Not("akita")).Return(0, nil)
m.EXPECT().Get(gomock.Any()).Do(func(key string) {
    t.Log(key)
})
m.EXPECT().Get(gomock.Any()).DoAndReturn(func(key string) (int, error) {
    if key == "akt" {
        return 630, nil
    }
    return 0, errors.New("not exist")
})
```

我们还可以使用`Times()`、`MaxTimes()`、`MinTimes()`和`AnyTimes()`进行次数的断言。

```golang
func TestGetFromDB(t *testing.T) {
        ctrl := gomock.NewController(t)
        defer ctrl.Finish()

        m := NewMockDB(ctrl)
        m.EXPECT().Get(gomock.Not("akita")).Return(0, nil).Times(2)
        GetFromDB(m, "akita")
        GetFromDB(m, "akt")
}
```

调用顺序也可以通过`InOrder`来进行断言。

```golang
func TestGetFromDB(t *testing.T) {
        ctrl := gomock.NewController(t)
        defer ctrl.Finish() // 断言 DB.Get() 方法是否被调用

        m := NewMockDB(ctrl)
        o1 := m.EXPECT().Get(gomock.Eq("akita")).Return(0, errors.New("not exist"))
        o2 := m.EXPECT().Get(gomock.Eq("akt")).Return(630, nil)
        gomock.InOrder(o1, o2)
        GetFromDB(m, "akita")
        GetFromDB(m, "akt")
}
```

回到我们最开始的例子：

```golang
// db.go
type DB interface {
        Get(key string) (int, error)
}

func GetFromDB(db DB, key string) int {
        if value, err := db.Get(key); err == nil {
                return value
        }

        return -1
}
```

我们突然发现，我们对 DB 接口的 mock 并不能作用于 `GetFromDB()` 内部，这样写是没办法进行测试的。要想修改其实是很简单的，你只需要将接口 `db DB` 通过参数传递到`GetFromDB()`，那么就可以轻而易举地传入 Mock 对象了。

# 最后的话

千万不要忘了，高内聚，低耦合是软件工程的原则，职责单一，参数类型简单，与其他函数耦合度低的函数往往更容易测试。当出现“这种代码没法测试”的时候，往往需要对函数的写法进行思考重构，相信我，为了代码可测试而重构是值得的。
