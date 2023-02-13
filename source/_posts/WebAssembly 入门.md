---
title: WebAssembly 入门
date: 2021-9-24 16:20:53
tags:
  - Golang
  - Rust
  - WebAssembly
categories: 学习笔记
---

> 所有可以用 WebAssembly 实现的终将会用 WebAssembly 实现。

# 前言

> WebAssembly 是一种新的编码方式，可以在现代的网络浏览器中运行 － 它是一种低级的类汇编语言，具有紧凑的二进制格式，可以接近原生的性能运行，并为诸如 C / C ++等语言提供一个编译目标，以便它们可以在 Web 上运行。它也被设计为可以与 JavaScript 共存，允许两者一起工作。

WebAssembly 不是为了取代 JavaScript，而是一种补充，在对性能或是对交互体验要求高的场景有一席之地。并且其产物可以在多个环境中使用，对于我们前端来说，他可以调用 JavaScript 函数，包括对 DOM 树的操作。接下来我将用 go 和 rust 带你熟悉 WebAssembly 的使用。

# Hello World

首先我们先实现最简单的一个 WebAssembly，在网页中弹出 Hello World。

## Go

需要注意的是，如果你使用 vscode，需要先新建.vscode 文件夹，设置 settings.json。

```json
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "go.toolsEnvVars": {
    "GOOS": "js",
    "GOARCH": "wasm"
  }
}
```

新建文件 main.go，使用 Go 语言内置的 syscall/js 包，调用 alert。

```golang
// main.go
package main

import "syscall/js"

func main() {
        alert := js.Global().Get("alert")
        alert.Invoke("Hello World!")
}

```

```bash
GOOS=js GOARCH=wasm go build -o static/main.wasm

```

将其拷贝至我们的目录

```bash
cp "$(go env GOROOT)/misc/wasm/wasm_exec.js" static

```

然后我们新建 html，引入`static/main.wasm` 和 `static/wasm_exec.js`。

```html
<html>
  <script src="static/wasm_exec.js"></script>
  <script>
    const go = new Go();
    WebAssembly.instantiateStreaming(fetch('static/main.wasm'), go.importObject).then(result =>
      go.run(result.instance),
    );
  </script>
</html>
```

最后我们起一个服务器（我这里方便起见就用），就可以看到 Hello World 了。

```bash
parcel ./index.html

```

## Rust

rust 首先需要安装`wasm-pack`和 `cargo-generate` 。

```bash
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh

cargo install cargo-generate

```

然后我们通过 cargo-generate 来创建。

```bash
cargo generate --git https://github.com/rustwasm/wasm-pack-template
```

我们在 demo/src/lib.rs 下编写我们的代码。

```rust
extern crate wasm_bindgen;

#[wasm_bindgen]
extern {
    fn alert(s: &str);
}

#[wasm_bindgen(start)]
pub fn start() {
    alert(&format!("Hello World!"));
}

```

编写完毕后，我们使用 wasm-pack 去构建我们的代码。

```bash
wasm-pack build
```

构建成功后会生成一个 pkg 文件夹，里面存放的我们的 wasm。

```bash
pkg/
├── package.json
├── README.md
├── demo_bg.wasm
├── demo.d.ts
└── demo.js

```

然后我们快速起一个项目。

```bash
npm init wasm-app www
```

在`package.json`中添加上

```bash
{
  // ...
  "dependencies": {
    "demo": "file:../pkg",
    // ...
  }
}

```

在`index.js`中引入我们刚编写的代码。

```javascript
import * as wasm from 'demo';

wasm.greet();
```

# 注册函数

## Go

刚才我们是调用了 js 中的函数，现在我们要将函数注册给 js 调用。我们以斐波那契数列为例。

