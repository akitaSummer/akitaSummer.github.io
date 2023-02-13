---
title: groupcache简易实现
date: 2021-7-31 01:17:09
tags:
  - NodeJs
  - 缓存
  - 分布式
categories: 学习笔记
---

> 商业世界里，现金为王；架构世界里，缓存为王。

# 序言

缓存在我们计算机系统中无处不在，对于我们前端来说，从大的网页和引用的 JS/CSS 等静态文件的缓存到小的点赞功能，都会运用到缓存，缓存的存在可以减少耗时的数据库查询，支持更多流量。当我们在设计一个缓存系统的时候，我们也会遇到很多问题：

> 1. 内存不够了怎么办？
> 2. 并发写入冲突了怎么办？
> 3. 单机性能不够怎么办？
> 4. ......

本文根据 golang 官方库 groupcache 实现了一个简易的分布式缓存系统，它的具体架构如下：

![groupcache 架构](/image/groupcache/main.jpg)

主要分为查询本机缓存（黑色）和查询分布节点缓存（红色），其中 client 是指接受请求的服务器，他会将其分发到自身的每个 group，group 是用于处理请求，返回缓存数据的重要模块，它包含 cache，server 和 local value generate 三个模块，当一个查询进来后，会先调用 cache 去内存中查询缓存，如果未命中，则调用 server 去判断是否需要查询其他分布式节点，如果不需要，则调用自身 local value generate 生成缓存。

接下来我将会从最小粒度的 cache 开始，一步步搭建，带你构建出一个简易的 groupcache 实现。

# LRU: 缓存淘汰策略

由于缓存全部存储在内存中，内存是有限的，因此不可能无限制地添加数据。如何判断缓存是否“没用”是关键的问题。常用的缓存淘汰策略有三种：FIFO，LFU 和 LRU。

FIFO : FIFO 算法是一种比较容易实现的算法。它的思想是先进先出（FIFO，队列），这是最简单、最公平的一种思想，即如果一个数据是最先进入的，那么可以认为在将来它被访问的可能性很小。**空间满的时候，最先进入的数据会被最早置换（淘汰）掉。**

LFU: LFU（Least Frequently Used ，最近最少使用算法）也是一种常见的缓存算法。顾名思义，LFU 算法的思想是：如果一个数据在最近一段时间很少被访问到，那么可以认为在将来它被访问的可能性也很小。**因此，当空间满时，最小频率访问的数据最先被淘汰。**

LRU: LRU（The Least Recently Used，最近最久未使用算法）是一种常见的缓存算法，在很多分布式缓存系统（如 Redis, Memcached）中都有广泛使用。LRU 算法的思想是：如果一个数据在最近一段时间没有被访问到，那么可以认为在将来它被访问的可能性也很小。**因此，当空间满时，最久没有访问的数据最先被置换（淘汰）。**

groupcache 中使用了 LRU 作为 Cache 的缓存淘汰策略，我们将实现一个简单的 LRU：

![lru](/image/groupcache/lru.png)

绿色部分是一个 map，用于映射 key，value，使得复杂度为 O(1)，红色部分是一个双向连表，用于将值进行排序，其 CRUD 的复杂度为 O(1)。

数据结构实现：

```typescript
export class Entry {
  // 存储的数据类型
  key: string;
  value: any;
  constructor(key: string, value: any) {
    this.key = key;
    this.value = value;
  }
}
// 双向链表
export class ListNode {
  before: ListNode | null;
  next: ListNode | null;
  value: Entry;
  constructor(value: Entry, before?: ListNode, next?: ListNode) {
    this.value = value;
    this.before = before || null;
    this.next = next || null;
  }
}

export class List {
  root: null | ListNode;
  len: number;

  constructor() {
    this.root = null;
    this.len = 0;
  }

  private checkNodeExist(el: ListNode): ListNode {
    let node = this.root;
    if (!node) {
      throw Error('this node is not exist');
    }
    while (!!node) {
      if (el !== node) {
        node = node.next;
      } else {
        node = el;
        break;
      }
    }
    if (!node) {
      throw Error('this node is not exist');
    }
    return node;
  }

  MoveToFront(el: ListNode) {
    const node = this.checkNodeExist(el);
    if (node.before) {
      node.before.next = node.next;
      const head = this.root;
      this.root = node;
      this.root.before = null;
      this.root.next = head;
    }
  }

  Back(): ListNode | null {
    let node = this.root;
    if (node === null) {
      return node;
    }
    while (node.next !== null) {
      node = node.next;
    }
    return node;
  }

  Remove(el: ListNode) {
    const node = this.checkNodeExist(el);
    if (node.before) {
      node.before.next = node.next;
      node.next = null;
      this.len -= 1;
    } else {
      this.root = node.next;
      node.next = null;
      this.len -= 1;
    }
  }

  PushFront(value: Entry) {
    const node = new ListNode(value, null, this.root);
    this.root = node;
    this.len += 1;
    return node;
  }
}
```

