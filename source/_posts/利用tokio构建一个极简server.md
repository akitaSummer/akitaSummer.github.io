---
title: 利用tokio构建一个极简server
date: 2022-03-26 09:22:33
tags:
  - tcp
  - rust
  - http
categories: 学习笔记
---

> Measure first. Optimize. Measure again.

# 前言

上一次我们使用 tokio 实现了一个简易 redis，其中我们是使用的 tcp 流，说道 tcp 流，就不得不说我们最常用的 http 了，这次会实现一个极简 http_server，它包含了：

- 并发数限制
- 路由解析
- 解析 Request
- 生成 Response
- 读取预先配置

设计图：

![architecture](/image/mini-server/architecture.jpg)

# Config

Http 中有很多配置，比如 status 和 mime_type，如果写在源码里，我们每次都需要更新源码，这样会非常麻烦，但是我们可以放置于 json 中，在运行时动态去读取

```rust
use once_cell::sync::OnceCell;
use serde_json::Result;
use std::collections::HashMap;
use std::io::Result as IOResult;
use std::{env, fs};
use tracing::{debug, error, info};

#[derive(Clone, Debug, PartialEq)]
pub struct Status(pub String, pub String);
pub struct MimeType(pub String, pub String);

// 全局共享单个数据
pub static GLOBAL_STATUSES: OnceCell<HashMap<String, Status>> = OnceCell::new();
pub static GLOBAL_MIME_CFG: OnceCell<HashMap<String, String>> = OnceCell::new();
pub static OCTECT_STREAM: &'static str = "application/octet-stream";

// 读取文件
fn read_file_to_string_rel_to_runtime_dir(file_path: &str) -> IOResult<String> {
    let mut runtime_dir = env::current_dir().unwrap();
    runtime_dir.push(file_path);
    return fs::read_to_string(runtime_dir.to_str().unwrap());
}

// status,使用serde_json解析json后，将其赋值与全局变量上
fn init_default_http_status() {
    let mut status_config = HashMap::<String, Status>::new();
    if let Ok(status_cfg_json) = read_file_to_string_rel_to_runtime_dir("src/config/status.json") {
        let map: Result<HashMap<String, String>> = serde_json::from_str(status_cfg_json.as_str());
        match map {
            Ok(map) => {
                map.iter().for_each(|(k, v)| {
                    status_config.insert(k.clone(), Status(k.clone(), v.clone()));
                });
            }
            Err(_) => {
                error!("failed to load status.json");
            }
        }
    }
    GLOBAL_STATUSES.set(status_config).unwrap();
}

// mime_type,使用serde_json解析json后，将其赋值与全局变量上
fn init_default_mime_type() {
    let mut entries = HashMap::<String, String>::new();
    let content_wrapper = read_file_to_string_rel_to_runtime_dir("src/config/mime.json");
    if let Ok(mime_cfg_json) = content_wrapper {
        let json: HashMap<String, String> = serde_json::from_str(mime_cfg_json.as_str()).unwrap();
        for (k, v) in json.iter() {
            v.split_whitespace()
                .filter(|&tp| !tp.is_empty())
                .for_each(|tp| {
                    entries.insert(tp.to_string(), k.to_string());
                })
        }
    }
    info!("begin loading mime config");
    GLOBAL_MIME_CFG.set(entries).unwrap();
}

pub fn init() {
    init_default_mime_type();
    init_default_http_status();
}
```

share 主要是对 store 实现 mutex 锁及过期更新功能

```rust
use bytes::Bytes;
use std::collections::{BTreeMap, HashMap};
use std::sync::{Arc, Mutex};
use tokio::sync::Notify;
use tokio::time::{self, Duration, Instant};

#[derive(Debug)]
struct Shared {
    state: Mutex<State>,
    // 检查过期值或关闭信号
    background_task: Notify,
}

impl Shared {
    // 清除过期键
    fn purge_expired_keys(&self) -> Option<Instant> {
        // 获取state原始值
        let mut state = &mut *(self.state.lock().unwrap());
        if state.shutdown {
            return None;
        }

        // 当前时间
        let now = Instant::now();

        while let Some(((expired_time, id), key)) = state.expirations.iter().next() {
            if *expired_time > now {
                return Some(*expired_time);
            }

            state.entries.remove(key);
            state.expirations.remove(&(*expired_time, *id));
        }

        None
    }

    fn is_shutdown(&self) -> bool {
        self.state.lock().unwrap().shutdown
    }
}
```

