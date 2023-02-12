---
title: Gin 简易实现
date: 2021-11-2 18:10:04
tags:
	- Golang
  - Gin
  - Web Server
categories: 学习笔记
---

# 0. 前言

各种语言都包含有 http 库，但是在我们真正开发的时候，我们会使用各种 web 框架，有 spring 一样大而全的框架，也有 koa 一样小而美的框架，但他们的核心其实都是一样的，帮助我们解决频繁手动处理的部分，简化我们的开发负担，提升开发体验。本次我们会实现一个简易的 gin，将会有以下功能：

> 1. 路由树
> 2. 分组控制
> 3. 中间件
> 4. 错误恢复

项目的架构图如下：
 ![Gin 架构](/image/gin/471676227283_.pic.jpg)
其中黑色的为 request，红色的为 response，整体请求的处理逻辑为： Request -> Group -> Middleware -> RouterTree -> Handler -> Middleware -> Response 接下来，我将会由大到小带你一步步的实现这个 mini 版 Gin。

# 1. 最简框架

我们的最简框架要求十分简单，就是能够处理请求即可。
 ![http](/image/gin/481676227283_.pic.jpg)
go 中内置了 net/http 库用于处理 http 请求，我们不妨使用它来完成我们的最简框架。

```golang
// akt.go
package akt

import (
        "fmt"
        "net/http"
)

type HandlerFunc func(http.ResponseWriter, *http.Request)

type Engine struct {
        router map[string]HandlerFunc // 简易的路由映射
}

func (engine *Engine) addRoute(method string, pattern string, handler HandlerFunc) {
        name := method + "-" + pattern
        engine.router[name] = handler
}

func (engine *Engine) GET(pattern string, handler HandlerFunc) {
        engine.addRoute("GET", pattern, handler)
}

func (engine *Engine) POST(pattern string, handler HandlerFunc) {
        engine.addRoute("POST", pattern, handler)
}

func (engine *Engine) Run(addr string) (err error) {
        return http.ListenAndServe(addr, engine)
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
        key := req.Method + "-" + req.URL.Path
        if handler, ok := engine.router[key]; ok != false {
                handler(w, req)
        } else {
                fmt.Fprintf(w, "404 NOT FOUND: %s\n", req.URL)
        }
}

func New() *Engine {
        return &Engine{router: make(map[string]HandlerFunc)}
}

```

代码非常简单，就是通过实现 ServeHTTP 接口的实例完成一个服务器的搭建，然后我们自己维护了一个路由映射表，通过请求的 Method 和 URL.Path 决定那个路由 handler 来进行处理。

Node 版

```typescript
import http from 'http';

class Akt {
  private server: http.Server;
  private router: Map<string, http.RequestListener>;

  constructor(requestListener?: http.RequestListener) {
    this.server = http.createServer(requestListener);
    this.router = new Map();
    this.server.on('request', (requset: http.IncomingMessage, response: http.ServerResponse) => {
      const handler = this.router.get(`${requset.method}-${requset.url}`);
      if (handler) {
        handler(requset, response);
      } else {
        console.log(`404 NOT FOUND: ${requset.url}`);
        response.statusCode = 404;
        response.statusMessage = `404 NOT FOUND: ${requset.url}`;
        response.end();
      }
    });
  }

  private addRoute = (method: string, pattern: string, handler: http.RequestListener) => {
    this.router.set(`${method}-${pattern}`, handler);
  };

  get = (pattern: string, handler: http.RequestListener) => {
    this.addRoute('GET', pattern, handler);
  };
  post = (pattern: string, handler: http.RequestListener) => {
    this.addRoute('POST', pattern, handler);
  };

  getServer = () => {
    return this.server;
  };

  run = (addr: string, listeningListener?: () => void) => {
    return this.server.listen(addr, listeningListener);
  };
}

const akt = (requestListener?: http.RequestListener) => new Akt(requestListener);

export default akt;
```

# 2. 封装 Context

将`http.ResponseWriter`和`http.Request`封装成为一个 Context 会带来很多的好处，比如简化接口调用，减少用户编写大量重复的代码等等，封装之后，用户想得到的东西只需要从 ctx 中直接获取即可。

Context 的封装实现：

