---
title: 200行代码实现极简deno
date: 2022-05-17 16:36:40
tags:
  - deno
  - rust
categories: 学习笔记
---

# 前言

之前的文章都比较偏向服务端的库实现，可能离我们前端比较远，这一次我们用 rust 来构建一个极简 deno，他包含了：

- 调用 v8 执行 js 代码
- 在 v8 中注册 rust 代码
- 支持 snapshot 快速启动
- 支持 module 及 async（block 非异步）

设计图：

![architecture](/image/mini-deno/architecture.jpg)

# v8

v8 是 rust 对 v8 引擎的一个封装，他对 v8 的接口进行了一个映射，我们在 rust 侧只需要调用 v8 暴露的接口，即可操作 v8 引擎，比如如下代码，我们启动 v8，并使其跑一个 js 代码：

```rust
use serde::{Deserialize, Serialize};
use v8::{CreateParams, HandleScope, Isolate, Script, V8};

#[derive(Debug, Serialize, Deserialize)]
pub struct Data {
    pub status: usize,
    pub message: String,
}

fn main() {
    // initialize v8 engine
    init();
    // create an isolate
    let params = CreateParams::default();
    let isolate = &mut Isolate::new(params);

    // create handle scope
    let handle_scope = &mut HandleScope::new(isolate);
    // create context
    let context = v8::Context::new(handle_scope);
    // create context scope
    let context_scope = &mut v8::ContextScope::new(handle_scope, context);

    // javascript code
    let source = r#"
        function hello_world() {
            return {
                status: 200,
                message: "Hello World"
            };
        }
        hello_world();
    "#;

    let source = v8::String::new(context_scope, source).unwrap();
    let script = Script::compile(context_scope, source, None).unwrap();
    let result = script.run(context_scope).unwrap();
    let value: Data = serde_v8::from_v8(context_scope, result).unwrap();
    let serde_value: serde_json::Value = serde_v8::from_v8(context_scope, result).unwrap();
    println!("Result is: {:#?}", value);
    println!("Result is: {:#?}", serde_value);
}

fn init() {
    let platform = v8::new_default_platform(0, false).make_shared();
    V8::initialize_platform(platform);
    V8::initialize();
}
```

如果你对 v8 的运行环境不是太了解的话，可能对上面的代码会一头雾水，简单来说，每个 vm 创建在 isolate 下，isolate 是单线程的运行环境。handle 是 v8 中对每个数据的内存地址的映射，而 handle_scope 是管理他们的，而 context 在 handle_scope 之上，可以理解为每个代码块的部分，他可以内部将一些函数和对象绑定在 context 作用域内，原理图如下：

![v8](/image/mini-deno/v8.jpg)

# Runtime

其实，通过刚才的代码可以发现，如果直接调用 v8，会有很多额外的步骤，比如引擎的启动，scope，context 和 context_scope 的创建以及类型的转换等等，其实我们可以把其封装成为一个 runtime，暴露较少的接口，减少编写逻辑，比如我们的 runtime 是这么启动的：

```rust
use v8_live::JsRuntime;

fn main() {
    JsRuntime::init();
    let mut runtime = JsRuntime::new(None);

    let code = r#"
        function hello_world() {
            return {
                status: 200,
                message: "Hello World"
            };
        }
        hello_world();
    "#;

    let result = runtime.execute_script(code, false);
    println!("Result is: {:#?}", result);
}
```

而 runtime 的编写就需要进行一定的封装：