# Route

route 其实可以细化为个部分：

- 路由分配
- 解析 request
- 执行业务逻辑
- 返回 response

![route](/image/mini-server/route.jpg)

## Request

request 主要作用是将 tcp 流进行解析转化为我们 rust 的数据结构，以便能被业务逻辑所运行
http 报文结构如下

![request](/image/mini-server/request.png)
第一行是请求方法，然后 header 会连续的以行出现，header 和 body 中有一空行作为分割识别
其实我们做的就是将 tcp 中的数据取出，然后逐行去解析，具体实现如下：

```rust
use std::collections::HashMap;
use tokio::io::{AsyncBufReadExt, AsyncReadExt, BufReader};
use tokio::net::TcpStream;

#[derive(Debug, PartialEq, Clone)]
pub enum Method {
    GET,
    POST,
    PUT,
    DELETE,
    UNKNOWN,
}

// http请求方法
impl From<&str> for Method {
    fn from(method: &str) -> Self {
        let lowercase_method = method.to_lowercase();
        let res = lowercase_method.as_str();
        match res {
            "get" => Method::GET,
            "post" => Method::POST,
            "put" => Method::PUT,
            "delete" => Method::DELETE,
            _ => Method::UNKNOWN,
        }
    }
}

// http版本
#[derive(Debug, PartialEq, Clone)]
pub enum Version {
    HTTP1_1,
    HTTP2_0,
    UNKNOWN,
}

#[derive(Debug, PartialEq, Clone)]
pub enum Resource {
    Path(String),
}

impl From<&str> for Version {
    fn from(ver: &str) -> Self {
        let ver_in_lowercase = ver.to_lowercase();
        match ver_in_lowercase.as_str() {
            "http/1.1" => Version::HTTP1_1,
            "http/2.0" => Version::HTTP2_0,
            _ => Version::UNKNOWN,
        }
    }
}

// request 结构
#[derive(Debug, PartialEq)]
pub struct HttpRequest {
    pub method: Method,
    pub version: Version,
    pub resource: Resource,
    pub headers: Option<HashMap<String, String>>,
    pub msg_body: Option<String>,
}

impl Default for HttpRequest {
    fn default() -> Self {
        Self {
            method: Method::GET,
            version: Version::HTTP1_1,
            resource: Resource::Path(String::from("/")),
            headers: None,
            msg_body: None,
        }
    }
}

impl HttpRequest {
    pub async fn from(stream: &mut TcpStream) -> Self {
        let mut reader = BufReader::<&mut TcpStream>::new(stream);
        let mut request = HttpRequest::default();
        let mut headers = HashMap::<String, String>::new();
        let mut content_len = 0;
        let mut is_req_line = true;
        // 读取header
        loop {
            let mut line = String::from("");
            reader.read_line(&mut line).await.unwrap();
            // 第一行
            if is_req_line {
                if line.is_empty() && is_req_line {
                    return HttpRequest::default();
                }
                // 解析请求信息
                let (method, resource, version) = process_req_line(line.as_str());
                request.method = method;
                request.resource = resource;
                request.version = version;
                is_req_line = false;
            } else if line.contains(":") {
                // 解析header
                let (key, value) = process_request_header(line.as_str());
                headers.insert(key.clone(), value.clone().trim().to_string());
                if key == "Content-Length" {
                    content_len = value.trim().parse::<usize>().unwrap();
                }
            } else if line == String::from("\r\n") {
                // header 与 body之间存在空行，header结束
                break;
            }
        }
        request.headers = Some(headers);
        if content_len > 0 {
            let mut buf = vec![0 as u8; content_len];
            let buf_slice = buf.as_mut_slice();
            // 读取请求体，注意，这里不能在使用stream进行读取，否则会一直卡在这里，要继续用reader进行读取.
            // BufReader::read(&mut reader, buf_slice).await.unwrap();
            reader.read(buf_slice).await.unwrap();
            request.msg_body = Some(String::from_utf8_lossy(buf_slice).to_string());
        }
        request
    }
}

impl From<String> for HttpRequest {
    fn from(diagram: String) -> Self {
        let mut request = HttpRequest::default();
        let mut headers = HashMap::<String, String>::new();
        for line in diagram.lines() {
            if line.contains("HTTP") {
                println!("request line is: {}", line);
                let (method, resource, version) = process_req_line(line);
                request.method = method;
                request.resource = resource;
                request.version = version;
            } else if line.contains(":") {
                let (key, value) = process_request_header(line);
                headers.insert(key, value);
            } else if line.is_empty() {
            } else {
                request.msg_body = Some(line.to_string());
            }
        }
        request.headers = Some(headers);
        request
    }
}

// 解析header
fn process_request_header(line: &str) -> (String, String) {
    let mut seg_iter = line.split(":");
    (
        seg_iter.next().unwrap().into(),
        seg_iter.next().unwrap().into(),
    )
}

// 解析首行
fn process_req_line(line: &str) -> (Method, Resource, Version) {
    let mut segments = line.split_whitespace();
    (
        segments.next().unwrap().into(),
        Resource::Path(segments.next().unwrap().to_string()),
        segments.next().unwrap().into(),
    )
}
```

