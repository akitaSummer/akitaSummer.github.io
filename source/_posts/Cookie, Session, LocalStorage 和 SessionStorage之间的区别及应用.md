---
title: Cookie, Session, LocalStorage 和 SessionStorage之间的区别及应用
date: 2019-08-31 16:50:25
tags:
	- JavaScript
	- HTML5
categories: 学习笔记
---

# Cookie
## 什么是Cookie
 >Cookie 是服务器保存在浏览器的一小段文本信息，每个 Cookie 的大小一般不能超过4KB。浏览器每次向服务器发出请求，就会自动附上这段信息。

简而言之, Cookie是每次发送请求时携带的一段数据。

## Cookie的分类
按存在时间来分类的话，可以分为会话期Cookie和持久性Cookie：
   - 会话期Cookie
   保存在内存中，当浏览器关闭时会自动销毁。
   - 持久性Cookie
   通过设置 `Expires` (过期时间)或 `Max-Age` (有效期)，使其保存在硬盘中，直到时间过期才消失。

## Cookie的应用
在[MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies)中有详细的介绍。
简而言之可以分为：
   - 记住密码， 实现自动登录。
   - 保存登陆时间，浏览次数等信息。

# Session
- 储存在服务器上
- 由于SessionID保存在Cookie中，其依赖Cookie
- 根据Session可以得到用户的信息

# webStorage
webStorage是 html5 提供的本地存储方案。目的是为了克服由cookie带来的一些限制，当数据需要被严格控制在客户端上时，无须持续地将数据发回服务器。
相比Cookie，webStorage有很多优点：
   - 保存在客户端， 不与服务器通信。
   - 储存空间比Cookies更大(5MB)。
   - 比Cookies更快，更安全。
   - HTML5提供了原生接口，操作更加方便。
      - `setItem(key, value)` 保存数据
	  - `getItem(key)` 提取数据
	  - `removeItem(key)` 删除某个数据
	  - `clear()` 清除所有数据
	  - `key(index)` 获取某个索引的key
	  
webStorage主要包括两种方式：sessionStorage 和 localStorage。

## localStorage
- 持久化本地存储，永久有效。
- 可应用于自动登录等长期保存在本地的数据。
```javascript
// 保存数据到 LocalStorage
localStorage.setItem('key', 'value') 

// 从 LocalStorage 提取数据
localStorage.getItem('key')

// 从 LocalStorage 删除某个单一的数据
localStorage.removeItem('key')

// 清除所有保存在 LocalStorage 的数据
localStorage.clear();
```
## sessionStorage
- 浏览器关闭后数据自动销毁。
- 应用于敏感信息。
```javascript
// 保存数据到 sessionStorage
sessionStorage.setItem('key', 'value') 

// 从 sessionStorage 提取数据
sessionStorage.getItem('key')

// 从 sessionStorage 删除某个单一的数据
sessionStorage.removeItem('key')

// 清除所有保存在 sessionStorage 的数据
sessionStorage.clear();
```
# 总结
## Cookie 和 Session
- Cookie保存在硬盘里，而Session是保存在服务端的内存里。
- Cookie每次都随着HTTP请求发送给服务器，Session通过发送保存在Cookie头中的SessionID即可得到对应的数据。
## Cookie 和 webStorage
- Cookie每次都会携带在HTTP头中，webStorage仅在客户端中保存，不参与与服务器之间的通信。
- Cookie只有4kb储存大小而webStorage有5MB。
- Cookie一般由服务器生成，用于标识用户身份，webStorage用于浏览器缓存数据储存。