LRU 实现：

```typescript
import { List, ListNode, Entry } from './list';
import sizeof from 'object-sizeof';

export class LruCache {
  maxBytes: number; // 最大内存
  nbytes: number; // 已使用的内存
  ll: List; // 双向链表
  cache: { [propsName: string]: ListNode };
  onEvicted: (key: string, value: any) => void | null; // 清除条目时执行的函数

  constructor(maxBytes: number, onEvicted?: (key: string, value: any) => void) {
    this.maxBytes = maxBytes;
    this.ll = new List();
    this.cache = {};
    this.onEvicted = onEvicted || null;
    this.nbytes = 0;
  }

  // 查询
  Get(key: string) {
    const ele = this.cache[key];
    if (ele) {
      // 如果键对应的链表节点存在，则将对应节点移动到队尾，并返回查找到的值。
      this.ll.MoveToFront(ele);
      const kv = ele.value;
      return kv;
    }
    return null;
  }

  // 缓存淘汰
  RemoveOldest() {
    const ele = this.ll.Back();
    if (ele) {
      // 移除队列首结点
      this.ll.Remove(ele);
      const kv = ele.value;
      delete this.cache[kv.key];
      this.nbytes -= sizeof(kv);
      // 调用回调
      if (this.onEvicted !== null) {
        this.onEvicted(kv.key, kv.value);
      }
    }
  }

  // 新增/更新
  Add(key: string, value: any) {
    let ele = this.cache[key];
    if (ele) {
      // 更新
      this.ll.MoveToFront(ele);
      const kv = ele.value;
      this.nbytes += sizeof(value) - sizeof(kv.value);
      kv.value = value;
    } else {
      // 新建
      const entry = new Entry(key, value);
      ele = this.ll.PushFront(entry);
      this.cache[key] = ele;
      this.nbytes += sizeof(entry);
    }

    // 超出最大值则清除最少访问的节点
    while (this.maxBytes !== 0 && this.maxBytes < this.nbytes) {
      this.RemoveOldest();
    }
  }

  // 查询数据数量
  Len() {
    return this.ll.len;
  }
}
```

测试：

```typescript
import test from 'ava';
import { List, ListNode, Entry } from './list';
import { LruCache } from './lru';
import sizeof from 'object-sizeof';

test('test cache get', t => {
  const lru = new LruCache(0);
  lru.Add('key1', '1234');

  const value1 = lru.Get('key1').value;
  t.is(value1, '1234');

  const value2 = lru.Get('key2');
  t.is(value2, null);

  t.pass();
});

test('test cache Removeoldest', t => {
  const [k1, k2, k3] = ['key1', 'key2', 'k3'];
  const [v1, v2, v3] = ['value1', 'value2', 'v3'];

  const cap = sizeof(new Entry(k1, v1)) + sizeof(new Entry(k2, v2));
  const lru = new LruCache(cap);
  lru.Add(k1, v1);
  lru.Add(k2, v2);
  lru.Add(k3, v3);

  const value1 = lru.Get('key1');
  t.is(value1, null);

  t.pass();
});

test('test cache onEvicted', t => {
  const keys: string[] = [];
  const callback = (key: string, value: any) => {
    keys.push(key);
  };

  const [k1, k2, k3] = ['key1', 'key2', 'k3'];
  const [v1, v2, v3] = ['value1', 'value2', 'v3'];

  const cap = sizeof(new Entry(k1, v1)) + sizeof(new Entry(k2, v2));
  const lru = new LruCache(cap, callback);
  lru.Add(k1, v1);
  lru.Add(k2, v2);
  lru.Add(k3, v3);

  t.deepEqual(keys, ['key1']);

  t.pass();
});

test('test cache add', t => {
  const [k1, k2, k3] = ['key1', 'key2', 'k3'];
  const [v1, v2, v3] = ['value1', 'value2', 'v3'];
  const size = sizeof(new Entry(k1, v1)) + sizeof(new Entry(k2, v2)) + sizeof(new Entry(k3, v3));
  const lru = new LruCache(0);
  lru.Add(k1, v1);
  lru.Add(k2, v2);
  lru.Add(k3, v3);

  t.deepEqual(lru.nbytes, size);

  t.pass();
});
```

# Cache: 并发缓存

我们已经实现了对缓存的 CRUD，但是，多线程高并发的时候，会发生冲突，在 GO 中，我们需要确保只有一个协程可以访问，这时候需要使用锁来解决。