测试：

```rust
#[test]
fn test_method_match() {
    let m: Method = "GET".into();
    assert_eq!(Method::GET, m);
    let m: Method = "posT".into();
    assert_eq!(Method::POST, m);
}

#[test]
fn test_version_match() {
    let v: Version = "HTTP/1.1".into();
    assert_eq!(v, Version::HTTP1_1);

    let v: Version = "Http/2.0".into();
    assert_eq!(v, Version::HTTP2_0);

    let v: Version = "HTTP/1.3".into();
    assert_eq!(v, Version::UNKNOWN);
}

#[test]
fn test_request_parse() {
    let req = "GET /index.js HTTP/1.1\r\nHost: localhost\r\nContent-Type: text/html\r\n\r\nxxxx";
    let actual_request: HttpRequest = String::from(req).into();
    let expected_request = HttpRequest {
        method: Method::GET,
        resource: Resource::Path("/index.js".to_string()),
        version: Version::HTTP1_1,
        headers: {
            let mut h = HashMap::<String, String>::new();
            h.insert("Host".to_string(), " localhost".to_string());
            h.insert("Content-Type".to_string(), " text/html".to_string());
            Some(h)
        },
        msg_body: Some(String::from("xxxx")),
    };
    assert_eq!(expected_request, actual_request);
}
```

## Response

response 主要做的就是将业务逻辑处理好的数据生成对应的 response 并写入 tcp 流中：

```rust
use std::collections::HashMap;
// 获取设置
use crate::config::{Status, GLOBAL_MIME_CFG};

// response数据结构
#[derive(Debug, PartialEq, Clone)]
pub struct HttpResponse {
    pub version: String,
    pub status_code: String,
    pub status_text: String,
    pub headers: Option<HashMap<String, String>>,
    pub resp_body: Option<String>,
}

impl Default for HttpResponse {
    fn default() -> Self {
        Self {
            version: String::from("HTTP/1.1"),
            status_code: String::from("200"),
            status_text: "OK".to_string(),
            headers: Some(HashMap::new()),
            resp_body: None,
        }
    }
}

impl HttpResponse {
    pub fn set_status(&mut self, status: Status) {
        let Status(status_code, status_text) = status;
        self.status_code = status_code;
        self.status_text = status_text;
    }

    pub fn set_headers(&mut self, headers: HashMap<String, String>) {
        self.headers = Some(headers);
    }

    pub fn set_body(&mut self, body: String) {
        self.resp_body = Some(body);
    }

    pub fn add_header(&mut self, key: String, value: String) {
        let headers = self.headers.as_mut().unwrap();
        headers.insert(key, value);
    }

    pub fn new(
        version: String,
        status_code: String,
        headers: Option<HashMap<String, String>>,
        resp_body: Option<String>,
    ) -> Self {
        let mut response = HttpResponse::default();
        response.version = version.to_string();
        if status_code != "200" {
            response.status_code = status_code.to_string();
            let code = status_code;
            let mime_config = &GLOBAL_MIME_CFG.get();
            let mut status_text = String::from("");
            if let Some(config) = mime_config {
                let desc = match config.get(&*code.to_string()) {
                    Some(status_desc) => status_desc,
                    None => "Unknown Status",
                };
                status_text.push_str(desc);
            };
            response.status_text = status_text;
        }
        if let Some(_) = headers {
            response.headers = headers;
        }

        if let Some(_) = resp_body {
            response.resp_body = resp_body;
        }
        response
    }

    fn get_serialized_headers(&self) -> String {
        let mut result = String::from("");
        match &self.headers {
            Some(headers) => {
                let mut keys = headers.keys().collect::<Vec<&String>>();
                keys.sort();
                keys.iter()
                    .for_each(|&k| result = format!("{}{}: {}\r\n", result, k, headers[k]));
                let content_len = if let Some(body) = &self.resp_body {
                    body.len()
                } else {
                    0
                };
                result = format!("{}{}: {}\r\n", result, "Content-Length", content_len);
            }
            None => {}
        }
        result
    }

    fn get_serialized_body(&self) -> String {
        match &self.resp_body {
            Some(body) => body.to_string(),
            None => String::from(""),
        }
    }
}

impl Into<String> for HttpResponse {
    fn into(self) -> String {
        format!(
            "{} {} {}\r\n{}\r\n{}",
            &self.version,
            &self.status_code,
            &self.status_text,
            &self.get_serialized_headers(),
            &self.get_serialized_body(),
        )
    }
}
```