```rust
mod state;
mod utils;

use once_cell::sync::OnceCell;
use state::JsRuntimeState;
use utils::execute_script;
use v8::{
    CreateParams, FunctionCodeHandling, HandleScope, Isolate, OwnedIsolate, SnapshotCreator, V8,
};

pub type LocalValue<'a> = v8::Local<'a, v8::Value>;

pub struct JsRuntime {
    isolate: OwnedIsolate,
}

impl JsRuntime {
    pub fn init() { // 初始化v8
        static V8_INSTANCE: OnceCell<()> = OnceCell::new(); // 使用OnceCell保证多次init也只会执行一次
        V8_INSTANCE.get_or_init(|| {
            let platform = v8::new_default_platform(0, false).make_shared();
            V8::initialize_platform(platform);
            V8::initialize();
        });
    }

    // pub fn new(params: JsRuntimeParams) -> Self {
    pub fn new(snapshot: Option<Vec<u8>>) -> Self { // 新建runtime
        let mut params = CreateParams::default();
        let mut initialized = false;
        let isolate = Isolate::new(params);
        JsRuntime::init_isolate(isolate, initialized)
    }

    pub fn execute_script(
        &mut self,
        code: impl AsRef<str>,
        is_module: bool,
    ) -> Result<serde_json::Value, serde_json::Value> { // 执行js代码并返回结果
        let context = JsRuntimeState::get_context(&mut self.isolate);
        let handle_scope = &mut HandleScope::with_context(&mut self.isolate, context);
        match execute_script(handle_scope, code, is_module) {
            Ok(value) => Ok(serde_v8::from_v8(handle_scope, value).unwrap()),
            Err(error) => Err(serde_v8::from_v8(handle_scope, error).unwrap()),
        }
    }

    pub fn create_snapshot() -> Vec<u8> {
        todo!()
    }

    fn init_isolate(mut isolate: OwnedIsolate, initialized: bool) -> Self { // 初始化Isolate,相关
        let state = JsRuntimeState::new(&mut isolate);
        isolate.set_slot(state);
        let context = JsRuntimeState::get_context(&mut isolate);
        let scope = &mut HandleScope::with_context(&mut isolate, context);
        JsRuntime { isolate }
    }
}

// utils.rs
use crate::LocalValue;
use v8::{script_compiler, HandleScope, Promise, Script, ScriptOrigin, TryCatch};

pub fn execute_script<'a>(
    scope: &mut HandleScope<'a>,
    code: impl AsRef<str>,
    is_module: bool,
) -> Result<LocalValue<'a>, LocalValue<'a>> {
    let scope = &mut TryCatch::new(scope);
    let source = v8::String::new(scope, code.as_ref()).unwrap();
    let origin = create_origin(scope, "dummy.js", is_module);
    Script::compile(scope, source, Some(&origin))
            .and_then(|script| script.run(scope))
            .map_or_else(|| Err(scope.stack_trace().unwrap()), Ok)
}

// state.rs
use std::{cell::RefCell, rc::Rc};

use v8::{HandleScope, Isolate};

type GlobalContext = v8::Global<v8::Context>;

type JsRuntimeStateRef = Rc<RefCell<JsRuntimeState>>;

pub struct JsRuntimeState {
    context: Option<v8::Global<v8::Context>>,
}

impl JsRuntimeState {
    pub fn new(isolate: &mut Isolate) -> JsRuntimeStateRef {
        let context = {
            let handle_scope = &mut HandleScope::new(isolate);
            let context = v8::Context::new(handle_scope);
            v8::Global::new(handle_scope, context)
        };

        Rc::new(RefCell::new(JsRuntimeState { // 保证线程安全
            context: Some(context),
        }))
    }

    pub fn get_context(isolate: &mut Isolate) -> GlobalContext {
        let state = isolate.get_slot::<JsRuntimeStateRef>().unwrap().clone();
        let ctx = &state.borrow().context;
        ctx.as_ref().unwrap().clone()
    }

    pub fn drop_context(isolate: &mut Isolate) {
        let state = isolate.get_slot::<JsRuntimeStateRef>().unwrap().clone();
        state.borrow_mut().context.take();
    }
}
```

# 注册 rust 函数

像 deno 和 node.js 中，不光有 v8 自身的函数，也会有这些 runtime 为他们封装好的其他函数，这里我们就封装一个同步和异步的函数：

```rust
use v8_live::JsRuntime;

fn main() {
    JsRuntime::init();
    let mut runtime = JsRuntime::new(None);

    let code = r#"
        function hello_world() {
            print("Hello Rust"); // 同步打印
            // return {
            //     status: 200,
            //     message: "Hello World"
            // };
            return fetch("https://www.rust-lang.org/") // 异步fetch
        }
        hello_world();
    "#;

    let result = runtime.execute_script(code, false);
    println!("Result is: {:#?}", result);
}
```