![cache](/image/groupcache/cache.jpg)

```typescript
import AsyncLock from 'async-lock';
import clone from 'rfdc';
import { LruCache } from './lru/lru';

const CACHENAMEMAP: { [propsName: string]: Cache } = {};

const GLOBALLOCK = new AsyncLock();

export class Cache {
  name: string;
  mu: AsyncLock;
  lru: LruCache;
  cacheBytes: number;
  constructor(name: string, cacheBytes: number) {
    name = `__CACHE__${name}`;
    this.name = name;
    this.mu = new AsyncLock();
    this.cacheBytes = cacheBytes;
    CACHENAMEMAP[name] = this;
  }

  async add(key: string, value: any) {
    try {
      await this.mu.acquire(this.name, () => {
        if (!this.lru) {
          this.lru = new LruCache(this.cacheBytes);
        }
        this.lru.Add(key, clone({ proto: true })(value));
      });
    } catch (e) {
      console.log(e.toString());
    }
  }

  async get(key: string) {
    try {
      const result = await this.mu.acquire(this.name, () => {
        if (!this.lru) {
          return null;
        }

        return clone({ proto: true })(this.lru.Get(key));
      });
      return result;
    } catch (e) {
      console.log(e.toString());
    }
  }
}

export const newCache = async (name: string, cacheBytes: number) => {
  try {
    const cache = await GLOBALLOCK.acquire('___CACHE___NEW___', () => {
      const mapName = `__CACHE__${name}`;
      if (CACHENAMEMAP[mapName]) {
        throw Error('this name is exist');
      }
      return new Cache(name, cacheBytes);
    });
    return cache;
  } catch (e) {
    console.log(e.toString());
    return null;
  }
};
```

# Group: 主体结构

Group 是整个项目最核心的主体结构，负责与用户的交互，并且控制缓存值存储和获取的流程。具体处理的流程如下：

![group](/image/groupcache/group.jpg)

实现 Group：

```typescript
import AsyncLock from 'async-lock';
import { Cache, newCache } from './cache';
import { Entry } from './lru/list';

const AKTCACHENAMEMAP: { [propsName: string]: Group } = {};

const GLOBALLOCK = new AsyncLock();

export interface Getter {
  Get(key: string): any | Promise<any> | null;
}

class Group {
  name: string;
  getter: Getter;
  mainCache: Cache;
  constructor(name: string, getters: Getter, mainCache: Cache) {
    name = `__AKTCACHE__${name}`;
    this.name = name;
    this.getter = getters;
    this.mainCache = mainCache;
    AKTCACHENAMEMAP[name] = this;
  }

  async get(key: string): Promise<any | null> {
    if (!key) {
      return null;
    }
    try {
      const v = await this.mainCache.get(key);
      if (v !== null) {
        return v.value;
      }
      const loadv = await this.load(key);
      return loadv ? loadv.value : null;
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }

  async load(key: string): Promise<Entry | null> {
    try {
      // TODO
      return await this.getLocall(key);
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }

  async getLocall(key: string): Promise<Entry | null> {
    try {
      const value = await this.getter.Get(key);

      if (!value) {
        return null;
      }

      await this.populateCache(key, value);

      return new Entry(key, value);
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }

  private async populateCache(key: string, v: any) {
    try {
      await this.mainCache.add(key, v);
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }
}

export const newGroup = async (name: string, getters: Getter, cacheBytes: number) => {
  try {
    const group = await GLOBALLOCK.acquire('___AKTCGROUP___NEW___', async () => {
      const mapName = `__AKTCACHE__${name}`;
      if (AKTCACHENAMEMAP[mapName]) {
        throw Error('this name is exist');
      }
      const cache = await newCache(`__GROUP__CREATE__${name}`, cacheBytes);
      if (!cache) {
        throw Error('create cache failed');
      }
      return new Group(name, getters, cache);
    });
    return group;
  } catch (e) {
    console.log(e.toString());
    return null;
  }
};

export const getGroup = async (name: string) => {
  try {
    const group = GLOBALLOCK.acquire('___AKTCGROUP___GET___', () => {
      const mapName = `__AKTCACHE__${name}`;
      return AKTCACHENAMEMAP[mapName] || null;
    });
    return group;
  } catch (e) {
    console.log(e.toString());
    return null;
  }
};
```

测试：