测试：

```rust
#[test]
fn test_response_tostring() {
    let expected_resp_string = "HTTP/1.1 200 OK\r\n\
    Content-Type: text/html\r\n\
    Cookie: name=akita\r\n\
    Host: localhost:8080\r\n\
    Content-Length: 4\r\n\
    \r\n\
    yyyy";
    let mut headers = HashMap::<String, String>::new();
    headers.insert("Content-Type".into(), "text/html".into());
    headers.insert("Cookie".into(), "name=akita".into());
    headers.insert("Host".into(), "localhost:8080".into());

    let resp_body = String::from("yyyy");

    let response = HttpResponse::new(
        "HTTP/1.1".to_string(),
        "200".to_string(),
        Some(headers),
        Some(resp_body),
    );
    let actual_resp_string: String = response.into();
    assert_eq!(expected_resp_string, actual_resp_string);
}
```

## Route

有了 request 和 response 后，就可以编写我们的 route 了

```rust
pub mod handler;

use crate::http::{
    request::{HttpRequest, Method, Resource},
    response::HttpResponse,
};
// 业务逻辑handler
use crate::router::handler::{ApiHandler, HttpHandler, StaticResHandler};
// 静态文件夹
const STATIC_RES: &str = "/staticres";

pub struct Router {}

impl Router {
    pub async fn route(request: &HttpRequest) -> HttpResponse {
        let mut resp = HttpResponse::default(); // 生成response
        match request.method { // 对request进行处理
            Method::GET => {
                let Resource::Path(path) = &request.resource;
                if path.starts_with(STATIC_RES) {
                    let handler_wrapper = get_request_handler(request); // 获取业务逻辑的handler
                    if let Some(handler) = handler_wrapper {
                        resp = handler.handle(request); // 从handler中得到response
                        return resp;
                    }
                }
                resp.set_body("this is api response.".into());
            }
            _ => {
                resp.set_body("hello world".into());
            }
        }
        resp
    }
}

const STATIC_RES_HANDLER: StaticResHandler = StaticResHandler {};
const API_RES_HANDLER: ApiHandler = ApiHandler {};

// 获取handler
fn get_request_handler(req: &HttpRequest) -> Option<Box<&dyn HttpHandler>> {
    match req.method {
        Method::GET => Some(Box::new(&STATIC_RES_HANDLER)),
        _ => Some(Box::new(&API_RES_HANDLER)),
    }
}
```

业务逻辑：