首先我们先在 rust 测实现这两个函数

```rust
use v8::{FunctionCallbackArguments, HandleScope, ReturnValue};

use crate::utils::execute_script;

const GLUE: &str = include_str!("glue.js");

pub struct Extensions;

impl Extensions {
    pub fn install(scope: &mut HandleScope) {
        let bindings = v8::Object::new(scope);

        let name = v8::String::new(scope, "print").unwrap();
        let func = v8::Function::new(scope, print).unwrap();
        bindings.set(scope, name.into(), func.into()).unwrap();

        let name = v8::String::new(scope, "fetch").unwrap();
        let func = v8::Function::new(scope, fetch).unwrap();
        bindings.set(scope, name.into(), func.into()).unwrap();

        if let Ok(result) = execute_script(scope, GLUE) {
            let func = v8::Local::<v8::Function>::try_from(result).unwrap();
            let v = v8::undefined(scope).into();
            let args = [bindings.into()];
            func.call(scope, v, &args).unwrap();
        };
    }
}

fn print(scope: &mut HandleScope, args: FunctionCallbackArguments, mut rv: ReturnValue) {
    let result: serde_json::Value = serde_v8::from_v8(scope, args.get(0)).unwrap();
    println!("Rust say: {:#?}", result);
    rv.set(serde_v8::to_v8(scope, result).unwrap());
}

fn fetch(scope: &mut HandleScope, args: FunctionCallbackArguments, mut rv: ReturnValue) {
    let url: String = serde_v8::from_v8(scope, args.get(0)).unwrap();
    let result = reqwest::blocking::get(&url).unwrap().text().unwrap();
    rv.set(serde_v8::to_v8(scope, result).unwrap());
}
```

然后我们要在 js 侧注册这两个函数

```rust
'use strict';

({ print, fetch }) => {
    globalThis.print = (args) => {
        return print(args);
    };
    globalThis.fetch = async (args) => {
        return await fetch(args);
    };
};
```

最后我们在 v8init 的时候将其绑定到对应的 scope 中

```rust
mod state;
mod utils;
mod extensions;

use extensions::{Extensions};
use once_cell::sync::OnceCell;
use state::JsRuntimeState;
use utils::execute_script;
use v8::{
    CreateParams, FunctionCodeHandling, HandleScope, Isolate, OwnedIsolate, SnapshotCreator, V8,
};

pub type LocalValue<'a> = v8::Local<'a, v8::Value>;

pub struct JsRuntime {
    isolate: OwnedIsolate,
}

impl JsRuntime {
    pub fn init() { // 初始化v8
        static V8_INSTANCE: OnceCell<()> = OnceCell::new(); // 使用OnceCell保证多次init也只会执行一次
        V8_INSTANCE.get_or_init(|| {
            let platform = v8::new_default_platform(0, false).make_shared();
            V8::initialize_platform(platform);
            V8::initialize();
        });
    }

    // pub fn new(params: JsRuntimeParams) -> Self {
    pub fn new(snapshot: Option<Vec<u8>>) -> Self { // 新建runtime
        let mut params = CreateParams::default();
        let mut initialized = false;
        let isolate = Isolate::new(params);
        JsRuntime::init_isolate(isolate, initialized)
    }

    pub fn execute_script(
        &mut self,
        code: impl AsRef<str>,
        is_module: bool,
    ) -> Result<serde_json::Value, serde_json::Value> { // 执行js代码并返回结果
        let context = JsRuntimeState::get_context(&mut self.isolate);
        let handle_scope = &mut HandleScope::with_context(&mut self.isolate, context);
        match execute_script(handle_scope, code, is_module) {
            Ok(value) => Ok(serde_v8::from_v8(handle_scope, value).unwrap()),
            Err(error) => Err(serde_v8::from_v8(handle_scope, error).unwrap()),
        }
    }

    pub fn create_snapshot() -> Vec<u8> {
        todo!()
    }

    fn init_isolate(mut isolate: OwnedIsolate, initialized: bool) -> Self { // 初始化Isolate,相关
        let state = JsRuntimeState::new(&mut isolate);
        isolate.set_slot(state);
        let context = JsRuntimeState::get_context(&mut isolate);
        let scope = &mut HandleScope::with_context(&mut isolate, context);
        // =========> new
        Extensions::install(scope);

        JsRuntime { isolate }
    }
}
```