```golang
// main.go
package main

import "syscall/js"

func fib(i int) int {
        if i == 0 || i == 1 {
                return 1
        }
        return fib(i-1) + fib(i-2)
}

func fibFunc(this js.Value, args []js.Value) interface{} {
        return js.ValueOf(fib(args[0].Int())) // 结果用 js.ValueOf 包装
}

func main() {
        done := make(chan int, 0)
        js.Global().Set("fibFunc", js.FuncOf(fibFunc)) // 注册函数，被调用时会起一个新子协程运行
        <-done // 阻塞主协程
}

```

再度编译后，我们全局就有了"fibFunc"这个函数给我们调用了。

## Rust

rust 编写相对简单，我们只需要加上 wasm_bindgen 特征即可。

```rust
extern crate wasm_bindgen;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn fib(i: u32) -> u32 {
    match i {
        0 => 0,
        1 => 1,
        _ => fib(i - 1) + fib(i - 2),
    }
}
```

Rust 没有注册在全局上，而是在所在的包中。

```javascript
import * as wasm from 'demo';

wasm.fib(12);
```

# 操作 DOM

## GO

在全局中注册了 fib 后，我们需要触发他，我们可以直接对 dom 元素进行操作，假设我们有个如下的 html

```html
<input id="num" type="number" />
<button id="btn">Click</button>
<p id="ans">1</p>
```

当点击 button 后进行计算。

```golang
package main

import (
        "strconv"
        "syscall/js"
)

func fib(i int) int {
        if i == 0 || i == 1 {
                return 1
        }
        return fib(i-1) + fib(i-2)
}

var (
        document = js.Global().Get("document")
        numEle   = document.Call("getElementById", "num")
        ansEle   = document.Call("getElementById", "ans")
        btnEle   = js.Global().Get("btn")
)

func fibFunc(this js.Value, args []js.Value) interface{} {
        v := numEle.Get("value")
        if num, err := strconv.Atoi(v.String()); err == nil {
                ansEle.Set("innerHTML", js.ValueOf(fib(num)))
        }
        return nil
}

func main() {
        done := make(chan int, 0)
        btnEle.Call("addEventListener", "click", js.FuncOf(fibFunc))
        <-done
}

```

## Rust

Rust 首先需要在`Cargo.toml`中添加

```toml
[dependencies.web-sys]
version = "0.3.4"
features = [
    'Document',
    'Element',
    'EventTarget',
    'HtmlElement',
    'MouseEvent',
    'Node',
    'Window',
    'HtmlInputElement',
]
```

然后编写我们的代码

```rust
extern crate wasm_bindgen;
use std::cell::Cell;
use std::rc::Rc;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;

#[wasm_bindgen]
pub fn fib(i: u32) -> u32 {
    match i {
        0 => 0,
        1 => 1,
        _ => fib(i - 1) + fib(i - 2),
    }
}

#[wasm_bindgen(start)]
pub fn main() -> Result<(), JsValue> {
    let document = web_sys::window().unwrap().document().unwrap();
    let numEle = document
        .get_element_by_id("num")
        .unwrap()
        .dyn_into::<web_sys::HtmlInputElement>()
        .unwrap();
    let ansEle = document.get_element_by_id("ans").unwrap();
    let btnEle = document.get_element_by_id("btn").unwrap();

    let clouse = Closure::wrap(Box::new(move |event: web_sys::MouseEvent| {
        let num = numEle.value().clone().parse::<u32>().unwrap();
        ansEle.set_inner_html(&fib(num).to_string());
    }) as Box<dyn FnMut(_)>);
    btnEle.add_event_listener_with_callback("click", clouse.as_ref().unchecked_ref())?;

    Ok(())
}

```

# 异步

在我们 js 中，回调函数是常见的场景，我们修改下 fib，支持一下回调方法

## Go

```golang
package main

import (
        "syscall/js"
        "time"
)

func fib(i int) int {
        if i == 0 || i == 1 {
                return 1
        }
        return fib(i-1) + fib(i-2)
}

func fibFunc(this js.Value, args []js.Value) interface{} {
        callback := args[len(args)-1]
        go func() {
                time.Sleep(3 * time.Second)
                v := fib(args[0].Int())
                callback.Invoke(v)
        }()

        js.Global().Get("ans").Set("innerHTML", "Waiting 3s...")
        return nil
}

func main() {
        done := make(chan int, 0)
        js.Global().Set("fibFunc", js.FuncOf(fibFunc))
        <-done
}
```