```typescript
import test from 'ava';
import { newGroup, Getter, getGroup } from './aktcache';

const db = {
  akt: '123',
  akita: '321',
  akitasummer: '213',
};

test('test AktCache get', async t => {
  const loadCounts: { [propsName: string]: number } = {};

  const getter: Getter = {
    Get(key: string) {
      const v = db[key];
      if (v) {
        const counts = loadCounts[key];
        if (!counts) {
          loadCounts[key] = 0;
        }
        loadCounts[key] += 1;
        return v;
      }

      return null;
    },
  };

  const akt = await newGroup('num', getter, 2048);

  const keys = Object.keys(db);

  for (let i = 0; i < keys.length; i++) {
    const key = keys[i];
    const v = await akt.get(key);
    t.is(v, db[key]);
    await akt.get(key);
    t.false(loadCounts[key] > 1);
  }

  const v = await akt.get('test');

  t.is(v, null);

  t.pass();
});

test('test AktCache getGroup', async t => {
  const groupName = 'num1';

  const getter: Getter = {
    Get(key: string) {
      return null;
    },
  };

  const akt = await newGroup(groupName, getter, 2048);

  const group = await getGroup(groupName);

  t.deepEqual(group, akt);

  const notGroup = await getGroup('1234');

  t.is(notGroup, null);

  t.pass();
});
```

这样，一个单机并发缓存已经完成了，现在我们已经构建的情况是这样的了：

![group_v2](/image/groupcache/group_v2.jpg)

# Server：HTTP 服务端

由于我们分布式缓存需要实现节点间通信，使用 HTTP 进行通信是较为常见的方式，接下来我们实现 Group 中的 server 部分，用来被其他节点访问：

```typescript
import express, { Express, Request, Response, NextFunction } from 'express';
import axios from 'axios';
import AsyncLock from 'async-lock';
import { ConsistentHashMap } from './consistenthash/consistenthash';
import { getGroup } from './aktcache';

const defaultBasePath = '/_aktcache/';
const defaultReplicas = 50;

const HTTPLOCK = new AsyncLock();

export class HTTP {
  self: string; // 记录自己的ip
  port: number; // 记录自己的ip/端口
  basePath: string; // 节点间通讯地址的前缀
  server: Express;

  constructor(self: string, port: number, callback?: () => void) {
    this.self = self;
    this.port = port;
    this.basePath = defaultBasePath;
    this.server = express();

    const router = express.Router();

    router.get(`${this.basePath}*`, async (request: Request, response: Response) => {
      console.log(
        `[Server ${this.self + ':' + this.port}] method: ${request.method}, path: ${request.path}`,
      );

      const parts = request.path.replace(this.basePath, '').split('/');

      if (parts.length !== 2) {
        return response.status(400).send({
          message: 'path error',
        });
      }

      const groupName = parts[0];
      const key = parts[1];

      try {
        const group = await getGroup(groupName);

        if (group === null) {
          return response.status(404).send({
            message: `no such group: ${groupName}`,
          });
        }

        const value = await group.get(key);

        if (value === null) {
          return response.status(500).send({
            message: `server error`,
          });
        }

        response.setHeader('Content-Type', 'application/octet-stream');
        response.status(200).send({
          data: value,
        });
      } catch (e) {
        console.log(e);
        return response.status(500).send({
          message: `server error`,
        });
      }
    });

    this.server.use(express.json());

    this.server.use('/', router);

    this.server.listen(this.port, () => {
      console.log(`server run in ${this.self + ':' + this.port}`);
      callback && callback();
    });
  }
}
```

这样，我们供其他节点访问的的 HTTPserver 就完成了。

# ConsistentHash: 一致性哈希值

当一个节点收到请求后，如果他没有缓存值，他需要知道去哪里获取新的数据，hash 算法可以做到对于给定的 key，返回一个特定的节点值，让其访问。

当节点数发生变化时，可能会导致缓存值对应的节点都发生了改变。即几乎所有的缓存值都失效，需要重新去数据源获取数据，容易引起缓存雪崩。

> 缓存雪崩：缓存在同一时刻全部失效，造成瞬时 DB 请求量大、压力骤增，引起雪崩。常因为缓存服务器宕机，或缓存设置了相同的过期时间引起。

groupcache 中选用了一致性哈希算法解决了这个问题，它将 key 映射到 2^32 的空间中，将这个数字首尾相连，形成一个环，通过计算节点的哈希值，放置在环上。当我们使用 key 获取 value 时，计算 key 的哈希值，顺时针寻找到的第一个节点，就是应选取的节点。

![ConsistentHash](/image/groupcache/b9d37767-7df2-482a-ac46-588dd67b6685.png)

但是当节点数过少时，会引起 key 的倾斜，导致节点间负载不均。groupcache 中引入了虚拟节点的概念，一个真实节点对应多个虚拟节点。并且使用一个 map 维护真实节点与虚拟节点的映射关系，通过这样增加虚拟节点，减少 key 的倾斜。

接下来我们实现一致性哈希：