# 支持 snapshot

snapshot 是指 v8 根据 js 代码生成的二进制文件，它能够加速整体 js 代码的加载，v8 提供了这样的接口，我们可以来实现一下，调用的方式应该如下：

```rust
use std::fs;

use clap::{Parser, Subcommand};
use v8_live::JsRuntime;

const SS_FILE: &str = "snapshot.blob";

#[derive(Debug, Parser)]
#[clap(author, version, about)]
struct Args {
    #[clap(subcommand)]
    action: Action,
}

#[derive(Subcommand, Debug)]
enum Action {
    Build,
    Run,
}

fn main() {
    JsRuntime::init();
    let args = Args::parse();
    match args.action {
        Action::Build => build_snapshot(SS_FILE), // cargo run --example snapshot build
        Action::Run => run_snapshot(SS_FILE),// cargo run --example snapshot run
    }
}

fn build_snapshot(path: &str) {
    let blob = JsRuntime::create_snapshot();
    fs::write(path, blob).unwrap();
}

fn run_snapshot(path: &str) {
    let blob = fs::read(path).unwrap();
    let mut runtime = JsRuntime::new(Some(blob));
    let code = r#"
        function hello_world() {
            print("Hello Rust");
            // return {
            //     status: 200,
            //     message: "Hello World"
            // };
            return fetch("https://www.rust-lang.org/")
        }
        hello_world();
    "#;
    let result = runtime.execute_script(code, false);
    println!("Result is: {:?}", result);
}
```

接下来我们实现一下：