你可能会说，都 1202 年了，谁还用回调，都是 promise，我要用 async/await

```golang
package main

import (
    "fmt"
    "syscall/js"
    "time"
)

func NewPromise() (p js.Value, resolve func(interface{}), reject func(interface{})) {
    var callback js.Func
    callback = js.FuncOf(func(_ js.Value, args []js.Value) interface{} {
        defer callback.Release()

        resolve = func(value interface{}) {
            args[0].Invoke(value)
        }

        reject = func(value interface{}) {
            args[1].Invoke(value)
        }

        return js.Undefined()
    })

    p = js.Global().Get("Promise").New(callback)

    return
}

func Await(p js.Value) (js.Value, error) { // 用于在go中进行await
    resChan := make(chan js.Value)

    var then js.Func

    then = js.FuncOf(func(_ js.Value, args []js.Value) interface{} {
        resChan <- args[0]
        return nil
    })

    defer then.Release()

    errChan := make(chan error)

    var catch js.Func

    catch = js.FuncOf(func(_ js.Value, args []js.Value) interface{} {
        errChan <- js.Error{args[0]}
        return nil
    })

    defer catch.Release()

    select {
    case res := <-resChan:
        return res, nil
    case err := <-errChan:
        return js.Undefined(), err
    }

}

func fib(i int) int {
    if i == 0 || i == 1 {
        return 1
    }
    return fib(i-1) + fib(i-2)
}

func fibFunc(this js.Value, args []js.Value) interface{} {
    var resPromise, resolve, reject = NewPromise()
    go func() {
        defer func() {
            if r := recover(); r != nil {
                if err, ok := r.(error); ok {
                    reject(fmt.Sprintf("fibFunc: panic: %+v\n", err))
                } else {
                    reject(fmt.Sprintf("fibFunc: panic: %v\n", r))
                }
            }
        }()
        time.Sleep(3 * time.Second)
        v := Await(fib(args[0].Int()))
        resolve(v)
    }()

    js.Global().Get("ans").Set("innerHTML", "Waiting 3s...")
    return resPromise
}

func main() {
    done := make(chan int, 0)
    js.Global().Set("fibFunc", js.FuncOf(fibFunc))
    <-done
}
```

## Rust

Rust 非常简单，会将 promise 与 future 自动进行转换，如果我们需要自己新建 promise，可以使用 wasm_bindgen_futures::JsFuture 中调用。

```rust
use async_std::task::sleep;
use std::time;
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::*;

#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(catch)] // 引入js的async函数, 调用时只需要await调用即可
    async fn async_func_3() -> Result<JsValue, JsValue>;
    #[wasm_bindgen(catch)]
    async fn async_func_4() -> Result<(), JsValue>;
}

pub fn fib(i: u32) -> u32 {
    match i {
        0 => 0,
        1 => 1,
        _ => fib(i - 1) + fib(i - 2),
    }
}

#[wasm_bindgen]
pub async fn sleep_fib(i: u32) -> u32 { // rust会自动转换成promise
    sleep(time::Duration::from_secs(1)).await;
    fib(i)
}
```

# 试一试

我们使用 WebAssembly 做一个简易画板来作为我们的实践项目。

## Go