```typescript
import { crc32 } from 'node-crc';

export type Hash = (data: any) => string;

export class ConsistentHashMap {
  hash: Hash;
  replicas: number;
  keys: string[];
  hashMap: Map<string, string>;

  constructor(replicas: number, fn?: Hash) {
    this.replicas = replicas;
    this.hashMap = new Map<string, string>();
    this.keys = [];

    if (!fn) {
      fn = (data: any) => crc32(Buffer.from(data, 'utf8')).toString('hex');
    }

    this.hash = fn;
  }

  Add(...keys: string[]) {
    for (let i = 0; i < keys.length; i++) {
      const key = keys[i];
      for (let j = 0; j < this.replicas; j++) {
        const hash = this.hash(j + key);
        this.keys.push(hash);
        this.hashMap[hash] = key;
      }
    }
    this.keys = this.keys.sort();
  }

  Get(key: string) {
    if (this.keys.length === 0) {
      return null;
    }

    const hash = this.hash(key);

    let idx = this.keys.findIndex(item => {
      return BigInt(item) >= BigInt(hash);
    });

    if (idx === -1) {
      idx = 0;
    }

    return this.hashMap[this.keys[Number(BigInt(idx) % BigInt(this.keys.length))]];
  }
}
```

测试：

```typescript
import test from 'ava';
import { ConsistentHashMap } from './consistenthash';

test('test hash', async t => {
  const hash = new ConsistentHashMap(3, (key: any) => key.toString());

  hash.Add('6', '4', '2');

  const testCases = {
    '2': '2',
    '11': '2',
    '23': '4',
    '27': '2',
  };

  const keys = Object.keys(testCases);
  for (let i = 0; i < keys.length; i++) {
    t.is(hash.Get(keys[i]), testCases[keys[i]]);
  }

  hash.Add('8');

  testCases['27'] = '8';

  for (let i = 0; i < keys.length; i++) {
    t.is(hash.Get(keys[i]), testCases[keys[i]]);
  }

  t.pass();
});
```

# 将 Server 结合至 Group

现在我们 server 已经存在了，获取节点地址的 ConsistentHash 也有了，Group 也做好了，我们只需要将这 3 者组合起来，就可以完成第 3 章中未完成的流程了。

首先我们先抽象出请求其他节点的 HTTP 客户端 PeerGetter 和 Server 提供访问其他节点的 HTTP 客户端 PeerPicker：

```typescript
export interface PeerPicker {
  pickPeer(key: string): Promise<PeerGetter> | null;
}

export interface PeerGetter {
  get(Group: string, key: string): any | Promise<any> | null;
}
```

然后我们为我们的 http 实现：

```typescript
import express, { Express, Request, Response, NextFunction } from 'express';
import axios from 'axios';
import AsyncLock from 'async-lock';
import { ConsistentHashMap } from './consistenthash/consistenthash';
import { getGroup } from './aktcache';
import { PeerGetter, PeerPicker } from './peers';

const defaultBasePath = '/_aktcache/';

// 新增
const defaultReplicas = 50;

const HTTPLOCK = new AsyncLock();

export class HTTP implements PeerPicker {
  self: string; // 记录自己的ip
  port: number; // 记录自己的ip/端口
  basePath: string; // 节点间通讯地址的前缀
  server: Express;
  peers: ConsistentHashMap;
  httpGetters: Map<string, HttpGetter>;

  constructor(self: string, port: number, callback?: () => void) {
    this.self = self;
    this.port = port;
    this.basePath = defaultBasePath;
    this.server = express();

    const router = express.Router();

    router.get(`${this.basePath}*`, async (request: Request, response: Response) => {
      console.log(
        `[Server ${this.self + ':' + this.port}] method: ${request.method}, path: ${request.path}`,
      );

      const parts = request.path.replace(this.basePath, '').split('/');

      if (parts.length !== 2) {
        return response.status(400).send({
          message: 'path error',
        });
      }

      const groupName = parts[0];
      const key = parts[1];

      try {
        const group = await getGroup(groupName);

        if (group === null) {
          return response.status(404).send({
            message: `no such group: ${groupName}`,
          });
        }

        const value = await group.get(key);

        if (value === null) {
          return response.status(500).send({
            message: `server error`,
          });
        }

        response.setHeader('Content-Type', 'application/octet-stream');
        response.status(200).send({
          data: value,
        });
      } catch (e) {
        console.log(e);
        return response.status(500).send({
          message: `server error`,
        });
      }
    });

    this.server.use(express.json());

    this.server.use('/', router);

    this.server.listen(this.port, () => {
      console.log(`server run in ${this.self + ':' + this.port}`);
      callback && callback();
    });
  }

  // 新增
  async set(...peers: string[]) {
    try {
      await HTTPLOCK.acquire(`___HTTP___${this.self}___`, () => {
        this.peers = new ConsistentHashMap(defaultReplicas, null);
        this.peers.Add(...peers);
        this.httpGetters = new Map<string, HttpGetter>();
        peers.forEach(peer => {
          this.httpGetters.set(peer, new HttpGetter(peer + this.basePath));
        });
      });
    } catch (err) {
      console.log(err);
      throw err;
    }
  }

  // 新增
  async pickPeer(key: string) {
    try {
      const result: HttpGetter | null = await HTTPLOCK.acquire(
        `___HTTP___${this.self}___`,
        async () => {
          // 获取真实结点
          const peer = this.peers.Get(key);
          if (peer !== '' && peer !== this.self) {
            return this.httpGetters[peer];
          }
          return null;
        },
      );
      return result;
    } catch (err) {
      console.log(err);
      throw err;
    }
  }
}

// 新增
class HttpGetter implements PeerGetter {
  baseUrl: string;
  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  async get(group: string, key: string) {
    const url = `${this.baseUrl}${group}/${key}`;
    console.log(`GET: ${url}`);
    const res = await axios({
      url,
      method: 'GET',
    });

    if (res.status !== 200) {
      return null;
    }

    return res.data.data;
  }
}
```

