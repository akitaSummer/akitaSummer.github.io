---
title: mini-redux的实现
date: 2019-07-03 19:54:35
tags:
	- JavaScript
	- Redux
categories: 学习笔记
---
# 学习的目的
Redux 是React生态中重要的组成部分。
通过对redux进行mini的实现能够加深自己对于Redux的工作流程的理解。
 ![Redux Flux](/image/mini_redux/redux_flux.png)
网上广为流传的Redux流向图，可以帮助我们更好地理解Redux的工作原理。

# 代码实现
我们的mini-redux模块将主要实现以下3个方面
1. createStore(reducer): 接收一个reducer函数, 返回一个store对象
2. combineReducers(reducers): 接收一个包含多个reducer函数的对象, 返回一个新的reducer函数
3. store对象
    getState(): 得到内部管理的state对象
    dispatch(action): 分发action, 会触发reducer调用, 返回一个新的state, 调用所有绑定的listener
    subscribe(listen): 订阅一个state的监听器

## 1. createStore(reducer)
在Redux中Store是一个大仓库,所有的state都会储存在store内部当中，而createStore则是创建它的API。
在调用createStore的时候，我们会传入reducer，让Store中能有管理的state对象的初始值。
```javascript
// 将createStore暴露，供人调用
export function createStore(reducer) {
    // Store对象中储存的state
    let state
    // 在创建Store时需自动调用一次reducer来得到初始状态并保存
    state = reducer(state, {type: '@mini-redux'})
    // 返回一个Store对象，内部具有getState，dispatch，subscribe三个方法可以调用，在第三部分具体实现
    function getState() {}
    function dispatch() {}
    function subscribe() {}
    return {getState, dispatch, subscribe}
}
```
## 2. combineReducers(reducers)
在我们制作的createStore所创建的Store只能保存一个state，当我们有多个state时，可以调用combineReducers，向其传入一个包含所有reducer的对象reducers，通过combineReducers将所有的reducer转化成为一个reducer。
```javascript
// combineReducers(reducers): 接收一个包含多个reducer函数的对象, 返回一个新的reducer函数
export function combineReducers(reducers) {
    // 返回一个整合的reducer函数
    return function (state={}, action) {
        // 准备一个保存所有状态的state
        const newState = {}
        // 得到所有的reducer函数名
        const keys = Object.keys(reducers)
        // 依次调用所有的reducer函数，得到n个state，封装成对象并返回
        keys.forEach(key => {
            // 得到对应的子state
            const childState = state[key]
            // 得到对应的子reducer
            const childReducer = reducers[key]
            // 调用reducer得到新的state
            const newChildState = childReducer(childState, action)
            // 将新state保存到总state中
            newState[key] = newChildState
        })
        // 所有state更新完后，返回新的state
        return newState
    }
}
```
我们可以利用Array中的reduce函数优化上述代码
```javascript
export function combineReducers2(reducers) {
  return function (state={}, action) { 
    return Object.keys(reducers).reduce((newState, reducerName) => {
      newState[reducerName] = reducers[reducerName](state[reducerName], action)
      return newState
    }, {})
  }
}
```
## 3. store对象
当将state初始化完成后，接下来就需要完成Store对象中的三个函数了
### 3.1 getState
getState最为简单，不需要任何参数，调用时只需要返回出当前的state状态即可
```javascript
// 将createStore暴露，供人调用
export function createStore(reducer) {
    ...
    // 得到state对象
    function getState() {
        return state
    }
    ...
    return {getState, dispatch, subscribe}
}
```
### 3.2 dispatch
dispatch的作用是分发action, 会触发reducer调用, 返回一个新的state, 调用所有绑定的listener
```javascript
export function createStore(reducer) {
    ...
    // 内部保存listener数组，
    const listeners = []
    // 分发action, 会触发reducer调用, 返回一个新的state, 调用所有绑定的listener
    function dispatch(action) {
        // 调用reducer，得到新的state，保存
        state = reducer(state, action)
        // 调用listeners中的所有回调函数，例：store.subscribe(function () {ReactDOM.render(<App store={store}/>, document.getElementById('root'))})为更新虚拟DOM
        listeners.forEach(listener => listener())
    }
    ...
    return {getState, dispatch, subscribe}
}
```
### 3.3 subscribe
subscribe的作用是订阅一个state的监听器，当state改变时，我们的界面也可跟着相应更新
```javascript
export function createStore(reducer) {
    ...
    // 订阅一个state监听器
    function subscribe(listener){
        listeners.push(listener)
    }
    return {getState, dispatch, subscribe}
}
```
# 最终代码
```javascript
// 将createStore暴露，供人调用
export function createStore(reducer) {
    // Store对象中储存的state
    let state
    // 在创建Store时需自动调用一次reducer来得到初始状态并保存
    state = reducer(state, {type: '@mini-redux'})
    // 返回一个Store对象，内部具有getState，dispatch，subscribe三个方法可以调用，在第三部分具体实现
    // 得到state对象
    function getState() {
        return state
    }
    // 内部保存listener数组
    const listeners = []
    // 分发action, 会触发reducer调用, 返回一个新的state, 调用所有绑定的listener
    function dispatch(action) {
        // 调用reducer，得到新的state，保存
        state = reducer(state, action)
        // 调用listeners中的所有回调函数，例：store.subscribe(function () {ReactDOM.render(<App store={store}/>, document.getElementById('root'))})为更新虚拟DOM
        listeners.forEach(listener => listener())
    }
    // 订阅一个state监听器
    function subscribe(listener){
        listeners.push(listener)
    }
    return {getState, dispatch, subscribe}
}

// combineReducers(reducers): 接收一个包含多个reducer函数的对象, 返回一个新的reducer函数
export function combineReducers(reducers) {
    // 返回一个整合的reducer函数
    return function (state={}, action) {
        // 准备一个保存所有状态的state
        const newState = {}
        // 得到所有的reducer函数名
        const keys = Object.keys(reducers)
        // 依次调用所有的reducer函数，得到n个state，封装成对象并返回
        keys.forEach(key => {
            // 得到对应的子state
            const childState = state[key]
            // 得到对应的子reducer
            const childReducer = reducers[key]
            // 调用reducer得到新的state
            const newChildState = childReducer(childState, action)
            // 将新state保存到总state中
            newState[key] = newChildState
        })
        // 所有state更新完后，返回新的state
        return newState
    }
}
```