```rust
use std::{env, fs};

use crate::config::{MimeType, GLOBAL_MIME_CFG, GLOBAL_STATUSES, OCTECT_STREAM};
use crate::http::{
    request::{HttpRequest, Resource},
    response::HttpResponse,
};
use crate::router::STATIC_RES;

pub trait HttpHandler {
    fn handle(&self, request: &HttpRequest) -> HttpResponse;
}

#[derive(Default)]
pub struct StaticResHandler {}

// 读个文件
impl HttpHandler for StaticResHandler {
    fn handle(&self, request: &HttpRequest) -> HttpResponse {
        let mut resp = HttpResponse::default();
        let Resource::Path(ref path) = request.resource;
        let real_path = &path[STATIC_RES.len()..];
        let mut runtime_dir = env::current_dir().unwrap();
        runtime_dir.push("public");
        real_path
            .split("/")
            .into_iter()
            .for_each(|s| runtime_dir.push(s));
        let res_content = fs::read_to_string(runtime_dir.to_str().unwrap());
        match res_content {
            Ok(content) => {
                resp.resp_body = Some(content);
            }
            _ => {
                let mut path_buf = env::current_dir().unwrap();
                path_buf.push("public/404.html");
                let not_found_page_path = path_buf.to_str().unwrap();
                resp.resp_body = Some(fs::read_to_string(not_found_page_path).unwrap());
                resp.add_header("Content-Type".into(), "text/html".into());
                let statuses = GLOBAL_STATUSES.get().unwrap();
                let status = statuses.get("404").unwrap();
                resp.set_status(status.clone());
                return resp;
            }
        }

        let content_type = match path.split("/").last() {
            Some(res_name) => match res_name.split(".").last() {
                Some(ext) => GLOBAL_MIME_CFG.get().map(|entries| {
                    if let Some(tp) = entries.get(ext) {
                        tp.clone()
                    } else {
                        OCTECT_STREAM.into()
                    }
                }),
                None => Some(OCTECT_STREAM.into()),
            }
            .unwrap(),
            _ => OCTECT_STREAM.into(),
        };

        resp.add_header("Content-Type".into(), content_type);

        resp
    }
}

#[derive(Default)]
pub struct ApiHandler {}

impl HttpHandler for ApiHandler {
    fn handle(&self, request: &HttpRequest) -> HttpResponse {
        todo!("to implement")
    }
}
```

# Handler

handler 的作用是用来处理请求，他在多线程下是处理请求的最小单位：

```rust
pub mod request;
pub mod response;
pub mod shutdown;

use std::future::Future;
use std::sync::Arc;
use tokio::io::AsyncWriteExt;
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::{broadcast, mpsc, Semaphore};
use tokio::time::{self, Duration};
use tracing::{debug, error, info};

use crate::config::init;
use crate::http::request::HttpRequest;
use crate::http::shutdown::Shutdown;
use crate::router::Router;
use crate::types::Result;

struct Handler {
    socket: TcpStream,
    limit_connections: Arc<Semaphore>,
    shutdown: Shutdown, // 监听ctrl+c进行退出，下面实现
    _shutdown_complete: mpsc::Sender<()>,
}

impl Handler {
    async fn run(&mut self) -> Result<()> {
        while !self.shutdown.is_shutdown() {
            let request = HttpRequest::from(&mut self.socket).await; // 从tcp流中获取request
            println!("request is: {:?}", request.resource);
            let resp = Router::route(&request).await; // 进行处理
            let resp_str: String = resp.into();
            self.socket
                .write(resp_str.as_bytes() as &[u8])
                .await
                .unwrap(); // 写入tcp流返回
            self.socket.flush().await.unwrap();
        }

        Ok(())
    }
}

impl Drop for Handler {
    fn drop(&mut self) {
        self.limit_connections.add_permits(1);
    }
}
```

# 优雅退出

上文中的 shutdown 是一个常见的批量杀死进程的方法，之前的文章中也有写到，这里就不细说了：

```rust
use tokio::sync::broadcast;

#[derive(Debug)]
pub struct Shutdown {
    shutdown: bool,
    // 监听关闭信息
    notify: broadcast::Receiver<()>,
}

impl Shutdown {
    pub fn new(notify: broadcast::Receiver<()>) -> Shutdown {
        Shutdown {
            shutdown: false,
            notify,
        }
    }

    pub fn is_shutdown(&self) -> bool {
        self.shutdown
    }

    pub async fn recv(&mut self) {
        if self.shutdown {
            return;
        }

        // 接受关闭信号
        let _ = self.notify.recv().await;

        self.shutdown = true;
    }
}
```

# HttpServer

最后，创建一个 tcplistener 作为 HttpServer，它将负责管理 handler：

