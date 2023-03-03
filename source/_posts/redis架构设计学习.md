---
title: redis架构设计学习
date: 2023-01-28 01:23:28
tags:
  - redis
  - golang
categories: 学习笔记
---

# 前言

redis 是后端常用的缓存数据库，他是一种基于内存的数据库，对数据的读写操作都是在内存中完成，因此读写速度非常快，常用于缓存，消息队列、分布式锁等场景。redis 设计是单线程的，但是却能做到高性能完成读写，学习其中的架构设计，对于我们同是单线程的 JavaScript 程序设计会有一定的帮助

# 整体流程总览

本文参考的是[redis2.6 版本](https://github.com/redis/redis/tree/2.6)的架构设计

流程图：：
![architecture](/image/go-redis/architecture.png)

简单来说，redis 在启动时，会读取 config 文件，然后创建 redis server 和 aeloop 对象，并在 aeloop 中注册 AcceptHandler 这个 FileEvent，随后启动 aeloop，在 aeloop 中不断的循环处理 FileEvent 和 TimeEvent 事件。

AcceptHandler 事件为接受请求，当 redis server 接收到请求时，会创建 redis client 对象保存其句柄并注册 ReadQueryFromClient，然后在下一次循环中调用 ReadQueryFromClient。

ReadQueryFromClient 作用是解析 client 传递来的数据，并执行，将执行的结果存放至 redis client 对象中的 reply 属性中，并注册 SendReplyToClient 事件，然后在下一次循环中调用 SendReplyToClient 事件

SendReplyToClient 事件为向 client 发送 server 处理结果，在完毕后，会主动注销与此 client 有关的所有 FileEvent 事件，这样就完成了一个流程的调用。

当然，这仅仅只是其一套流程，其中数据储存的 redis object 的设计，aeloop 的实现，Dist 在单线程下的分段式 rehash 等等，都会在下面具体来讲一下。

阅读源码的时候，顺手用 go 编写了一个 mini 版的 redis，源码地址是：
https://github.com/akitaSummer/akt-redis/tree/main/go，
配合这个项目，应该能够更加快速的入门。

# 基础模块

在实现 redis server 这个对象之前，我们先需要把项目基础所需要的 redis obj，dict，List， net 和 aeloop 这几个基础组件实现，其中各自的用处我都会解释。

## redis object

Redis obj 的作用是数据在内存中的最终存储形式，由于原来的项目是用 c 编写的，redis 使用了引用计数的形式，当其引用为 0 时，会进行析构释放内存，redis obj 的定义如下

```golang
type Gtype uint8

type Gval interface{}

const (
        // string
        GSTR Gtype = 0x00
        // list
        GLIST Gtype = 0x01
        // set
        GSET Gtype = 0x02
        // zset
        GZSET Gtype = 0x03
        // dist
        GDICT Gtype = 0x04
)

type Gobj struct {
        Type_    Gtype // 数据类型
        Val_     Gval // 值
        RefCount int // 计数
}
创建析构等实现如下：
package obj

import "strconv"

type Gtype uint8

type Gval interface{}

const (
        // string
        GSTR Gtype = 0x00
        // list
        GLIST Gtype = 0x01
        // set
        GSET Gtype = 0x02
        // zset
        GZSET Gtype = 0x03
        // dist
        GDICT Gtype = 0x04
)

type Gobj struct {
        Type_    Gtype
        Val_     Gval
        RefCount int
}

func (obj *Gobj) IntVal() int64 {
        if obj.Type_ != GSTR {
                return 0
        }
        val, _ := strconv.ParseInt(obj.Val_.(string), 10, 64)
        return val
}

func (o *Gobj) StrVal() string {
        if o.Type_ != GSTR {
                return ""
        }
        return o.Val_.(string)
}

func CreateFromInt(val int64) *Gobj {
        return &Gobj{
                Type_:    GSTR,
                Val_:     strconv.FormatInt(val, 10),
                RefCount: 1,
        }
}

func CreateObject(typ Gtype, ptr interface{}) *Gobj {
        return &Gobj{
                Type_:    typ,
                Val_:     ptr,
                RefCount: 1,
        }
}

func (o *Gobj) IncrRefCount() {
        o.RefCount++
}

func (o *Gobj) DecrRefCount() {
        o.RefCount--
        if o.RefCount == 0 {
                // 垃圾回收自动回收Val_
                o.Val_ = nil
        }
}
```

## List

list 是一个双向链表，在 aeloop 中被用于存储 FileEvent 和 TimeEvent，每次 loop 的时候 aeloop 都会去遍历这两个 list，双向链表没什么难得，实现如下：

```golang
package list

import "akt-redis/obj"

// 双向链表

type Node struct {
        Val  *obj.Gobj
        next *Node
        prev *Node
}

type ListType struct {
        EqualFunc func(a, b *obj.Gobj) bool
}

type List struct {
        ListType
        Head   *Node
        Tail   *Node
        length int
}

func ListCreate(listType ListType) *List {
        return &List{
                ListType: listType,
        }
}

func (list *List) Append(val *obj.Gobj) {
        node := Node{
                Val: val,
        }
        if list.Head == nil {
                list.Head = &node
                list.Tail = &node
        } else {
                node.prev = list.Tail
                list.Tail.next = &node
                list.Tail = &node
        }
        list.length += 1
}

func (list *List) DelNode(n *Node) {
        if n == nil {
                return
        }
        if n == list.Head {
                if n.next != nil {
                        n.next.prev = nil
                }
                list.Head = n.next
                n.next = nil
        } else if n == list.Tail {
                if n.prev != nil {
                        n.prev.next = nil
                }
                list.Tail = n.prev
                n.prev = nil
        } else {
                if n.prev != nil {
                        n.prev.next = n.next
                }
                if n.next != nil {
                        n.next.prev = n.prev
                }
                n.prev = nil
                n.next = nil
        }
        list.length -= 1
}

func (list *List) Length() int {
        return list.length
}

func (list *List) First() *Node {
        return list.Head
}

func (list *List) Last() *Node {
        return list.Tail
}

func (list *List) Delete(val *obj.Gobj) {
        list.DelNode(list.Find(val))
}

func (list *List) Find(val *obj.Gobj) *Node {
        p := list.Head
        for p != nil {
                if list.EqualFunc(p.Val, val) {
                        return p
                }
                p = p.next
        }
        return nil
}

func (list *List) LPush(val *obj.Gobj) {
        node := Node{
                Val:  val,
                next: list.Head,
                prev: nil,
        }
        if list.Head != nil {
                list.Head.prev = &node
        } else {
                list.Tail = &node

        }

        list.Head = &node
        list.length += 1
}
```

## Dict

dict 是用于 redis 底层 k-v 存储的字典

```golang
type Entry struct {
        Key  *obj.Gobj
        Val  *obj.Gobj
        next *Entry
}

type htable struct {
        table []*Entry
        size  int64
        mask  int64
        used  int64
}

type DictType struct {
        HashFunc  func(key *obj.Gobj) int64
        EqualFunc func(k1 *obj.Gobj, k2 *obj.Gobj) bool
}

type Dict struct {
        DictType
        hts       [2]*htable // 存储的table
        rehashidx int64
}
```

可以发现，他的 hashtable 最大为两个，主要目的是 rehash，rehash 是指当存储空间不够时，进行扩容。而扩容不是一口气将老的所有值全部移动到新的 table 上，这样会引发 server 卡主，而是渐进式移动，即新的数据直接存储到新的 table 上，老的值分段，多次逐步移动到新 table 上。实现为：

```golang
func (dict *Dict) rehash(step int) {
        // 由于单线程，redis将rehash拆分成为小步骤进行rehash，以保证不会因为table过大而引发block线程
        for step > 0 {
                // 已经rehash结束时
                if dict.hts[0].used == 0 {
                        dict.hts[0] = dict.hts[1]
                        dict.hts[1] = nil
                        dict.rehashidx = -1
                        return
                }
                // 寻找第一个非空值
                for dict.hts[0].table[dict.rehashidx] == nil {
                        dict.rehashidx++
                }

                // 开始重新遍历非空值下hash，迁移至扩容后的table
                entry := dict.hts[0].table[dict.rehashidx]
                for entry != nil {
                        ne := entry.next
                        idx := dict.HashFunc(entry.Key) & dict.hts[1].mask
                        entry.next = dict.hts[1].table[idx]
                        dict.hts[1].table[idx] = entry
                        dict.hts[0].used -= 1
                        dict.hts[1].used += 1
                        entry = ne
                }
                dict.hts[0].table[dict.rehashidx] = nil
                dict.rehashidx += 1
                step -= 1
        }
}
```

而在增删改查的时候，如果在 rehash，则会两个表均查询，dict 完整实现如下：

```golang
package dict

import (
        "akt-redis/obj"
        "errors"
        "math"
        "math/rand"
)

const (
        INIT_SIZE    int64 = 8
        FORCE_RATIO  int64 = 2 // 强制扩容阈值
        GROW_RATIO   int64 = 2 // 扩容幅度
        DEFAULT_STEP int   = 1
)

var (
        EP_ERR = errors.New("expand error")
        EX_ERR = errors.New("key exists error")
        NK_ERR = errors.New("key doesnt exist error")
)

type Entry struct {
        Key  *obj.Gobj
        Val  *obj.Gobj
        next *Entry
}

type htable struct {
        table []*Entry
        size  int64
        mask  int64
        used  int64
}

type DictType struct {
        HashFunc  func(key *obj.Gobj) int64
        EqualFunc func(k1 *obj.Gobj, k2 *obj.Gobj) bool
}

type Dict struct {
        DictType
        hts       [2]*htable
        rehashidx int64
}

func DictCreate(distType DictType) *Dict {
        return &Dict{
                DictType:  distType,
                rehashidx: -1,
        }
}

func nextPower(size int64) int64 {
        num := INIT_SIZE
        for num < math.MaxInt64 {
                if num >= size {
                        return num
                }
                num *= 2
        }
        return -1
}

func (dict *Dict) expand(size int64) error {
        // 找寻大于等于size的最近2**n
        num := nextPower(size)
        if dict.isRehashing() || (dict.hts[0] != nil && dict.hts[0].size >= num) {
                return EP_ERR
        }
        ht := htable{
                size:  num,
                mask:  num - 1,
                table: make([]*Entry, num),
                used:  0,
        }
        // 初始化时
        if dict.hts[0] == nil {
                dict.hts[0] = &ht
                return nil
        }
        // rehash
        dict.hts[1] = &ht
        dict.rehashidx = 0
        return nil
}

func (dict *Dict) isRehashing() bool {
        return dict.rehashidx != -1
}

func (dict *Dict) expandIfNeeded() error {
        // rehash 中
        if dict.isRehashing() {
                return nil
        }
        // 刚初始化完毕
        if dict.hts[0] == nil {
                return dict.expand(INIT_SIZE) // redis 源码INIT_SIZE为4
        }
        // 当前数目/大小 > 强制扩容阈值，进行扩容
        hashTable := dict.hts[0]
        if (hashTable.used > hashTable.size) && (hashTable.used/hashTable.size > FORCE_RATIO) {
                return dict.expand(hashTable.size * GROW_RATIO)
        }
        return nil
}

func (dict *Dict) rehashStep() {
        dict.rehash(DEFAULT_STEP)
}

func (dict *Dict) rehash(step int) {
        // 由于单线程，redis将rehash拆分成为小步骤进行rehash，以保证不会因为table过大而引发block线程
        for step > 0 {
                // 已经rehash结束时
                if dict.hts[0].used == 0 {
                        dict.hts[0] = dict.hts[1]
                        dict.hts[1] = nil
                        dict.rehashidx = -1
                        return
                }
                // 寻找第一个非空值
                for dict.hts[0].table[dict.rehashidx] == nil {
                        dict.rehashidx++
                }

                // 开始重新遍历非空值下hash，迁移至扩容后的table
                entry := dict.hts[0].table[dict.rehashidx]
                for entry != nil {
                        ne := entry.next
                        idx := dict.HashFunc(entry.Key) & dict.hts[1].mask
                        entry.next = dict.hts[1].table[idx]
                        dict.hts[1].table[idx] = entry
                        dict.hts[0].used -= 1
                        dict.hts[1].used += 1
                        entry = ne
                }
                dict.hts[0].table[dict.rehashidx] = nil
                dict.rehashidx += 1
                step -= 1
        }
}

func freeEntry(e *Entry) {
        e.Key.DecrRefCount()
        e.Val.DecrRefCount()
}

func (dict *Dict) Delete(key *obj.Gobj) error {
        if dict.hts[0] == nil {
                return NK_ERR
        }
        if dict.isRehashing() {
                dict.rehashStep()
        }

        h := dict.HashFunc(key)
        for i := 0; i <= 1; i++ {
                idx := h & dict.hts[i].mask
                e := dict.hts[i].table[idx]
                var prev *Entry
                for e != nil {
                        if dict.EqualFunc(e.Key, key) {
                                if prev == nil {
                                        dict.hts[i].table[idx] = e.next
                                } else {
                                        prev.next = e.next
                                }
                                freeEntry(e)
                                return nil
                        }
                        prev = e
                        e = e.next
                }
                if !dict.isRehashing() {
                        break
                }
        }
        return NK_ERR
}

func (dict *Dict) RandomGet() *Entry {
        if dict.hts[0] == nil {
                return nil
        }
        t := 0
        if dict.isRehashing() {
                dict.rehashStep()
                if dict.hts[1] != nil && dict.hts[1].used > dict.hts[0].used {
                        t = 1
                }
        }
        idx := rand.Int63n(dict.hts[t].size)
        cnt := 0
        for dict.hts[t].table[idx] == nil && cnt < 1000 {
                idx = rand.Int63n(dict.hts[t].size)
                cnt += 1
        }
        if dict.hts[t].table[idx] == nil {
                return nil
        }
        var listLen int64
        p := dict.hts[t].table[idx]
        for p != nil {
                listLen += 1
                p = p.next
        }
        listIdx := rand.Int63n(listLen)
        p = dict.hts[t].table[idx]
        for i := int64(0); i < listIdx; i++ {
                p = p.next
        }
        return p
}

func (dict *Dict) keyIndex(key *obj.Gobj) int64 {
        err := dict.expandIfNeeded()
        if err != nil {
                return -1
        }
        h := dict.HashFunc(key)
        var idx int64
        for i := 0; i <= 1; i++ {
                idx = h & dict.hts[i].mask
                e := dict.hts[i].table[idx]
                for e != nil {
                        if dict.EqualFunc(e.Key, key) {
                                return -1
                        }
                        e = e.next
                }
                if !dict.isRehashing() {
                        break
                }
        }
        return idx
}

func (dict *Dict) AddRaw(key *obj.Gobj) *Entry {
        if dict.isRehashing() {
                dict.rehashStep()
        }
        // 获取index
        idx := dict.keyIndex(key)
        if idx == -1 {
                return nil
        }
        var ht *htable
        if dict.isRehashing() {
                ht = dict.hts[1]
        } else {
                ht = dict.hts[0]
        }
        var e Entry
        e.Key = key
        key.IncrRefCount()
        e.next = ht.table[idx]
        ht.table[idx] = &e
        ht.used += 1
        return &e
}

func (dict *Dict) Add(key *obj.Gobj, val *obj.Gobj) error {
        entry := dict.AddRaw(key)
        if entry == nil {
                return EX_ERR
        }
        entry.Val = val
        val.IncrRefCount()
        return nil
}

func (dict *Dict) Set(key *obj.Gobj, val *obj.Gobj) {
        if err := dict.Add(key, val); err == nil {
                return
        }
        entry := dict.Find(key)
        entry.Val.DecrRefCount()
        entry.Val = val
        val.IncrRefCount()
}

func (dict *Dict) Find(key *obj.Gobj) *Entry {
        if dict.hts[0] == nil {
                return nil
        }
        if dict.isRehashing() {
                dict.rehashStep()
        }
        h := dict.HashFunc(key)
        for i := 0; i <= 1; i++ {
                idx := h & dict.hts[i].mask
                e := dict.hts[i].table[idx]
                for e != nil {
                        if dict.EqualFunc(e.Key, key) {
                                return e
                        }
                        e = e.next
                }
                if !dict.isRehashing() {
                        break
                }
        }
        return nil
}

func (dict *Dict) Get(key *obj.Gobj) *obj.Gobj {
        entry := dict.Find(key)
        if entry == nil {
                return nil
        }
        return entry.Val
}
```

## Net

net 模块主要是用于监听端口，每轮循环会查询是否有 client 接入，如果有则将其句柄返回，另外还有往句柄中读写相关 buffer

```golang
package net

import (
        "log"

        "golang.org/x/sys/unix"
)

const BACKLOG int = 64

func Accept(fd int) (int, error) {
        nfd, _, err := unix.Accept(fd)
        return nfd, err
}

func Close(fd int) {
        unix.Close(fd)
}

func Connect(host [4]byte, port int) (int, error) {
        // 创建socket
        s, err := unix.Socket(unix.AF_INET, unix.SOCK_STREAM, 0)
        if err != nil {
                log.Printf("init socket err: %v\n", err)
                return -1, err
        }
        // 地址绑定
        var addr unix.SockaddrInet4
        addr.Addr = host
        addr.Port = port
        // 链接
        err = unix.Connect(s, &addr)
        if err != nil {
                log.Printf("connect err: %v\n", err)
                return -1, err
        }
        return s, nil
}

func TcpServer(port int) (int, error) {
        // 创建socket
        s, err := unix.Socket(unix.AF_INET, unix.SOCK_STREAM, 0)
        if err != nil {
                log.Printf("init socket err: %v\n", err)
                return -1, nil
        }
        // 设置正在使用及端口复用
        err = unix.SetsockoptInt(s, unix.SOL_SOCKET, unix.SO_REUSEPORT, port)
        if err != nil {
                log.Printf("set SO_REUSEPORT err: %v\n", err)
                unix.Close(s)
                return -1, nil
        }
        // 地址绑定
        var addr unix.SockaddrInet4
        addr.Port = port
        err = unix.Bind(s, &addr)
        if err != nil {
                log.Printf("bind addr err: %v\n", err)
                unix.Close(s)
                return -1, nil
        }
        err = unix.Listen(s, BACKLOG)
        if err != nil {
                log.Printf("listen socket err: %v\n", err)
                unix.Close(s)
                return -1, nil
        }
        return s, nil
}

func Read(fd int, buf []byte) (int, error) {
        return unix.Read(fd, buf)
}

func Write(fd int, buf []byte) (int, error) {
        return unix.Write(fd, buf)
}
```

## AeLoop

AeLoop 是 redis 运行时的主要处理对象，通过创建 Epoll 后，遍历 FileEvents 和 TimeEvents 两个 list，找寻需要执行的 event，执行他们。数据结构如下：

```golang
type FeType int

const (
        AE_READABLE FeType = 1
        AE_WRITABLE FeType = 2
)

type TeType int

const (
        AE_NORMAL TeType = 1
        AE_ONCE   TeType = 2
)

type FileCallback func(loop *AeLoop, fd int, extra interface{})
type TimeCallback func(loop *AeLoop, fd int, extra interface{})

var fe2ep = [3]uint32{0, unix.EPOLLIN, unix.EPOLLOUT}

type AeFileEvent struct {
        fd    int
        mask  FeType
        cb    FileCallback
        extra interface{}
}

type AeTimeEvent struct {
        id       int
        mask     TeType
        when     int64 //ms
        interval int64 // 频率
        cb       TimeCallback
        extra    interface{}
        next     *AeTimeEvent
}

type AeLoop struct {
        FileEvents      map[int]*AeFileEvent
        TimeEvents      *AeTimeEvent
        fileEventFd     int
        timeEventNextId int
        stop            bool
}
```

创建及处理逻辑如下：

```golang
// 返回fileEvent的key，可读为正，可写为负
func getFeKey(fd int, mask FeType) int {
        if mask == AE_READABLE {
                return fd
        } else {
                return fd * -1
        }
}

func AeLoopCreate() (*AeLoop, error) {
        // mac 未暴露
        epollFd, err := unix.EpollCreate1(0)
        if err != nil {
                return nil, err
        }
        return &AeLoop{
                FileEvents:      make(map[int]*AeFileEvent),
                fileEventFd:     epollFd,
                timeEventNextId: 1,
                stop:            false,
        }, nil
}

// 主循环
func (loop *AeLoop) AeMain() {
        for !loop.stop {
                tes, fes := loop.AeWait()
                loop.AeProcess(tes, fes)
        }
}

// 获取最近的时间
func (loop *AeLoop) nearestTime() int64 {
        var nearest int64 = utils.GetMsTime() + 1000
        p := loop.TimeEvents
        for p != nil {
                if p.when < nearest {
                        nearest = p.when
                }
                p = p.next
        }
        return nearest
}

// 等待事件
func (loop *AeLoop) AeWait() (tes []*AeTimeEvent, fes []*AeFileEvent) {
        // 获取超时时间
        timeout := loop.nearestTime() - utils.GetMsTime()
        if timeout <= 0 {
                timeout = 10 // at least wait 10ms
        }
        var events [128]unix.EpollEvent
        // 获取两次timeout之间的所有fe事件
        n, err := unix.EpollWait(loop.fileEventFd, events[:], int(timeout))
        if err != nil {
                log.Printf("epoll wait warnning: %v\n", err)
        }
        if n > 0 {
                log.Printf("ae get %v epoll events\n", n)
        }
        // 查询对应事件，并存储到fes中
        for i := 0; i < n; i++ {
                if events[i].Events&unix.EPOLLIN != 0 {
                        fe := loop.FileEvents[getFeKey(int(events[i].Fd), AE_READABLE)]
                        if fe != nil {
                                fes = append(fes, fe)
                        }
                }
                if events[i].Events&unix.EPOLLOUT != 0 {
                        fe := loop.FileEvents[getFeKey(int(events[i].Fd), AE_WRITABLE)]
                        if fe != nil {
                                fes = append(fes, fe)
                        }
                }
        }
        // 查询需要执行的事件
        now := utils.GetMsTime()
        p := loop.TimeEvents
        for p != nil {
                if p.when <= now {
                        tes = append(tes, p)
                }
                p = p.next
        }
        return nil, nil
}

// 处理事件
func (loop *AeLoop) AeProcess(tes []*AeTimeEvent, fes []*AeFileEvent) {
        for _, te := range tes {
                te.cb(loop, te.id, te.extra)
                if te.mask == AE_ONCE {
                        loop.RemoveTimeEvent(te.id)
                } else {
                        te.when = utils.GetMsTime() + te.interval
                }
        }

        for _, fe := range fes {
                fe.cb(loop, fe.fd, fe.extra)
        }
}

func (loop *AeLoop) getEpollMask(fd int) (ev uint32) {
        if loop.FileEvents[getFeKey(fd, AE_READABLE)] != nil {
                ev |= fe2ep[AE_READABLE]
        }
        if loop.FileEvents[getFeKey(fd, AE_WRITABLE)] != nil {
                ev |= fe2ep[AE_WRITABLE]
        }
        return
}
```

添加事件和移除事件时，均会更新 epoll 的句柄

```golang
func (loop *AeLoop) AddFileEvent(fd int, mask FeType, callback FileCallback, extra interface{}) {
        // 查询当前fd是否存在可读或可写事件
        ev := loop.getEpollMask(fd)
        // ev != 0 && fe2ep[mask] != 0
        if ev&fe2ep[mask] != 0 {
                return
        }
        op := unix.EPOLL_CTL_ADD
        // 没有可读或可写事件
        if ev != 0 {
                op = unix.EPOLL_CTL_MOD
        }

        ev |= fe2ep[mask]
        err := unix.EpollCtl(loop.fileEventFd, op, fd, &unix.EpollEvent{Fd: int32(fd), Events: ev})
        if err != nil {
                log.Printf("epoll ctr err: %v\n", err)
                return
        }

        loop.FileEvents[getFeKey(fd, mask)] = &AeFileEvent{
                fd:    fd,
                mask:  mask,
                cb:    callback,
                extra: extra,
        }
        log.Printf("ae add file event fd:%v, mask:%v\n", fd, mask)
}

func (loop *AeLoop) RemoveFileEvent(fd int, mask FeType) {
        op := unix.EPOLL_CTL_DEL
        ev := loop.getEpollMask(fd)
        ev &= ^fe2ep[mask]
        if ev != 0 {
                op = unix.EPOLL_CTL_MOD
        }
        err := unix.EpollCtl(loop.fileEventFd, op, fd, &unix.EpollEvent{Fd: int32(fd), Events: ev})
        if err != nil {
                log.Printf("epoll del err: %v\n", err)
        }

        loop.FileEvents[getFeKey(fd, mask)] = nil
        log.Printf("ae remove file event fd:%v, mask:%v\n", fd, mask)
}

func (loop *AeLoop) AddTimeEvent(mask TeType, interval int64, callback TimeCallback, extra interface{}) int {
        id := loop.timeEventNextId
        loop.timeEventNextId++
        loop.TimeEvents = &AeTimeEvent{
                id:       id,
                mask:     mask,
                interval: interval,
                when:     utils.GetMsTime() + interval,
                cb:       callback,
                extra:    extra,
                next:     loop.TimeEvents,
        }
        return id
}

func (loop *AeLoop) RemoveTimeEvent(id int) {
        p := loop.TimeEvents
        var pre *AeTimeEvent
        for p != nil {
                if p.id == id {
                        if pre == nil {
                                loop.TimeEvents = p.next
                        } else {
                                pre.next = p.next
                                p.next = nil
                        }
                        break
                }
                pre = p
                p = p.next
        }
}
```

# 顶层实现

在基础模块处理好后，我们就可以完成顶层的 redis server 和 redis client 的实现了

## redis server

Redis server 比较简单，内部主要是实现了储存的 db，client 的列表和 aeloop，在 redis 运行起来时，就是创建 redis server，运行 server 里的 aeloop，将 AcceptHandler 放置于 FileEventList 中，调用 AcceptHandler 时创建 redis client

```golang
type GodisDB struct {
        data   *dict.Dict
        expire *dict.Dict
}

type GodisServer struct {
        fd      int
        port    int
        db      *GodisDB
        clients map[int]*GodisClient
        aeLoop  *ae.AeLoop
}

// 接受client
func AcceptHandler(loop *ae.AeLoop, fd int, extra interface{}) {
        cfd, err := net.Accept(fd)
        if err != nil {
                log.Printf("accept err: %v\n", err)
                return
        }

        client := CreateClient(cfd)
        //TODO: check max clients limit
        server.clients[cfd] = client
        server.aeLoop.AddFileEvent(cfd, ae.AE_READABLE, ReadQueryFromClient, client)
        log.Printf("accept client, fd: %v\n", cfd)
}

const EXPIRE_CHECK_COUNT int = 100

// 随机取数据判断是否过期
func ServerCron(loop *ae.AeLoop, id int, extra interface{}) {
        for i := 0; i < EXPIRE_CHECK_COUNT; i++ {
                entry := server.db.expire.RandomGet()
                if entry == nil {
                        break
                }
                if entry.Val.IntVal() < time.Now().Unix() {
                        server.db.data.Delete(entry.Key)
                        server.db.expire.Delete(entry.Key)
                }
        }
}

// 初始化godis server
func initServer(config *conf.Config) error {
        server.port = config.Port
        server.clients = make(map[int]*GodisClient)
        server.db = &GodisDB{
                data:   dict.DictCreate(dict.DictType{HashFunc: utils.GStrHash, EqualFunc: utils.GStrEqual}),
                expire: dict.DictCreate(dict.DictType{HashFunc: utils.GStrHash, EqualFunc: utils.GStrEqual}),
        }
        var err error
        if server.aeLoop, err = ae.AeLoopCreate(); err != nil {
                return err
        }
        server.fd, err = net.TcpServer(server.port)
        return err
}

func main() {
        path := os.Args[1]
        config, err := conf.LoadConfig(path)

        if err != nil {
                log.Printf("config error: %v\n", err)
        }
        err = initServer(config)
        if err != nil {
                log.Printf("init server error: %v\n", err)
        }

        server.aeLoop.AddFileEvent(server.fd, ae.AE_READABLE, AcceptHandler, nil)
        server.aeLoop.AddTimeEvent(ae.AE_NORMAL, 100, ServerCron, nil)
        log.Println("godis server is up.")
        server.aeLoop.AeMain()
}
```

## redis client

redis 就相对复杂一些，首先先要解析命令，判断是 InLine 还是 MuttiBulk 命令，协议为：https://xargin.com/redis-protocal/

```golang
type GodisClient struct {
        fd       int
        db       *GodisDB
        args     []*obj.Gobj
        reply    *list.List
        queryBuf []byte
        queryLen int // client.queryLen: 未处理的长度
        cmdType  utils.CmdType
        bulkNum  int
        bulkLen  int
        sentLen  int
}

// 通过首字符和结尾 "\r\n" 区分每一条命令
// +: 简单string
// -: error 如错误返回
// :: int 如自增命令后的返回
// $: string，"$4\r\ntest\r\n"后接数字，告诉string长度，\r\n后为string文本，可以包含换行符之类的字符
// *: array 后接数字，表示数组长度

// redis 命令
// InLine: "set key val\r\n"
// MuttiBulk: "*3\r\n$3\r\nset\r\n$3\r\nkey\r\n$3\r\nval\r\n"

// 处理client query
func ProcessQueryBuf(client *GodisClient) error {
        // 不断取值
        for len(client.queryBuf) > 0 {
                if client.cmdType == utils.COMMAND_UNKNOWN {
                        if client.queryBuf[0] == '*' {
                                client.cmdType = utils.COMMAND_BULK
                        } else {
                                client.cmdType = utils.COMMAND_INLINE
                        }
                }

                // 读取内容转化为cmd args
                var ok bool
                var err error
                if client.cmdType == utils.COMMAND_INLINE {
                        ok, err = handleInlineBuf(client)
                } else if client.cmdType == utils.COMMAND_BULK {
                        ok, err = handleBulkBuf(client)
                } else {
                        return errors.New("unknow Godis Command Type")
                }
                if err != nil {
                        return err
                }

                // 执行cmd
                if ok {
                        if len(client.args) == 0 {
                                resetClient(client)
                        } else {
                                ProcessCommand(client)
                        }
                } else {
                        // 未读取完，下次再处理
                        break
                }
        }
        return nil
}


func handleInlineBuf(client *GodisClient) (bool, error) {
        index, err := client.findLineInQuery()
        if index < 0 {
                return false, err
        }

        subs := strings.Split(string(client.queryBuf[:index]), " ")
        client.queryBuf = client.queryBuf[index+2:]
        client.queryLen -= index + 2
        client.args = make([]*obj.Gobj, len(subs))

        for i, v := range subs {
                client.args[i] = obj.CreateObject(obj.GSTR, v)
        }
        return true, nil
}

func handleBulkBuf(client *GodisClient) (bool, error) {
        // init bulkNum
        if client.bulkNum == 0 {
                index, err := client.findLineInQuery()
                if index < 0 {
                        return false, err
                }

                bnum, err := client.getNumInQuery(1, index)
                if err != nil {
                        return false, err
                }
                if bnum == 0 {
                        return true, nil
                }
                client.bulkNum = bnum
                client.args = make([]*obj.Gobj, bnum)
        }
        // 读取所有 bulk string
        for client.bulkNum > 0 {
                // init bulkLen
                if client.bulkLen == 0 {
                        index, err := client.findLineInQuery()
                        if index < 0 {
                                return false, err
                        }

                        if client.queryBuf[0] != '$' {
                                return false, errors.New("expect $ for bulk length")
                        }

                        blen, err := client.getNumInQuery(1, index)
                        if err != nil || blen == 0 {
                                return false, err
                        }
                        if blen > utils.GODIS_MAX_BULK {
                                return false, errors.New("too big bulk")
                        }
                        client.bulkLen = blen
                }
                // read bulk string
                if client.queryLen < client.bulkLen+2 {
                        return false, nil
                }
                index := client.bulkLen
                // 判断结尾是否为\r\n
                if client.queryBuf[index] != '\r' || client.queryBuf[index+1] != '\n' {
                        return false, errors.New("expect CRLF for bulk end")
                }
                client.args[len(client.args)-client.bulkNum] = obj.CreateObject(obj.GSTR, string(client.queryBuf[:index]))
                client.queryBuf = client.queryBuf[index+2:]
                client.queryLen -= index + 2
                client.bulkLen = 0
                client.bulkNum -= 1
        }

        return true, nil
}
```

然后执行对应的命令

```golang
func (c *GodisClient) AddReply(o *obj.Gobj) {
        c.reply.Append(o)
        o.IncrRefCount()
        server.aeLoop.AddFileEvent(c.fd, ae.AE_WRITABLE, SendReplyToClient, c)
}

func (c *GodisClient) AddReplyStr(str string) {
        o := obj.CreateObject(obj.GSTR, str)
        c.AddReply(o)
        o.DecrRefCount()
}

// 寻找对应的cmd
func lookupCommand(cmdStr string) *GodisCommand {
        for _, c := range cmdTable {
                if c.name == cmdStr {
                        return &c
                }
        }
        return nil
}

// 执行cmd
func ProcessCommand(client *GodisClient) {
        cmdStr := client.args[0].StrVal()
        log.Printf("process command: %v\n", cmdStr)
        if cmdStr == "quit" {
                freeClient(client)
                return
        }
        cmd := lookupCommand(cmdStr)
        if cmd == nil {
                client.AddReplyStr("-ERR: unknow command")
                resetClient(client)
                return
        } else if cmd.arity != len(client.args) {
                client.AddReplyStr("-ERR: wrong number of args")
                resetClient(client)
                return
        }
        cmd.proc(client)
        resetClient(client)
}

func handleInlineBuf(client *GodisClient) (bool, error) {
        index, err := client.findLineInQuery()
        if index < 0 {
                return false, err
        }

        subs := strings.Split(string(client.queryBuf[:index]), " ")
        client.queryBuf = client.queryBuf[index+2:]
        client.queryLen -= index + 2
        client.args = make([]*obj.Gobj, len(subs))

        for i, v := range subs {
                client.args[i] = obj.CreateObject(obj.GSTR, v)
        }
        return true, nil
}
```

最后调用 SendReplyToClient 注销 client

```golang
// 通知client
func SendReplyToClient(loop *ae.AeLoop, fd int, extra interface{}) {
        client := extra.(*GodisClient)
        log.Printf("SendReplyToClient, reply len:%v\n", client.reply.Length())
        for client.reply.Length() > 0 {
                rep := client.reply.First()
                buf := []byte(rep.Val.StrVal())
                bufLen := len(buf)
                if client.sentLen < bufLen {
                        n, err := net.Write(fd, buf[client.sentLen:])
                        if err != nil {
                                log.Printf("send reply err: %v\n", err)
                                freeClient(client)
                                return
                        }
                        client.sentLen += n
                        log.Printf("send %v bytes to client:%v\n", n, client.fd)
                        if client.sentLen == bufLen {
                                client.reply.DelNode(rep)
                                rep.Val.DecrRefCount()
                                client.sentLen = 0
                        } else {
                                break
                        }
                }
        }
        if client.reply.Length() == 0 {
                client.sentLen = 0
                loop.RemoveFileEvent(fd, ae.AE_WRITABLE)
        }
}
```

# 生产环境应用

由于 redis 是单线程（即使在 4.0 后加入多线程也仅仅是在网络层，在储存层依旧是单线程），在真实的生产环境开发中，往往选择 1 核/2G 或者 2 核/4G 作为单例，采取多实例去集群部署。
在集群部署的时候，部署通信策略往往更为重要，目前有以下几种策略：

- Smart client：这种是最简单且不好的形式，即 client 来计算（比如通过 hash）存储到哪个实例上
- Redis cluster：这是官方提供的集群部署方式，redis 实例间相同通信来自动分配，但是这种情况在扩容的时候，会引发大量的 k-v 移动，而且在集群数量大的时候，即使是单个实例错误率很低（比如 1%），在大集群时（比如上万）集群的错误率也是不可忽视的。
- proxy： 这是业界主流的策略，即在 client 层和 redis 层之间添加一层 proxy 来代理计算调度储存分布，并且也可监听底层 redis 状态，出现问题也能及时进行重启等容灾处理，像解决 redis 单核问题的 [dragonfly](https://github.com/dragonflydb/dragonfly) 项目，也是这个设计思路。

另外在持久化方面，也是在 redis 缓存下加了一层 rocksdb 进行储存，感兴趣的话可以去查查，这里就不展开描述了。

# 番外

goroutine 是协程吗，什么是协程？
在用 go 写这个项目的时候，顺便多了解了一下协程这个概念，并且产生了这个疑问。
首先先回答什么是协程，在[C++20](https://en.cppreference.com/w/cpp/language/coroutines)中说道 A coroutine is a function that can suspend execution to be resumed later.简单来说可以认为协程是线程里不同的函数，这些函数之间可以相互快速切换。
而在 go 中，我们使用的是 go func 的形式去处理，而这个又与协程的描述不相符，在 go 中，goroutine 更像是用户态线程而不是协程。
那么协程到底是什么样子呢？其实就是我们 js 的 async/await 的样子,做到了函数之间可以相互快速切换。

```javascript
async function sample() {
  const result = await sampleResolve();
  console.log(result);
}
```

但是实际上，只有在 ES7 及以上，底层的实现，才是真正的协程，参考：https://www.zhihu.com/question/305443189
