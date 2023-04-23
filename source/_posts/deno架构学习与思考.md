---
title: deno架构学习与思考
date: 2022-08-29 22:29:30
tags:
  - deno
  - rust
categories: 学习笔记
---

# 前置知识

1. v8: V8 是一个由 Google 开发的开源 JavaScript 引擎，用于 Google Chrome 及 Chromium 中。
2. v8 isolate: 运行 js 代码的安全沙箱（受控，受限，chrome 中每一个 tab 为一个 isolate）

![v8](/image/deno-architecture/v8.jpg)

3. v8 bindings: v8 内部提供的允许 js 调用原生代码的能力 4. Future: rust 中的异步方案，类似于 js 的 promise 5. Tokio: rust 中的异步 runtime 的实现

# Nodejs 架构

![node-architecture](/image/deno-architecture/node-architecture.png)

1. v8 执行 js 代码
2. v8 bindings 代用 napi
3. napi 将事件放置于 libuv 的 event queue
4. 在 libuv 中执行事件
5. 如果发生阻塞（如网络请求，磁盘 io），将其交于 thread pool 处理

# Nodejs 的问题

nodejs 为 js 提供了原本 js 不支持的功能，但是当时未考虑到带来的相关的安全性，直接把 v8 提供的隔离打破了。
但是在云服务的时代里，我们往往需要将 nodejs 进行一次打包成 container，但是 v8 的 isolate 本身就是一个很好的隔离容器，这样往往显得多此一举，在 serverless 这种更加强调隔离（每个请求都进行隔离）的条件下，会将 node 的隔离问题暴露的更加明显。
另外，node 在扩展 napi 的时候，设计上也有一定的问题，跟 v8 耦合性太高，在开发时，需要了解 v8 的 isolate，context，handle 等相关的概念，才能理解如何处理。
https://juejin.cn/post/6961201207964598286

# deno 的架构

## 总览

![deno-architecture](/image/deno-architecture/deno-architecture.png)
首先，deno 将用户空间和内核空间进行了一个隔离，在执行内核空间的事情的时候，会通过 permissions 进行权限判断，保证系统的安全性。
其次，deno 还提供了 serde_v8，保障了基本的 v8 类型在与 rust 之间进行了简化操作，在大 data 层面，提供了 binaryArray 实现了 zerocopy 的能力。
剩下的架构设计部分，其实是与 nodejs 大同小异的。

## ops

Rid -> fd
Ops -> syscall

![syscall](/image/deno-architecture/syscall.png)
![fd](/image/deno-architecture/fd.png)
![ops](/image/deno-architecture/ops.png)
node:

```cpp
#include <node.h>

namespace demo {

    using v8::FunctionCallbackInfo;
    using v8::Isolate;
    using v8:Local;
    using v8:Object;
    using v8:String;
    using v8:Value;

    void Method(const FunctionCallbackInfo<Value> &args) {
        Isolate *isolate = args.GetIsolate();
        args.GetReturnValue().Set(String::NewFromUtf8(isolate, "world").ToLocalChecked());
    }

    void Initialize(Local<Object> exports) {
        NODE_SET_METHOD(exports, "hello", Method);
    }

    NODE_MODULE(NODE_GYP_MODULE_NAME, Initalize)

}
```

// 还要写一些脚手架代码
deno:

```rust
// rust
#[op]
fn op_hello(name: String) -> String {
    format!("hello {:?}!", name);
}

// js
((window) => {
    function hello(name) {
        return Deno.core.opSync("op_hello", name);
    }
    window.hello = hello
})(this)
```

## permissions

```rust
pub struct Permissions {
    pub read: UnaryPermission<ReadDesciptor>,
    pub wirte: UnaryPermission<WriteDesciptor>,
    pub net: UnaryPermission<NetDesciptor>,
    pub env: UnaryPermission<EnvDesciptor>,
    pub run: UnaryPermission<RunDesciptor>,
    pub ffi: UnaryPermission<FfiDesciptor>,
    pub hrtime: UnitPermission,
}

pub struct UnaryPermission<T: Eq + Hash> {
    pub name: &'static str,
    pub description: &'static str,
    pub global_state: PermissionState,
    pub granted_list: HashSet<T>,
    pub denied_list: HashSet<T>,
    pub prompt: bool,
}
```

Permissions 保证 js 在 server 的代码安全，在 node 中，你必须要保证你的代码你要完全信任，但是这是非常困难的，你不能保证你的库不会去操作你的系统文件，扫码本地磁盘，盗取私钥等等，deno 却能做到，比如访问仅仅设置的几个域名，通过 RBAC（oso）根据用户信息，设置权限。
![evil](/image/deno-architecture/evil.png)

## 例子

![ex](/image/deno-architecture/ex.png)

# 思考

## 架构元素拆分设计

1. 设计时，相关架构元素概念的定义
1. 很多时候往往 bug 是由于刚开始元素的概念定义有误从而引发的
1. 好的元素概念：pipe, steam, flow, socket
1. 思考架构的核心目标概念
1. 如何定义的你的架构目标，组织相关的元素，成为你的架构基石
   🌰：
   一个检索系统，检索 top10 文本及使用频率
   单词解析(split) -> 映射(map) -> 聚合 -> reduce -> sort -> filter

## 架构设计层次性

1. 整体产品 MVC MVP MVVM
2. 后端逻辑
   ![layer](/image/deno-architecture/layer.png)
3. 层级的思考
   1. 为什么需要一个新的层级，目的？（Permissions）
   2. 如何定义这个层级？（明确概念）
   3. 我们需要多少层级？（过犹不及，多层相关联时会增加复杂度）
      ![indirectoin](/image/deno-architecture/indirectoin.png)

## 接口的设计

好的接口的设计会提升你的系统的稳定性，安全性
![interface](/image/deno-architecture/interface.png)

## 延迟决策

获取需求时需要考虑潜在需求，考虑架构时需要考虑进来，抽象你的代码，让其能应对后续需求的变迁
🌰：
1.0 TCP server -> sync call
2.0 TCP+ TLS -> async call
3.0 TCP + TLS + thrift -> steam
4.0 QUIC -> ?

## 可复用的架构资源

![deno-packages](/image/deno-architecture/deno_packages.png)
rusty_v8: 可用于处理 js，作为 js 解释器
deno_core：拥有 rust 异步时间循环的低级 js runtime，可高效快捷的扩展 js 能力
deno_runtime: 可自定义的 deno runtime

## deno 可能的未来

serverless framework
![isolates_vs_wasm](/image/deno-architecture/isolates_vs_wasm.png)
![deno_runtime](/image/deno-architecture/deno_runtime.jpg)
![serverless](/image/deno-architecture/serverless.jpg)
Cross platform framwork
![cross_platform](/image/deno-architecture/cross_platform.png)
继承 node
deno 官方接下来 3 个月内将会让 deno 兼容 npm 生态，以及努力使 deno 成为最快的 js runtime
https://deno.com/blog/changes