```golang
package main

import (
    "syscall/js"
)

var (
    pressed = false
)

func main() {
    done := make(chan int, 0)
    document := js.Global().Get("document")
    canvas := document.Call("createElement", "canvas")
    document.Call("body").Call("appendChild", canvas)
    canvas.Set("width", 700)
    canvas.Set("height", 600)
    context := canvas.Call("getContext", "2d")
    canvas.Call("addEventListener", "mousedown", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        event := args[0]
        context.Call("beginPath")
        context.Call("moveTo", event.Call("offset_x"), event.Call("offset_y"))
        pressed = true
        return nil
    }))
    canvas.Call("addEventListener", "mousemove", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        event := args[0]
        context.Call("lineTo", event.Call("offset_x"), event.Call("offset_y"))
        context.Call("stroke")
        context.Call("beginPath")
        context.Call("moveTo", event.Call("offset_x"), event.Call("offset_y"))
        return nil
    }))
    canvas.Call("addEventListener", "mouseup", js.FuncOf(func(this js.Value, args []js.Value) interface{} {
        event := args[0]
        pressed = false
        context.Call("lineTo", event.Call("offset_x"), event.Call("offset_y"))
        context.Call("stroke")
        return nil
    }))
    <-done
}
```

```rust
use std::cell::Cell;
use std::rc::Rc;
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;

#[wasm_bindgen(start)] // 渲染dom时运行
pub fn start() -> Result<(), JsValue> {
    let document = web_sys::window().unwrap().document().unwrap();
    let canvas = document
        .create_element("canvas")?
        .dyn_into::<web_sys::HtmlCanvasElement>()
        .unwrap();

    document.body().unwrap().append_child(&canvas)?;
    canvas.set_width(700);
    canvas.set_height(600);
    canvas.style().set_property("border", "solid")?;

    let context = canvas
        .get_context("2d")?
        .unwrap()
        .dyn_into::<web_sys::CanvasRenderingContext2d>()?;

    let context = Rc::new(context);
    let pressed = Rc::new(Cell::new(false));

    {
        let context = context.clone();
        let pressed = pressed.clone();
        let closure = Closure::wrap(Box::new(move |event: web_sys::MouseEvent| {
            context.begin_path();
            context.move_to(event.offset_x() as f64, event.offset_y() as f64);
            pressed.set(true);
        }) as Box<dyn FnMut(_)>);
        canvas.add_event_listener_with_callback("mousedown", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let context = context.clone();
        let pressed = pressed.clone();
        let closure = Closure::wrap(Box::new(move |event: web_sys::MouseEvent| {
            if pressed.get() {
                context.line_to(event.offset_x() as f64, event.offset_y() as f64);
                context.stroke();
                context.begin_path();
                context.move_to(event.offset_x() as f64, event.offset_y() as f64);
            }
        }) as Box<dyn FnMut(_)>);
        canvas.add_event_listener_with_callback("mousemove", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    {
        let context = context.clone();
        let pressed = pressed.clone();
        let closure = Closure::wrap(Box::new(move |event: web_sys::MouseEvent| {
            pressed.set(false);
            context.line_to(event.offset_x() as f64, event.offset_y() as f64);
            context.stroke();
        }) as Box<dyn FnMut(_)>);
        canvas.add_event_listener_with_callback("mouseup", closure.as_ref().unchecked_ref())?;
        closure.forget();
    }

    Ok(())
}
```

# 6. 更进一步

由于 Go 存在运行时，从而会导致 go 的 wasm 非常大，因此我们需要进行进一步的优化，我个人推荐的是使用 [tinygo](https://tinygo.org/)，它会让你的 wasm 变得非常的小，但是仍需注意的是，tinygo 对 go 的标准库并不是全部支持，因此在选用 go 作为 wasm 的开发语言时，需要注意 tinygo 是否支持，[详情](https://tinygo.org/docs/reference/lang-support/stdlib/)。

并且 Go 对 async/await 相关的支持也不是很好，可能需要我们自己去封装一个（如 4 章中我封装的 await），但如果你使用 rust，这些问题都将没有，rust 也非常简单易学，相信聪明的你能 3 天上手，5 天出活。

在这最后的时候，我个人稍微表达下自己的看法，wasm 实际上是不能达到汇编级性能效果，并且他的生态链也与前端工具链关联不大，当你决定使用 wasm 需要考虑一下 js 是否能够完成，毕竟在浏览器的世界里 js 可没有怕过谁，就像《人月神话》中说的那样，软件工程没有银弹。