```rust
pub mod request;
pub mod response;
pub mod shutdown;

use std::future::Future;
use std::sync::Arc;
use tokio::io::AsyncWriteExt;
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::{broadcast, mpsc, Semaphore};
use tokio::time::{self, Duration};
use tracing::{debug, error, info};

use crate::config::init;
use crate::http::request::HttpRequest;
use crate::http::shutdown::Shutdown;
use crate::router::Router;
use crate::types::Result;

pub struct HttpServer {
    listener: TcpListener,
    limit_connections: Arc<Semaphore>, // 限制并发数
    notify_shutdown: broadcast::Sender<()>, // 广播终止信息
    shutdown_complete_rx: mpsc::Receiver<()>,
    shutdown_complete_tx: mpsc::Sender<()>,
}

impl HttpServer {
    pub async fn run(&self) -> Result<()> {
        loop {
        // 达到线程限制数目会在此阻塞
            self.limit_connections.acquire().await.unwrap().forget();
            // 获取tcp流
            let socket = self.accept().await?;

            let mut handler = Handler {
                socket: socket,
                limit_connections: self.limit_connections.clone(),
                shutdown: Shutdown::new(self.notify_shutdown.subscribe()),
                _shutdown_complete: self.shutdown_complete_tx.clone(),
            };

            tokio::spawn(async move {
                if let Err(err) = handler.run().await {
                    error!(cause = ?err, "connection error");
                }
            });
        }
    }

// 多线程获取tcp流
    async fn accept(&self) -> Result<TcpStream> {
        let mut try_time = 1;

        loop {
            match self.listener.accept().await {
                Ok((socket, _)) => return Ok(socket),
                Err(err) => {
                    if try_time > 64 {
                        return Err(err.into());
                    }
                }
            }

            time::sleep(Duration::from_secs(try_time)).await;

            try_time *= 2
        }
    }
}

// 运行服务
pub async fn run<T: Future>(listener: TcpListener, shutdown: T) {
    init();
    let (notify_shutdown, _) = broadcast::channel(1);
    let (shutdown_complete_tx, shutdown_complete_rx) = mpsc::channel(1);

    let mut server = HttpServer {
        listener,
        limit_connections: Arc::new(Semaphore::new(500)),
        notify_shutdown,
        shutdown_complete_tx,
        shutdown_complete_rx,
    };

    tokio::select! {
        res = server.run() => {
            if let Err(err) = res {
                error!(cause = %err, "failed to accept");
            }
        }
        _ = shutdown => {
            info!("shutting down");
        }
    };

    drop(server.notify_shutdown);
    drop(server.shutdown_complete_tx);
    let _ = server.shutdown_complete_rx.recv().await;
}

struct Handler {
    socket: TcpStream,
    limit_connections: Arc<Semaphore>,
    shutdown: Shutdown, // 监听ctrl+c进行退出，下面实现
    _shutdown_complete: mpsc::Sender<()>,
}

impl Handler {
    async fn run(&mut self) -> Result<()> {
        while !self.shutdown.is_shutdown() {
            let request = HttpRequest::from(&mut self.socket).await; // 从tcp流中获取request
            println!("request is: {:?}", request.resource);
            let resp = Router::route(&request).await; // 进行处理
            let resp_str: String = resp.into();
            self.socket
                .write(resp_str.as_bytes() as &[u8])
                .await
                .unwrap(); // 写入tcp流返回
            self.socket.flush().await.unwrap();
        }

        Ok(())
    }
}

impl Drop for Handler {
    fn drop(&mut self) {
        self.limit_connections.add_permits(1);
    }
}
```

# Run!

在我们的 main.rs 中让你的项目跑起来吧！

```rust
mod config;
mod http;
mod router;
mod types;

use tokio::net::TcpListener;
use tokio::signal;
use tokio::spawn;

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("localhost:8888").await.unwrap();

    spawn(async move {
        http::run(listener, signal::ctrl_c()).await;
    });
}
```

7. 最后
   这个 demo 是一个极简的 httpserver，他有很多不足，但是他的核心是解析 tcp 流中的 http 请求并处理返回，这也是很多 http 库的最核心的部分，这个 demo 并不复杂，主要是熟悉 rust 的编程习惯以及 tokio 的使用，如果想要了解更复杂的 http_server，可以看一下我之前的 Gin 简易实现。