```rust
mod extensions;
mod state;
mod utils;

use extensions::{Extensions, EXTERNAL_REFERNCES};
use once_cell::sync::OnceCell;
use state::JsRuntimeState;
use utils::execute_script;
use v8::{
    CreateParams, FunctionCodeHandling, HandleScope, Isolate, OwnedIsolate, SnapshotCreator, V8,
};

pub type LocalValue<'a> = v8::Local<'a, v8::Value>;

pub struct JsRuntime {
    isolate: OwnedIsolate,
}

impl JsRuntime {
    pub fn init() {
        static V8_INSTANCE: OnceCell<()> = OnceCell::new();
        V8_INSTANCE.get_or_init(|| {
            let platform = v8::new_default_platform(0, false).make_shared();
            V8::initialize_platform(platform);
            V8::initialize();
        });
    }

    // pub fn new(params: JsRuntimeParams) -> Self {
    pub fn new(snapshot: Option<Vec<u8>>) -> Self {
        // =========> new
        let mut params = CreateParams::default().external_references(&**EXTERNAL_REFERNCES); // 注册相关函数
        let mut initialized = false;
        if let Some(snapshot) = snapshot {
            params = params.snapshot_blob(snapshot);
            initialized = true;
        }
        let isolate = Isolate::new(params);
        JsRuntime::init_isolate(isolate, initialized)
    }

    pub fn execute_script(
        &mut self,
        code: impl AsRef<str>,
        is_module: bool,
    ) -> Result<serde_json::Value, serde_json::Value> {
        let context = JsRuntimeState::get_context(&mut self.isolate);
        let handle_scope = &mut HandleScope::with_context(&mut self.isolate, context);
        match execute_script(handle_scope, code, is_module) {
            Ok(value) => Ok(serde_v8::from_v8(handle_scope, value).unwrap()),
            Err(error) => Err(serde_v8::from_v8(handle_scope, error).unwrap()),
        }
    }

    // =========> new
    pub fn create_snapshot() -> Vec<u8> {
        // 注册相关函数
        let mut sc = SnapshotCreator::new(Some(&EXTERNAL_REFERNCES));
        let isolate = unsafe { sc.get_owned_isolate() };
        let mut runtime = JsRuntime::init_isolate(isolate, false);
        {
            let context = JsRuntimeState::get_context(&mut runtime.isolate);
            let handle_scope = &mut HandleScope::new(&mut runtime.isolate);
            let context = v8::Local::new(handle_scope, context);
            sc.set_default_context(context);
        }

        JsRuntimeState::drop_context(&mut runtime.isolate);
        std::mem::forget(runtime); // 保证runtime在此生命周期内不被回收

        match sc.create_blob(FunctionCodeHandling::Keep) { // 创建blob
            Some(blob) => blob.to_vec(),
            None => panic!("Failed to create snapshot"),
        }
    }

    fn init_isolate(mut isolate: OwnedIsolate, initialized: bool) -> Self {
        let state = JsRuntimeState::new(&mut isolate);
        isolate.set_slot(state);
        // =========> new
        if !initialized { // 非blob情况下执行原来的逻辑
            let context = JsRuntimeState::get_context(&mut isolate);
            let scope = &mut HandleScope::with_context(&mut isolate, context);
            Extensions::install(scope);
        };
        JsRuntime { isolate }
    }
}

// extensions/mod.rs
use lazy_static::lazy_static;
use v8::{
    ExternalReference, ExternalReferences, FunctionCallbackArguments, HandleScope, MapFnTo,
    ReturnValue,
};

use crate::utils::execute_script;

const GLUE: &str = include_str!("glue.js");

// =========> new
lazy_static! {
    pub static ref EXTERNAL_REFERNCES: ExternalReferences = ExternalReferences::new(&[
        ExternalReference {
            function: MapFnTo::map_fn_to(print),
        },
        ExternalReference {
            function: MapFnTo::map_fn_to(fetch),
        }
    ]);
}

pub struct Extensions;

impl Extensions {
    pub fn install(scope: &mut HandleScope) {
        let bindings = v8::Object::new(scope);

        let name = v8::String::new(scope, "print").unwrap();
        let func = v8::Function::new(scope, print).unwrap();
        bindings.set(scope, name.into(), func.into()).unwrap();

        let name = v8::String::new(scope, "fetch").unwrap();
        let func = v8::Function::new(scope, fetch).unwrap();
        bindings.set(scope, name.into(), func.into()).unwrap();

        if let Ok(result) = execute_script(scope, GLUE) {
            let func = v8::Local::<v8::Function>::try_from(result).unwrap();
            let v = v8::undefined(scope).into();
            let args = [bindings.into()];
            func.call(scope, v, &args).unwrap();
        };
    }
}

fn print(scope: &mut HandleScope, args: FunctionCallbackArguments, mut rv: ReturnValue) {
    let result: serde_json::Value = serde_v8::from_v8(scope, args.get(0)).unwrap();
    println!("Rust say: {:#?}", result);
    rv.set(serde_v8::to_v8(scope, result).unwrap());
}

fn fetch(scope: &mut HandleScope, args: FunctionCallbackArguments, mut rv: ReturnValue) {
    let url: String = serde_v8::from_v8(scope, args.get(0)).unwrap();
    let result = reqwest::blocking::get(&url).unwrap().text().unwrap();
    rv.set(serde_v8::to_v8(scope, result).unwrap());
}
```

# 支持 module

如果我们的 js 代码是以 module 的形式引入 v8 的话，就可以在主代码中直接使用 async/await，比如：

```rust
use v8_live::JsRuntime;

fn main() {
    JsRuntime::init();
    let mut runtime = JsRuntime::new(None);

    let code = r#"
        async function hello_world() {
            print("Hello Rust");
            // return {
            //     status: 200,
            //     message: "Hello World"
            // };
            return await fetch("https://www.rust-lang.org/")
        }
        let result = await hello_world();
        print(result);
    "#;

    let result = runtime.execute_script(code, true);
    println!("Result is: {:#?}", result);
}
```

而我们在 rust 需要做一定修改来支持 module 的引入：