然后我们只需要将我们的 Server 挂载到 Group 上：

```typescript
import AsyncLock from 'async-lock';
import { Cache, newCache } from './cache';
import { Entry } from './lru/list';
import { PeerGetter, PeerPicker } from './peers';

const AKTCACHENAMEMAP: { [propsName: string]: Group } = {};

const GLOBALLOCK = new AsyncLock();

export interface Getter {
  Get(key: string): any | Promise<any> | null;
}

class Group {
  name: string;
  getter: Getter;
  mainCache: Cache;
  peers: PeerPicker;
  constructor(name: string, getters: Getter, mainCache: Cache) {
    name = `__AKTCACHE__${name}`;
    this.name = name;
    this.getter = getters;
    this.mainCache = mainCache;
    AKTCACHENAMEMAP[name] = this;
  }

  // 新增
  registerPeers(peers: PeerPicker) {
    if (!!this.peers) {
      throw new Error('egisterPeerPicker called more than once');
    }
    this.peers = peers;
  }

  async get(key: string): Promise<any | null> {
    if (!key) {
      return null;
    }
    try {
      const v = await this.mainCache.get(key);
      if (v !== null) {
        return v.value;
      }
      const loadv = await this.load(key);
      return loadv ? loadv.value : null;
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }

  async load(key: string): Promise<Entry | null> {
    try {
      // 新增
      if (!this.peers) {
        const peer = await this.peers.pickPeer(key);
        if (peer) {
          const value: Entry | null = await this.getFromPeer(peer, key);
          if (value) {
            return value;
          }
        }
      }
      return await this.getLocall(key);
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }
  // 新增
  async getFromPeer(peer: PeerGetter, key: string) {
    const value = await peer.get(this.name, key);
    if (!value) {
      return null;
    }
    return new Entry(key, value);
  }

  async getLocall(key: string): Promise<Entry | null> {
    try {
      const value = await this.getter.Get(key);

      if (!value) {
        return null;
      }

      await this.populateCache(key, value);

      return new Entry(key, value);
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }

  private async populateCache(key: string, v: any) {
    try {
      await this.mainCache.add(key, v);
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }
}

export const newGroup = async (name: string, getters: Getter, cacheBytes: number) => {
  try {
    const group = await GLOBALLOCK.acquire('___AKTCGROUP___NEW___', async () => {
      const mapName = `__AKTCACHE__${name}`;
      if (AKTCACHENAMEMAP[mapName]) {
        throw Error('this name is exist');
      }
      const cache = await newCache(`__GROUP__CREATE__${name}`, cacheBytes);
      if (!cache) {
        throw Error('create cache failed');
      }
      return new Group(name, getters, cache);
    });
    return group;
  } catch (e) {
    console.log(e.toString());
    return null;
  }
};

export const getGroup = async (name: string) => {
  try {
    const group = GLOBALLOCK.acquire('___AKTCGROUP___GET___', () => {
      const mapName = `__AKTCACHE__${name}`;
      return AKTCACHENAMEMAP[mapName] || null;
    });
    return group;
  } catch (e) {
    console.log(e.toString());
    return null;
  }
};
```

现在我们已经实现了一个完整的 Group 了！

![server_with_group](/image/groupcache/server_with_group.jpg)

接下来我们通过创建一个 client 和多个 Group 来完成最开头的架构吧！

![mult_group](/image/groupcache/mult_group.jpg)

测试 server:

```typescript
import { HTTP } from './src/http';
import { newGroup, Getter, Group } from './src/aktcache';
import express, { Express, Request, Response, NextFunction } from 'express';
import { spawn } from 'child_process';

const db = {
  akt: '123',
  akita: '321',
  akitasummer: '213',
};

const createGroup = async () => {
  const getter: Getter = {
    Get(key: string) {
      const v = db[key];
      if (v) {
        return v;
      }

      return null;
    },
  };

  return await newGroup('num', getter, 2048);
};

const startCacheServer = async (addr: string, addrs: string[], akt: Group) => {
  let peers: HTTP;
  try {
    await new Promise(resolve => {
      peers = new HTTP('localhost', Number(addr), () => {
        console.log(`aktcache is running at ${addr}`);
        resolve(null);
      });
    });
    await peers.set(...addrs);
    akt.registerPeers(peers);
  } catch (err) {
    throw err;
  }
};

const startAPIServer = async (apiAddr: string, akt: Group) => {
  const server = express();
  const router = express.Router();
  router.get('/api', async (request: Request, response: Response) => {
    try {
      const key = request.query.key;
      const value = await akt.get(key.toString());
      if (!value) {
        return response.status(500).send({
          message: `server error`,
        });
      }
      response.setHeader('Content-Type', 'application/octet-stream');
      response.status(200).send({
        data: value,
      });
    } catch (e) {
      console.log(e);
      return response.status(500).send({
        message: `server error`,
      });
    }
  });

  server.use(express.json());

  server.use('/', router);

  server.listen(apiAddr, () => {
    console.log(`server run in ${apiAddr}`);
  });
};

const apiAddr = '9999';

const addrMap = {
  '8001': 'http://localhost:8001',
  '8002': 'http://localhost:8002',
  '8003': 'http://localhost:8003',
};

(async (apiAddr: string, addrMap: { [propsName: string]: string }) => {
  const addrs = Object.keys(addrMap);
  const akt = await createGroup();

  addrs.forEach(addr => {
    spawn('node', ['-port', addr]);
  });

  startCacheServer(apiAddr, addrs, akt);
})(apiAddr, addrMap);
```

测试 client:

```typescript
import { argv } from 'process';
import { HTTP } from './src/http';
import { newGroup, Getter, Group } from './src/aktcache';
import express, { Express, Request, Response, NextFunction } from 'express';

const db = {
  akt: '123',
  akita: '321',
  akitasummer: '213',
};

const createGroup = async () => {
  const getter: Getter = {
    Get(key: string) {
      const v = db[key];
      if (v) {
        return v;
      }

      return null;
    },
  };

  return await newGroup('num', getter, 2048);
};

const startAPIServer = async (apiAddr: string, akt: Group) => {
  const server = express();
  const router = express.Router();
  router.get('/api', async (request: Request, response: Response) => {
    try {
      const key = request.query.key;
      const value = await akt.get(key.toString());
      if (!value) {
        return response.status(500).send({
          message: `server error`,
        });
      }
      response.setHeader('Content-Type', 'application/octet-stream');
      response.status(200).send({
        data: value,
      });
    } catch (e) {
      console.log(e);
      return response.status(500).send({
        message: `server error`,
      });
    }
  });
};

const post = argv[3];

(async (post: string) => {
  const akt = await createGroup();
  startAPIServer(post, akt);
})(post);
```

# SingleFlight: 防止缓存击穿

> 缓存雪崩：缓存在同一时刻全部失效，造成瞬时 DB 请求量大、压力骤增，引起雪崩。缓存雪崩通常因为缓存服务器宕机、缓存的 key 设置了相同的过期时间等引起。

> 缓存击穿：一个存在的 key，在缓存过期的一刻，同时有大量的请求，这些请求都会击穿到 DB ，造成瞬时 DB 请求量大、压力骤增。

> 缓存穿透：查询一个不存在的数据，因为不存在则不会写到缓存中，所以每次都会去请求 DB，如果瞬间流量过大，穿透到 DB，导致宕机。

现在我们的项目已经基本完成了，但是我们需要考虑缓存击穿的问题，现在我们的 server 会将每次请求都发送：

![mult_req](/image/groupcache/mult_req.jpg)

这时候我们需要加个锁，当一次请求完成后，将所有相同 key 的请求都返回：

![single_req](/image/groupcache/single_req.jpg)

我们首先来实现 singleflight 部分：