```golang
// context.go
package akt

import (
        "encoding/json"
        "fmt"
        "net/http"
)

type Context struct {
        Req        *http.Request
        Writer     http.ResponseWriter
        Path       string
        Method     string
        StatusCode int
}

type Obj map[string]interface{}

func newContext(writer http.ResponseWriter, req *http.Request) *Context {
        return &Context{
                Req:    req,
                Writer: writer,
                Path:   req.URL.Path,
                Method: req.Method,
        }
}

func (c *Context) PostForm(key string) string {
        return c.Req.FormValue(key)
}

func (c *Context) Query(key string) string {
        return c.Req.URL.Query().Get(key)
}

func (c *Context) Status(code int) {
        c.StatusCode = code
        c.Writer.WriteHeader(code)
}

func (c *Context) SetHeader(key string, value string) {
        c.Writer.Header().Set(key, value)
}

func (c *Context) String(code int, format string, values ...interface{}) {
        c.Status(code)
        c.SetHeader("Content-Type", "text/plain")
        c.Writer.Write([]byte(fmt.Sprintf(format, values...)))
}

func (c *Context) JSON(code int, obj interface{}) {
        c.SetHeader("Content-Type", "application/json")
        c.Status(code)
        encoder := json.NewEncoder(c.Writer)
        if err := encoder.Encode(obj); err != nil {
                http.Error(c.Writer, err.Error(), 500)
        }
}

func (c *Context) Data(code int, data []byte) {
        c.Status(code)
        c.Writer.Write(data)
}

func (c *Context) HTML(code int, html string) {
        c.SetHeader("Content-Type", "text/html")
        c.Status(code)
        c.Writer.Write([]byte(html))
}
```

这时候 router 的 handler 的参数就会变成 Context 了

```golang
// router.go
package akt

import (
        "net/http"
)

type router struct {
        handlers map[string]HandlerFunc
}

func newRouter() *router {
        return &router{handlers: make(map[string]HandlerFunc)}
}

func (router *router) addRoute(method string, pattern string, handler HandlerFunc) {
        name := method + "-" + pattern
        router.handlers[name] = handler
}

func (router *router) handle(c *Context) {
        key := c.Method + "-" + c.Path
        if hander, ok := router.handlers[key]; ok {
                hander(c)
        } else {
                c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
        }
}

```

我们的入口文件就会变得比较简单了。

```golang
// akt.go
package akt

import (
        "net/http"
)

type HandlerFunc func(*Context)

type Engine struct {
        router *router
}

func (engine *Engine) GET(pattern string, handler HandlerFunc) {
        engine.router.addRoute("GET", pattern, handler)
}

func (engine *Engine) POST(pattern string, handler HandlerFunc) {
        engine.router.addRoute("POST", pattern, handler)
}

func (engine *Engine) Run(addr string) (err error) {
        return http.ListenAndServe(addr, engine)
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
        c := newContext(w, req)
        engine.router.handle(c)
}

func New() *Engine {
        return &Engine{router: newRouter()}
}

```

Node 版

Context.ts:

```typescript
import http from 'http';
import querystring from 'querystring';
import url from 'url';
import util from 'util';

export type ContextOptionsType = {
  [propsName: string]: any;
};

export class Context {
  req: http.IncomingMessage & { body?: any };
  res: http.ServerResponse;
  path: string;
  method: string;
  statusCode: number;

  private URL: url.URL;
  constructor(requset: http.IncomingMessage, response: http.ServerResponse) {
    this.req = requset;
    this.res = response;
    this.URL = new url.URL(`http://${this.req.headers.host}${this.req.url}`);
    this.path = this.URL.pathname;
    this.method = requset.method;
  }

  query = (key: string) => {
    return this.URL.searchParams.get(key);
  };

  postForm = (key: string) => {
    return this.req.body?.[key];
  };

  status = (code: number, message?: string) => {
    this.statusCode = code;
    this.res.statusCode = code;
    this.res.statusMessage = message;
    return this;
  };

  setHeader = (key: string, value: string) => {
    this.res.setHeader(key, value);
    return this;
  };

  string = (code: number, str: string) => {
    this.setHeader('Content-Type', 'text/plain');
    this.status(code);
    this.res.end(str);
  };

  JSON = (code: number, obj: object) => {
    this.setHeader('Content-Type', 'application/json');
    this.status(code);
    try {
      const data = JSON.stringify(obj);
      this.res.end(data);
    } catch (e) {
      console.log(e);
      this.res.end(JSON.stringify({ err: e.toStirng() }));
    }
  };

  data = (code: number, data: any) => {
    this.status(code);
    this.res.end(data);
  };

  HTML = (code: number, HTML: string) => {
    this.setHeader('Content-Type', 'text/html');
    this.status(code);
    this.res.end(HTML);
  };
}
```

router.ts

```typescript
import http from 'http';
import { Context } from './context';
import { HandlerFunc } from './index';

export class Router {
  private handlers: Map<string, HandlerFunc>;

  constructor() {
    this.handlers = new Map();
  }

  addRoute = (method: string, pattern: string, handler: HandlerFunc) => {
    this.handlers.set(`${method}-${pattern}`, handler);
  };