```rust
use crate::LocalValue;
use v8::{script_compiler, HandleScope, Promise, Script, ScriptOrigin, TryCatch};

pub fn execute_script<'a>(
    scope: &mut HandleScope<'a>,
    code: impl AsRef<str>,
    is_module: bool,
) -> Result<LocalValue<'a>, LocalValue<'a>> {
    let scope = &mut TryCatch::new(scope);
    let source = v8::String::new(scope, code.as_ref()).unwrap();
    let origin = create_origin(scope, "dummy.js", is_module);
    if is_module {
        let source = script_compiler::Source::new(source, Some(&origin));
        let module = script_compiler::compile_module(scope, source).unwrap();
        module.instantiate_module(scope, module_callback).unwrap();
        Ok(module.evaluate(scope).unwrap())
    } else {
        Script::compile(scope, source, Some(&origin))
            .and_then(|script| script.run(scope))
            .map_or_else(|| Err(scope.stack_trace().unwrap()), Ok)
    }
}

fn module_callback<'a>(
    context: v8::Local<'a, v8::Context>,
    name: v8::Local<'a, v8::String>,
    arr: v8::Local<'a, v8::FixedArray>,
    module: v8::Local<'a, v8::Module>,
) -> Option<v8::Local<'a, v8::Module>> {
    println!(
        "context: {:?}, name: {:?}, arr: {:?}, module: {:?}",
        context, name, arr, module
    );
    Some(module)
}

fn create_origin<'a>(
    scope: &mut HandleScope<'a>,
    filename: impl AsRef<str>,
    is_module: bool,
) -> ScriptOrigin<'a> {
    let name: LocalValue = v8::String::new(scope, filename.as_ref()).unwrap().into();
    ScriptOrigin::new(
        scope,
        name.clone(),
        0,
        0,
        false,
        0,
        name,
        false,
        false,
        is_module,
    )
}
```

# async/await

如同 js，在 rust 中我们可以使用 tokio 来支持 async/await，前面我们使用的是 reqwest::blocking 来实现的 fetch，现在我们可以用 tokio 来改造：