```typescript
import AsyncLock from 'async-lock';

class Call {
  wg: AsyncLock;
  val: any;
  name: string;
  constructor(key: string) {
    key = `__CALL__${key}`;
    this.name = key;
    this.wg = new AsyncLock();
  }
}

export class SingleFlight {
  mu: AsyncLock;
  name: string;
  map: Map<string, Call>;
  constructor(name: string) {
    name = `__CALL__${name}`;
    this.name = name;
    this.mu = new AsyncLock();
    this.map = new Map<string, Call>();
  }

  async Do(key: string, fn: () => Promise<any | null>) {
    try {
      // 确保只有唯一一个能修改map
      const call: Call = await this.mu.acquire(this.name, async () => {
        if (this.map.get(key)) {
          return this.map.get(key); // 如果请求正在进行中，则等待
        }
        const call = new Call(key);
        this.map[key] = call; // 表明 key 已经有对应的请求在处理
        return call;
      });

      const result = await call.wg.acquire(call.name, async () => {
        try {
          if (!call.val) {
            // 没有值，调用fn
            const val = await fn();
            call.val = val;
            return val;
          } else {
            return call.val; // 存在值，获取
          }
        } catch (err) {
          throw err;
        }
      });
      this.map.delete(key); // 更新map
      return result;
    } catch (err) {
      throw err;
    }
  }
}
```

集成：

```typescript
import AsyncLock from 'async-lock';
import { Cache, newCache } from './cache';
import { Entry } from './lru/list';
import { PeerGetter, PeerPicker } from './peers';
import { SingleFlight } from './singleflight/singleflight';

const AKTCACHENAMEMAP: { [propsName: string]: Group } = {};

const GLOBALLOCK = new AsyncLock();

export interface Getter {
  Get(key: string): any | Promise<any> | null;
}

export class Group {
  name: string;
  getter: Getter;
  mainCache: Cache;
  peers: PeerPicker;
  // 新增
  loader: SingleFlight;
  constructor(name: string, getters: Getter, mainCache: Cache) {
    name = `__AKTCACHE__${name}`;
    this.name = name;
    this.getter = getters;
    this.mainCache = mainCache;
    // 新增
    this.loader = new SingleFlight(name);
    AKTCACHENAMEMAP[name] = this;
  }

  registerPeers(peers: PeerPicker) {
    if (!!this.peers) {
      throw new Error('egisterPeerPicker called more than once');
    }
    this.peers = peers;
  }

  async get(key: string): Promise<any | null> {
    if (!key) {
      return null;
    }
    try {
      const v = await this.mainCache.get(key);
      if (v !== null) {
        return v.value;
      }
      const loadv = await this.load(key);
      return loadv ? loadv.value : null;
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }

  async load(key: string): Promise<Entry | null> {
    try {
      // 新增
      return await this.loader.Do(key, async () => {
        if (!this.peers) {
          const peer = await this.peers.pickPeer(key);
          if (peer) {
            const value: Entry | null = await this.getFromPeer(peer, key);
            if (value) {
              return value;
            }
          }
        }
        return await this.getLocall(key);
      });
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }

  async getFromPeer(peer: PeerGetter, key: string) {
    const value = await peer.get(this.name, key);
    if (!value) {
      return null;
    }
    return new Entry(key, value);
  }

  async getLocall(key: string): Promise<Entry | null> {
    try {
      const value = await this.getter.Get(key);

      if (!value) {
        return null;
      }

      await this.populateCache(key, value);

      return new Entry(key, value);
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }

  private async populateCache(key: string, v: any) {
    try {
      await this.mainCache.add(key, v);
    } catch (e) {
      console.log(e.toString());
      return null;
    }
  }
}

export const newGroup = async (name: string, getters: Getter, cacheBytes: number) => {
  try {
    const group = await GLOBALLOCK.acquire('___AKTCGROUP___NEW___', async () => {
      const mapName = `__AKTCACHE__${name}`;
      if (AKTCACHENAMEMAP[mapName]) {
        throw Error('this name is exist');
      }
      const cache = await newCache(`__GROUP__CREATE__${name}`, cacheBytes);
      if (!cache) {
        throw Error('create cache failed');
      }
      return new Group(name, getters, cache);
    });
    return group;
  } catch (e) {
    console.log(e.toString());
    return null;
  }
};

export const getGroup = async (name: string) => {
  try {
    const group = GLOBALLOCK.acquire('___AKTCGROUP___GET___', () => {
      const mapName = `__AKTCACHE__${name}`;
      return AKTCACHENAMEMAP[mapName] || null;
    });
    return group;
  } catch (e) {
    console.log(e.toString());
    return null;
  }
};
```

# 写在最后

非常感谢你能阅读到这里，如果你能从我这个实现 demo 中学到一些知识和技术，那我是非常开心的。由于我的语文并不是很好，导致文章中存在大量的代码，而解释性文字比较少，如果你有任何不懂的部分，可以评论或者私聊联系我，我会尽我所能的帮助你解决你的困惑。