  handle = (ctx: Context) => {
    const key = `${ctx.method}-${ctx.path}`;
    const handler = this.handlers.get(key);
    if (handler) {
      handler(ctx);
    } else {
      ctx.string(404, `404 NOT FOUND: ${ctx.path}`);
    }
  };
}
```

akt.ts

```typescript
import http from 'http';
import { Context } from './context';
import { Router } from './router';

export { Context as AktContext } from './context';
export { Router as AktRouter } from './router';
import util from 'util';
import querystring from 'querystring';
import parse from 'co-body';

export type HandlerFunc = (ctx: Context) => void;

class Akt {
  private server: http.Server;
  private router: Router;

  constructor(requestListener?: http.RequestListener) {
    this.server = http.createServer(requestListener);
    this.router = new Router();
    this.server.on(
      'request',
      (requset: http.IncomingMessage & { body?: any }, response: http.ServerResponse) => {
        // 处理post请求
        requset.body = '';
        requset.on('data', chuck => {
          requset.body += chuck;
        });
        requset.on('end', () => {
          if (requset.headers['content-type']?.includes('application/json')) {
            requset.body = JSON.parse(requset.body);
          } else if (requset.method === 'POST') {
            requset.body = querystring.parse(requset.body);
          }
          const ctx = new Context(requset, response);
          this.router.handle(ctx);
        });
      },
    );
  }

  get = (pattern: string, handler: HandlerFunc) => {
    this.router.addRoute('GET', pattern, handler);
  };
  post = (pattern: string, handler: HandlerFunc) => {
    this.router.addRoute('POST', pattern, handler);
  };

  getServer = () => {
    return this.server;
  };

  run = (addr: string, listeningListener?: () => void) => {
    return this.server.listen(addr, listeningListener);
  };
}

const akt = (requestListener?: http.RequestListener) => new Akt(requestListener);