```rust
use std::{borrow::Borrow, future::Future};

use lazy_static::lazy_static;
use v8::{
    ExternalReference, ExternalReferences, FunctionCallbackArguments, HandleScope, MapFnTo,
    ReturnValue,
};

use crate::utils::execute_script;

const GLUE: &str = include_str!("glue.js");

lazy_static! {
    pub static ref EXTERNAL_REFERNCES: ExternalReferences = ExternalReferences::new(&[
        ExternalReference {
            function: MapFnTo::map_fn_to(print),
        },
        ExternalReference {
            function: MapFnTo::map_fn_to(fetch),
        }
    ]);
}

pub struct Extensions;

impl Extensions {
    pub fn install(scope: &mut HandleScope) {
        let bindings = v8::Object::new(scope);

        let name = v8::String::new(scope, "print").unwrap();
        let func = v8::Function::new(scope, print).unwrap();
        bindings.set(scope, name.into(), func.into()).unwrap();

        let name = v8::String::new(scope, "fetch").unwrap();
        let func = v8::Function::new(scope, fetch).unwrap();
        bindings.set(scope, name.into(), func.into()).unwrap();

        if let Ok(result) = execute_script(scope, GLUE, false) {
            let func = v8::Local::<v8::Function>::try_from(result).unwrap();
            let v = v8::undefined(scope).into();
            let args = [bindings.into()];
            func.call(scope, v, &args).unwrap();
        };
    }
}

fn print(scope: &mut HandleScope, args: FunctionCallbackArguments, mut rv: ReturnValue) {
    let result: serde_json::Value = serde_v8::from_v8(scope, args.get(0)).unwrap();
    println!("Rust say: {:#?}", result);
    rv.set(serde_v8::to_v8(scope, result).unwrap());
}

// =========> new
fn fetch(scope: &mut HandleScope, args: FunctionCallbackArguments, mut rv: ReturnValue) {
    let url: String = serde_v8::from_v8(scope, args.get(0)).unwrap();
    let fut = async move {
        let result = reqwest::get(&url).await.unwrap().text().await.unwrap();
        rv.set(serde_v8::to_v8(scope, result).unwrap());
    };

    run_local_future(fut);
}
// =========> new
fn run_local_future<R>(fut: impl Future<Output = R>) { // 并没有异步，实际上仍是阻塞
    let rt = tokio::runtime::Builder::new_current_thread()
        .enable_all()
        .build()
        .unwrap();
    let local = tokio::task::LocalSet::new();
    local.block_on(&rt, fut);
}

// utils.rs
use crate::LocalValue;
use v8::{script_compiler, HandleScope, Promise, Script, ScriptOrigin, TryCatch};

pub fn execute_script<'a>(
    scope: &mut HandleScope<'a>,
    code: impl AsRef<str>,
    is_module: bool,
) -> Result<LocalValue<'a>, LocalValue<'a>> {
    let scope = &mut TryCatch::new(scope);
    let source = v8::String::new(scope, code.as_ref()).unwrap();
    let origin = create_origin(scope, "dummy.js", is_module);
    if is_module {
        let source = script_compiler::Source::new(source, Some(&origin));
        let module = script_compiler::compile_module(scope, source).unwrap();
        module.instantiate_module(scope, module_callback).unwrap();
        // =========> new
        let result = module.evaluate(scope).unwrap();
        let promise = v8::Local::<v8::Promise>::try_from(result).unwrap();
        match promise.state() {
            v8::PromiseState::Pending => panic!("We don't know hot to process pending promise"),
            v8::PromiseState::Fulfilled => Ok(promise.result(scope)),
            v8::PromiseState::Rejected => Err(promise.result(scope)),
        }
        // ==============
    } else {
        Script::compile(scope, source, Some(&origin))
            .and_then(|script| script.run(scope))
            .map_or_else(|| Err(scope.stack_trace().unwrap()), Ok)
    }
}

fn module_callback<'a>(
    context: v8::Local<'a, v8::Context>,
    name: v8::Local<'a, v8::String>,
    arr: v8::Local<'a, v8::FixedArray>,
    module: v8::Local<'a, v8::Module>,
) -> Option<v8::Local<'a, v8::Module>> {
    println!(
        "context: {:?}, name: {:?}, arr: {:?}, module: {:?}",
        context, name, arr, module
    );
    Some(module)
}

fn create_origin<'a>(
    scope: &mut HandleScope<'a>,
    filename: impl AsRef<str>,
    is_module: bool,
) -> ScriptOrigin<'a> {
    let name: LocalValue = v8::String::new(scope, filename.as_ref()).unwrap().into();
    ScriptOrigin::new(
        scope,
        name.clone(),
        0,
        0,
        false,
        0,
        name,
        false,
        false,
        is_module,
    )
}
```

需要注意的是，在这里使用的 tokio 实际上仍然是 block 状态，并没有做到异步的效果，由于使用 tokio 执行异步的时候，需要将 HandleScope 的所有权 move 到线程池中，导致后续的编码非常困难，如果感兴趣的的话可以去看 deno 的 core 源码，在其中有很详细的实现，这里我就不过多实现了。
最后附上如果使用 deno_core 的源码：
Basic:

```rust
use deno_core::{
    anyhow::Result, serde::de::DeserializeOwned, serde_v8, v8, JsRuntime, RuntimeOptions,
};

#[tokio::main]
async fn main() -> Result<()> {
    let options = RuntimeOptions::default();
    let mut rt = JsRuntime::new(options);
    let code = include_str!("basic.js");
    let result: String = eval(&mut rt, code).await?;
    print!("Rust: {:?}", result);
    Ok(())
}

async fn eval<T>(rt: &mut JsRuntime, code: &str) -> Result<T>
where
    T: DeserializeOwned,
{
    let ret = rt.execute_script("hello_worlld", code)?;
    let result = rt.resolve_value(ret).await?;
    let scope = &mut rt.handle_scope();
    let result = v8::Local::new(scope, result);
    Ok(serde_v8::from_v8(scope, result).unwrap())
}
```

Module:

```rust
use std::rc::Rc;

use deno_core::{
    anyhow::Result, resolve_url_or_path, serde::de::DeserializeOwned, serde_v8, v8, FsModuleLoader,
    JsRuntime, RuntimeOptions,
};

#[tokio::main]
async fn main() -> Result<()> {
    let options = RuntimeOptions {
        module_loader: Some(Rc::new(FsModuleLoader)),
        ..Default::default()
    };
    let mut rt = JsRuntime::new(options);
    let path = format!("{}/examples/module.js", env!("CARGO_MANIFEST_DIR"));
    execute_main_module(&mut rt, &path).await?;
    Ok(())
}

async fn execute_main_module(rt: &mut JsRuntime, path: impl AsRef<str>) -> Result<()> {
    let url = resolve_url_or_path(path.as_ref())?;
    let id = rt.load_main_module(&url, None).await?;
    let mut receiver = rt.mod_evaluate(id);
    tokio::select! { // 这里是真实的异步
        resolved = &mut receiver => {
            resolved.expect("failed to evaluate module")
        },
        _ = rt.run_event_loop(false) => {
            receiver.await.expect("failed to evaluate module")
        }
    }
}
```

Snapshot:

```rust
// build
use deno_core::{JsRuntime, RuntimeOptions};
use std::{fs, path::Path};
use zstd::encode_all;

const SNAPSHOT_FILE: &str = "snapshots/main.bin";

fn main() {
    let options = RuntimeOptions {
        will_snapshot: true,
        ..Default::default()
    };
    let mut rt = JsRuntime::new(options);
    let data = rt.snapshot();
    let filename = Path::new(env!("CARGO_MANIFEST_DIR")).join(SNAPSHOT_FILE);
    let compressed = encode_all(&*data, 7).unwrap();
    fs::write(filename, compressed).unwrap();
}

// load
use lazy_static::lazy_static;
use std::rc::Rc;

use deno_core::{
    anyhow::Result, resolve_url_or_path, serde::de::DeserializeOwned, serde_v8, v8, FsModuleLoader,
    JsRuntime, RuntimeOptions, Snapshot,
};

lazy_static! {
    static ref SNAPSHOT: &'static [u8] = {
        let data = include_bytes!("../snapshots/main.bin");
        let decompressed = zstd::decode_all(&data[..]).unwrap().into_boxed_slice();
        Box::leak(decompressed)
    };
}

#[tokio::main]
async fn main() -> Result<()> {
    let options = RuntimeOptions {
        module_loader: Some(Rc::new(FsModuleLoader)),
        startup_snapshot: Some(Snapshot::Static(&*SNAPSHOT)),
        ..Default::default()
    };
    let mut rt = JsRuntime::new(options);
    let path = format!("{}/examples/module.js", env!("CARGO_MANIFEST_DIR"));
    execute_main_module(&mut rt, &path).await?;
    Ok(())
}

async fn execute_main_module(rt: &mut JsRuntime, path: impl AsRef<str>) -> Result<()> {
    let url = resolve_url_or_path(path.as_ref())?;
    let id = rt.load_main_module(&url, None).await?;
    let mut receiver = rt.mod_evaluate(id);
    tokio::select! {
        resolved = &mut receiver => {
            resolved.expect("failed to evaluate module")
        },
        _ = rt.run_event_loop(false) => {
            receiver.await.expect("failed to evaluate module")
        }
    }
}
```

# 最后

这个项目只是稍微带你熟悉一下 v8 与 rust 之间的交互，如果想要感受更加细节的部分，可以去看 deno 的源码，里面有很多的细节，非常值得学习。

在实际的 fe 使用中，我们需要用 nvm 切换 node 版本，fvm 切换 fe 的版本，如果我们将 fe 的不同代码生成 blob，将 deno 的 runtime 使用 rust 封装到二进制文件中，我们是不是就可以在 fe d 的时候，让 rust 读取 fe.config.js 文件，根据 config 信息，将对应的 fe 版本的 blob 由内置的 deno runtime 去运行，这样我们是不是每次在运行项目时，就不必再需要考虑 nvm 和 fvm 的版本问题了呢，或许这是一种解决方案。