export default akt;
```

# 3. 动态路由解析

前面我们的路由使用了一个 map 进行储存，这个问题其实是非常明显的，我们并不支持类似于`/docs/:name/history`或者`/static/*filename`等形式的路由解析，这将是非常致命的。

现在我们要实现如下的架构：
 ![router](/image/gin/4491676227284_.pic.jpg)
对于路由的解析，我们可以使用树数据结构进行处理：

我们的路由可以根据/划分成一个又一个树状节点，然后我们根据每个节点的样式，来决定判断是否正确，寻找下一个节点，最终节点的 handler 进行处理请求。

首先我们先实现每个节点的数据结构：
 ![tree](/image/gin/501676227284_.pic.jpg)

```golang
// trie.go
package akt

import "strings"

type node struct {
        pattern  string
        part     string
        children []*node
        isWild   bool
}

func (n *node) matchChild(part string) *node { // 匹配第一个节点，用于创建
        for _, child := range n.children {
                if child.part == part || child.isWild {
                        return child
                }
        }
        return nil
}

func (n *node) matchChildren(part string) []*node { // 匹配所有节点，用于创建
        nodes := make([]*node, 0)
        for _, child := range n.children {
                if child.part == part || child.isWild {
                        nodes = append(nodes, child)
                }
        }

        return nodes
}

func (n *node) insert(pattern string, parts []string, height int) { // 插入节点
        if len(parts) == height {
                n.pattern = pattern
                return
        }

        part := parts[height]
        child := n.matchChild(part)
        if child == nil {
                child = &node{part: part, isWild: part[0] == ':' || part[0] == '*'}
                n.children = append(n.children, child)
        }
        child.insert(pattern, parts, height+1)
}

func (n *node) search(parts []string, height int) *node { // 查询节点
        if len(parts) == height || strings.HasPrefix(n.part, "*") {
                if n.pattern == "" {
                        return nil
                }
                return n
        }

        part := parts[height]
        children := n.matchChildren(part)

        for _, child := range children {
                result := child.search(parts, height)
                if result != nil {
                        return result
                }
        }

        return nil
}
```

然后我们来更新路由文件

```golang
// router.go
package akt

import (
        "net/http"
        "strings"
)

type router struct {
        handlers map[string]HandlerFunc
        roots    map[string]*node
}

func parsePattern(pattern string) []string {
        vs := strings.Split(pattern, "/")

        parts := make([]string, 0)
        for _, item := range vs {
                if item != "" {
                        parts = append(parts, item)
                        if item[0] == '*' {
                                break
                        }
                }
        }

        return parts
}

func newRouter() *router {
        return &router{handlers: make(map[string]HandlerFunc), roots: make(map[string]*node)}
}

func (router *router) addRoute(method string, pattern string, handler HandlerFunc) {
        parts := parsePattern(pattern)

        name := method + "-" + pattern

        _, ok := router.roots[method]

        if !ok {
                router.roots[method] = &node{}
        }

        router.roots[method].insert(pattern, parts, 0)
        router.handlers[name] = handler
}

func (router *router) getRoute(method string, path string) (*node, map[string]string) {
        searchParts := parsePattern(path)

        params := make(map[string]string)

        root, ok := router.roots[method]

        if !ok {
                return nil, nil
        }

        n := root.search(searchParts, 0)

        if n != nil {
                parts := parsePattern(n.pattern)
                for i, part := range parts {
                        if part[0] == ':' {
                                params[part[1:]] = searchParts[i]
                        }
                        if part[0] == '*' && len(part) > 1 {
                                params[part[1:]] = strings.Join(searchParts[i:], "/")
                                break
                        }
                }
                return n, params
        }

        return nil, nil
}

func (router *router) handle(c *Context) {
        n, params := router.getRoute(c.Method, c.Path)
        if n != nil {
                c.Params = params
                key := c.Method + "-" + n.pattern
                router.handlers[key](c)
        } else {
                c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
        }
}

```

对由于我们支持了`/docs/:name/history`或者`/static/*filename`等形式的路由解析，我们需要对 Context 提供一个 Params

```golang
package akt

import (
        "encoding/json"
        "fmt"
        "net/http"
)

type Context struct {
        Req        *http.Request
        Writer     http.ResponseWriter
        Path       string
        Method     string
        StatusCode int
        // 新增
        Params     map[string]string
}
// ...

func (c *Context) Param(key string) string {
        value, _ := c.Params[key]
        return value
}

```

Node 版本：

tire.ts

```typescript
class TrieNode {
  pattern: string;
  part: string;
  children: TrieNode[];
  isWild: boolean;

  constructor(pattern?: string, part?: string, children?: TrieNode[], isWild?: boolean) {
    this.pattern = pattern || '';
    this.part = part || '';
    this.children = children || [];
    this.isWild = isWild || false;
  }

  matchChild = (part: string) => {
    for (let child of this.children) {
      if (child.part === part || child.isWild) {
        return child;
      }
    }

    return null;
  };

  matchChildren = (part: string) => {
    const nodes: TrieNode[] = [];
    for (let child of this.children) {
      if (child.part === part || child.isWild) {
        nodes.push(child);
      }
    }

    return nodes;
  };

  insert = (pattern: string, parts: string[], height: number) => {
    if (parts.length === height) {
      this.pattern = pattern;
      return;
    }

    const part = parts[height];
    let child = this.matchChild(part);

    if (!child) {
      child = new TrieNode('', part, [], part[0] === ':' || part[0] === '*');
      this.children.push(child);
    }

    child.insert(pattern, parts, height + 1);
  };

  search = (parts: string[], height: number): TrieNode | null => {
    if (parts.length === height || this.part.includes('*')) {
      if (this.pattern === '') {
        return null;
      }
      return this;
    }

    const part = parts[height];
    const children = this.matchChildren(part);

    for (let child of children) {
      const result = child.search(parts, height + 1);
      if (result) {
        return result;
      }
    }

    return null;
  };
}

export default TrieNode;
```

router.ts

```typescript
import { Context } from './context';
import { HandlerFunc } from './akt';
import TrieNode from './trie';

export const parsePattern = (pattern: string) => {
  const vs = pattern.split('/');
  const parts: string[] = [];
  for (let item of vs) {
    if (item !== '') {
      parts.push(item);
      if (item[0] === '*') {
        break;
      }
    }
  }

  return parts;
};

export class Router {
  private handlers: Map<string, HandlerFunc>;
  roots: Map<string, TrieNode>;

  constructor() {
    this.handlers = new Map();
    this.roots = new Map();
  }

  // 添加路由
  addRoute = (method: string, pattern: string, handler: HandlerFunc) => {
    const parts = parsePattern(pattern);
    if (!this.roots.get(method)) {
      const node = new TrieNode('', '', [], false);
      this.roots.set(method, node);
    }
    this.roots.get(method).insert(pattern, parts, 0);
    this.handlers.set(`${method}-${pattern}`, handler);
  };

  handle = (ctx: Context) => {
    const { node, params } = this.getRouter(ctx.method, ctx.path);
    if (node) {
      ctx.params = params;
      const key = `${ctx.method}-${node.pattern}`;
      const handler = this.handlers.get(key);
      handler(ctx);
    } else {
      ctx.string(404, `404 NOT FOUND: ${ctx.path}`);
    }
  };
  getRouter = (method: string, path: string) => {
    const searchParts = parsePattern(path);
    const params: Map<string, string> = new Map();
    const root = this.roots.get(method);

    if (!root) {
      return null;
    }

    const n = root.search(searchParts, 0);

    if (n) {
      const parts = parsePattern(n.pattern);
      for (let i = 0; i < parts.length; i++) {
        if (parts[i][0] === ':') {
          params.set(parts[i].slice(1), searchParts[i]);
        }
        if (parts[i][0] === '*') {
          params.set(parts[i].slice(1), searchParts.slice(i).join('/'));
          break;
        }
      }
      return {
        node: n,
        params,
      };
    }

    return null;
  };
}
```

context.ts

```typescript
import http from 'http';
import url from 'url';

export type ContextOptionsType = {
  [propsName: string]: any;
};

export class Context {
  req: http.IncomingMessage & { body?: any };
  res: http.ServerResponse;
  path: string;
  method: string;
  statusCode: number;
  // 新增
  params: Map<string, string>;
  // ...
  param = (key: string) => {
    return this.params.get(key);
  };
}
```

# 4. 路由分组

实际业务场景中，往往一些路由需要相似的处理，比如有的路由需要鉴权，有的路由是接口路由等等。由于需要相似处理的路由往往具有相同的前缀，因此我们可以选用前缀作为路由分组的依据。

架构图如下：
 ![group](/image/gin/511676227285_.pic.jpg)

Group 的实现：

```golang
// group.go
package akt

type RouterGroup struct {
        prefix      string
        parent      *RouterGroup
        middlewares []HandlerFunc
        engine      *Engine
}

func (group *RouterGroup) Group(prefix string) *RouterGroup {
        newGroup := &RouterGroup{
                prefix:      group.prefix + prefix,
                parent:      group,
                middlewares: make([]HandlerFunc, 0),
                engine:      group.engine,
        }
        group.engine.groups = append(group.engine.groups, newGroup)
        return newGroup
}

func (group *RouterGroup) addRoute(method string, comp string, handler HandlerFunc) {
        pattern := group.prefix + comp
        group.engine.router.addRoute(method, pattern, handler)
}

func (group *RouterGroup) GET(comp string, handler HandlerFunc) {
        group.addRoute("GET", comp, handler)
}

func (group *RouterGroup) POST(comp string, handler HandlerFunc) {
        group.addRoute("POST", comp, handler)
}

```

然后我们小幅度修改下 server

```golang
// akt
package akt

import (
        "net/http"
)

type HandlerFunc func(*Context)

type Engine struct {
    // 新增
        *RouterGroup
        router *router
    // 新增
        groups []*RouterGroup
}

func (engine *Engine) GET(pattern string, handler HandlerFunc) {
        engine.router.addRoute("GET", pattern, handler)
}

func (engine *Engine) POST(pattern string, handler HandlerFunc) {
        engine.router.addRoute("POST", pattern, handler)
}

func (engine *Engine) Run(addr string) (err error) {
        return http.ListenAndServe(addr, engine)
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
        c := newContext(w, req)
        engine.router.handle(c)
}

// 新增
func New() *Engine {
        engine := &Engine{router: newRouter()}
        engine.RouterGroup = &RouterGroup{engine: engine}
        engine.groups = []*RouterGroup{engine.RouterGroup}
        return engine
}

```

Node 版本 akt.ts

```typescript
import http from 'http';
import querystring from 'querystring';

import getContext, { AktContext } from './context';
import getRouter, { AktRouter } from './router';

export type RouterGroupType = {
  // 前缀
  prefix: string;
  middlewares: HandlerFunc[];
  // 父组
  parent: RouterGroupType | AktType;
  // 根akt
  akt: AktType;
  // 建立新分组
  group: (prefix: string, group?: RouterGroupType) => RouterGroupType;
  // 添加路由
  addRoute: (
    method: string,
    pattern: string,
    handler: HandlerFunc,
    group?: RouterGroupType,
  ) => void;
  get: (pattern: string, handler: HandlerFunc) => void;
  post: (pattern: string, handler: HandlerFunc) => void;
};

export type HandlerFunc = (ctx: AktContext) => void;

export type AktType = {
  server: http.Server;
  router: AktRouter;
  groups: (RouterGroupType | AktType)[];
  get: (pattern: string, handler: HandlerFunc) => void;
  post: (pattern: string, handler: HandlerFunc) => void;
  run: (addr: string, listeningListener?: () => void) => http.Server;
} & Omit<RouterGroupType, 'addRoute' | 'post' | 'get'>;

const akt = (requestListener?: http.RequestListener) => {
  const akt: AktType = {
    // server
    server: http.createServer(requestListener),
    // 路由
    router: getRouter(),
    prefix: '',
    middlewares: [],
    parent: null,
    akt: null,
    groups: [],
    // 设置get请求的路由
    get: (pattern: string, handler: HandlerFunc, group?: RouterGroupType) => {
      akt.router.addRoute('GET', pattern, handler);
    },
    // 设置post请求的路由
    post: (pattern: string, handler: HandlerFunc, group?: RouterGroupType) => {
      akt.router.addRoute('POST', pattern, handler);
    },
    // 设置监听端口
    run: (addr: string, listeningListener?: () => void) =>
      akt.server.listen(addr, listeningListener),
    // 建立新分组
    group: (prefix: string, parentGroup?: RouterGroupType) => {
      const group = parentGroup || akt;
      const newGroup: RouterGroupType = {
        prefix: group.prefix + prefix,
        parent: group,
        akt: akt,
        middlewares: [],
        group: (prefix: string) => akt.group(prefix, newGroup),
        addRoute: (method: string, pattern: string, handler: HandlerFunc) => {
          pattern = newGroup.prefix + pattern;
          akt.router.addRoute(method, pattern, handler);
        },
        get: (pattern: string, handler: HandlerFunc) => {
          newGroup.addRoute('GET', pattern, handler, newGroup);
        },
        post: (pattern: string, handler: HandlerFunc) => {
          newGroup.addRoute('POST', pattern, handler, newGroup);
        },
      };

      group.akt.groups.push(newGroup);

      return newGroup;
    },
  };

  // 监听请求
  akt.server.on(
    'request',
    (requset: http.IncomingMessage & { body?: any }, response: http.ServerResponse) => {
      // 处理post请求
      requset.body = '';
      requset.on('data', chuck => {
        requset.body += chuck;
      });
      requset.on('end', () => {
        if (requset.headers['content-type']?.includes('application/json')) {
          requset.body = JSON.parse(requset.body);
        } else if (requset.method === 'POST') {
          requset.body = querystring.parse(requset.body);
        }
        // data接收完毕后创建context,并执行
        const ctx = getContext(requset, response);
        akt.router.handle(ctx);
      });
    },
  );

  akt.akt = akt;
  akt.groups.push(akt);

  return akt;
};

export default akt;
```

# 5. 中间件

距离我们最开始画的架构图现在只缺中间件了，想必用过 koa 的同学肯定都知道中间件是什么啦，我就不具体解释了，接下来我们就来实现它。

```golang
// Context.go

package akt

import (
        "encoding/json"
        "fmt"
        "net/http"
)

type Context struct {
        Req        *http.Request
        Writer     http.ResponseWriter
        Path       string
        Method     string
        StatusCode int
        Params     map[string]string
        index      int  // 指示位置
        handlers   []HandlerFunc // 存储中间件
}

// 。。。

func (c *Context) Next() { // 调用Next()执行下一个中间件
        c.index++
        s := len(c.handlers)
        for c.index < s {
                c.handlers[c.index](c)
                c.index++
        }
}

// group.go
package akt

type RouterGroup struct {
        prefix      string
        parent      *RouterGroup
        middlewares []HandlerFunc
        engine      *Engine
}
// ...

func (group *RouterGroup) Use(handler HandlerFunc) { // 每个分组添加中间件
        group.middlewares = append(group.middlewares, handler)
}

// router.go
package akt

import (
        "net/http"
        "strings"

        "fmt"
)

type router struct {
        handlers map[string]HandlerFunc
        roots    map[string]*node
}
// ...
func (router *router) handle(c *Context) {
        n, params := router.getRoute(c.Method, c.Path)
        fmt.Printf("%v", n)
        if n != nil { // 添加上不同分组的中间件
                c.Params = params
                key := c.Method + "-" + n.pattern
                handler := router.handlers[key]
                c.handlers = append(c.handlers, handler)
        } else {
                c.String(http.StatusNotFound, "404 NOT FOUND: %s\n", c.Path)
        }
        c.Next()
}
```

Node 版本

```typescript
// context.ts
import http from 'http';
import url from 'url';

import { HandlerFunc } from './akt';

export type ContextOptionsType = {
  [propsName: string]: any;
};

export class Context {
  req: http.IncomingMessage & { body?: any };
  res: http.ServerResponse;
  path: string;
  method: string;
  statusCode: number;
  params: Map<string, string>;
  private index: number;
  hanlders: HandlerFunc[];
  resData: any;

  private URL: url.URL;
  constructor(requset: http.IncomingMessage, response: http.ServerResponse) {
    this.req = requset;
    this.res = response;
    this.URL = new url.URL(`http://${this.req.headers.host}${this.req.url}`);
    this.path = this.URL.pathname;
    this.method = requset.method;
    this.params = new Map();
    this.index = -1;
    this.hanlders = [];
    this.resData = null;
  }
  // ...

  next = async () => {
    this.index++;
    while (this.index < this.hanlders.length) {
      this.hanlders[this.index](this);
      this.index++;
    }
  };
}

// akt.ts
import http from 'http';
import querystring from 'querystring';

import { Context } from './context';
import { Router } from './router';

// 新增
export type HandlerFunc = (ctx: Context) => void;

class RouterGroup {
  prefix: string;
  protected parent: RouterGroup | Akt;
  middlewares: HandlerFunc[];
  protected akt: Akt;

  constructor(prefix: string, parent: RouterGroup | Akt, akt: Akt) {
    this.prefix = (parent ? parent.prefix : '') + prefix;
    this.parent = parent;
    this.akt = akt;
    // 新增
    this.middlewares = [];
  }

  group = (prefix: string) => {
    const newGruop = new RouterGroup(prefix, this, this.akt);
    this.akt.groups.push(newGruop);
    return newGruop;
  };

  get = (pattern: string, handler: HandlerFunc) => {
    this.addRoute('GET', pattern, handler);
  };
  post = (pattern: string, handler: HandlerFunc) => {
    this.addRoute('POST', pattern, handler);
  };

  protected addRoute = (method: string, comp: string, handler: HandlerFunc) => {
    const pattern = this.prefix + comp;
    this.akt.addRoute(method, pattern, handler);
  };

  // 新增
  use = (handler: HandlerFunc) => {
    this.middlewares.push(handler);
  };
}

class Akt extends RouterGroup {
  private server: http.Server;
  private router: Router;
  groups: (RouterGroup | Akt)[];

  constructor(requestListener?: http.RequestListener) {
    super('', null, null);
    this.akt = this;
    this.parent = null;
    this.server = http.createServer(requestListener);
    this.router = new Router();
    this.groups = [this];
    // 监听请求
    this.server.on(
      'request',
      (requset: http.IncomingMessage & { body?: any }, response: http.ServerResponse) => {
        // 处理post请求
        requset.body = '';
        requset.on('data', chuck => {
          requset.body += chuck;
        });
        requset.on('end', () => {
          if (requset.headers['content-type']?.includes('application/json')) {
            requset.body = JSON.parse(requset.body);
          } else if (requset.method === 'POST') {
            requset.body = querystring.parse(requset.body);
          }
          // 新增
          let middlewares: HandlerFunc[] = [];
          for (let i = 0; i < this.groups.length; i++) {
            if (requset.url.includes(this.groups[i].prefix)) {
              middlewares = [...middlewares, ...this.groups[i].middlewares];
            }
          }
          const ctx = new Context(requset, response);
          ctx.hanlders = middlewares;
          this.router.handle(ctx);
        });
      },
    );
  }

  // 返回server实例，可用于多进程
  getServer = () => {
    return this.server;
  };

  // 设置监听端口
  run = (addr: string, listeningListener?: () => void) => {
    return this.server.listen(addr, listeningListener);
  };

  addRoute = (method: string, pattern: string, handler: HandlerFunc) => {
    this.router.addRoute(method, pattern, handler);
  };
}

const akt = (requestListener?: http.RequestListener) => new Akt(requestListener);

export default akt;
```

# 6. 错误恢复

现在我们的项目已经基本完成了，但是存在一个问题，如果抛出 panic，导致我们的项目直接挂掉，因此，我们需要编写一个中间件用于处理错误恢复。

```golang
package akt

import (
        "fmt"
        "log"
        "net/http"
        "runtime"
        "strings"
)


func trace(message string) string { // 展示堆栈信息
        var pcs [32]uintptr

        n := runtime.Callers(3, pcs[:])

        var str strings.Builder

        str.WriteString(message + "\nTraceback:")

        for _, pc := range pcs[:n] {
                fn := runtime.FuncForPC(pc)
                file, line := fn.FileLine(pc)
                str.WriteString(fmt.Sprintf("\n\t%s%d", file, line))
        }

        return str.String()

}


func Recovery(c *Context) { // 处理错误
        defer func() {
                if err := recover(); err != nil {
                        message := fmt.Sprintf("%s", err)
                        log.Printf("%s\n\n", trace(message))
                        c.String(http.StatusInternalServerError, "Internal Server Error")
                        if c.onError != nil {
                                c.onError(err)
                        }
                }
        }()

        c.Next()
}

```

Node:

```typescript
// akt.ts
import http from 'http';
import querystring from 'querystring';
import { join, resolve, parse } from 'path';
import { createReadStream } from 'fs';
import { stat } from 'fs/promises';
import { getType } from 'mime';
import pug from 'pug';

import { Context } from './context';
import { Router } from './router';
import { AktContext } from 'classVersion';

export type HandlerFunc = (ctx: Context) => void;

const createStaticHandler = (root: string): HandlerFunc => {
  const absolutePath = join(resolve('.'), root);
  return async (ctx: AktContext) => {
    let pipe: http.ServerResponse;
    try {
      const filePath = join(absolutePath, ctx.param('filepath'));
      const fileName = await stat(filePath);
      if (fileName.isFile()) {
        const ext = parse(filePath).ext;
        const mimeType = getType(ext);
        ctx.setHeader('Content-Type', mimeType);
        pipe = createReadStream(filePath).pipe(ctx.res);
        await new Promise(resolve => {
          pipe.on('end', () => {
            resolve('');
          });
        });
      } else {
        throw Error('File is not found!');
      }
    } catch (e) {
      if (pipe) pipe.destroy();
      if (ctx.akt.onError) ctx.akt.onError(e);
      ctx.string(404, 'File is not found!');
      ctx.setHeader('Content-Type', 'text/plain');
      ctx.res.end();
    }
  };
};

class RouterGroup {
  prefix: string;
  protected parent: RouterGroup | Akt;
  middlewares: HandlerFunc[];
  protected akt: Akt;
  onError: (err: any, p?: Promise<any>) => void;

  constructor(prefix: string, parent: RouterGroup | Akt, akt: Akt) {
    this.prefix = (parent ? parent.prefix : '') + prefix;
    this.parent = parent;
    this.akt = akt;
    this.middlewares = [];
  }

  group = (prefix: string) => {
    const newGruop = new RouterGroup(prefix, this, this.akt);
    this.akt.groups.push(newGruop);
    return newGruop;
  };

  get = (pattern: string, handler: HandlerFunc) => {
    this.addRoute('GET', pattern, handler);
  };
  post = (pattern: string, handler: HandlerFunc) => {
    this.addRoute('POST', pattern, handler);
  };

  protected addRoute = (method: string, comp: string, handler: HandlerFunc) => {
    const pattern = this.prefix + comp;
    this.akt.addRoute(method, pattern, handler);
  };

  use = (handler: HandlerFunc) => {
    this.middlewares.push(handler);
  };

  static = (relativePath: string, root: string) => {
    const handler = createStaticHandler(root);
    const urlPattern = join(relativePath, '/*filepath').replace(/\\/g, '/');
    this.get(urlPattern, handler);
  };
}

export class Akt extends RouterGroup {
  private server: http.Server;
  private router: Router;
  groups: (RouterGroup | Akt)[];
  pug: typeof pug;
  templateRoot: string;

  constructor(requestListener?: http.RequestListener) {
    super('', null, null);
    this.akt = this;
    this.parent = null;
    this.server = http.createServer(requestListener);
    this.router = new Router();
    this.groups = [this];
    this.pug = pug;
    this.templateRoot = null;
    // 监听请求
    this.server.on(
      'request',
      (requset: http.IncomingMessage & { body?: any }, response: http.ServerResponse) => {
        // 处理post请求
        requset.body = '';
        requset.on('data', chuck => {
          requset.body += chuck;
        });
        requset.on('end', () => {
          if (requset.headers['content-type']?.includes('application/json')) {
            requset.body = JSON.parse(requset.body);
          } else if (requset.method === 'POST') {
            requset.body = querystring.parse(requset.body);
          }
          let middlewares: HandlerFunc[] = [];
          for (let i = 0; i < this.groups.length; i++) {
            if (requset.url.includes(this.groups[i].prefix)) {
              middlewares = [...middlewares, ...this.groups[i].middlewares];
            }
          }
          const ctx = new Context(requset, response, this);
          ctx.hanlders = middlewares;
          this.router.handle(ctx);
        });
      },
    );

    // 新增
    // 捕获错误，用于防止服务器崩溃
    process.on('uncaughtException', function (err) {
      if (this.onError) this.onError(err);
      console.log(err.stack);
    });

    process.on('unhandledRejection', (reason, p) => {
      if (this.onError) this.onError(reason, p);
      console.log('Unhandled Rejection at: Promise', p, 'reason:', reason);
    });
  }

  // ...
}
```

# 7. 写在最后

至此，我们已经将 Gin 比较核心的几个点进行了实现，Gin 也是小而美的框架，虽然有 14k 的代码，但是其中的测试代码达到了 9k，还有一些例如静态资源获取，模板渲染等功能，因为时间篇幅有限，我实现了一些（[静态资源获取](https://github.com/akitaSummer/akt/blob/main/akt-golang/akt/group.go)， [模板渲染](https://github.com/akitaSummer/akt/blob/main/akt-golang/akt/context.go)），但没有在文章中写出，有兴趣的同学可以去自己实现一下，感受自己编写一个框架的乐趣吧。

在这文章的最后，我还是要再一次感谢你的阅读，本人的能力有限，可能有错误之处也请指出，如果有空，我还是非常推荐你阅读一下 Gin 的代码，他的精妙之处是我这 demo 无以比拟的。
