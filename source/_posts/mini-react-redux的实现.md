---
title: mini-react-redux的实现
date: 2019-07-10 16:17:54
tags:
	- JavaScript
  - Redux
  - React
categories: 学习笔记
---
# 学习的目的
在学习完Redux后，在React项目中实际应用时，往往我们使用的是Redux作者封装的React专用的库React-Redux。
React-Redux提供了便利，能更好的操作Redux,并且可以让redux和react的耦合度进一步的降低。
在学习完Redux后学习React-Redux，可以让自己对于React和Redux建立起一个桥梁。
# 代码实现
React-Redux 将所有组件分成两大类：UI 组件（presentational component）和容器组件（container component）。具体的介绍可以参照[阮一峰老师的博客](http://www.ruanyifeng.com/blog/2016/09/redux_tutorial_part_three_react-redux.html)。
我们主要实现其中的Provider组件类和connect函数
# Provider组件类
当我们渲染主页面时，使用的是
```javascript
ReactDOM.render((<Provider store={store}><App/></Provider>), document.getElementById('root'))
```
所以Provider组件类中props会被传入Redux的store，并且在子组件中都会得到store数据，这时就需要使用到childContextTypes和getChildContext()
首先我们先将Provider组件类的结构制作出来
```javascript
import React, {Component} from 'react'
import PropTypes from 'prop-types'
// Provider组件类
// 为所有容器子组件提供store
// <Provider store={store}><App/></Provider>
export class Provider extends Component {

  static propTypes = {
    store: PropTypes.object.isRequired
  }

  render() {
    return this.props.children // 将所有子标签返回
  }
}
```
再向其中添加向子组件提供数据的方法(childContextTypes和getChildContext())
```javascript

  // 声明向子组件提供哪些context数据
  static childContextTypes = {
    store: PropTypes.object.isRequired
  }

  // 为子组件提供包含store的context
  getChildContext() {
    // 返回的就是context对象
    return {
      store: this.props.store
    }
  }
```
最终完成Provider组件类
```javascript
import React, {Component} from 'react'
import PropTypes from 'prop-types'
// Provider组件类
// 为所有容器子组件提供store
// <Provider store={store}><App/></Provider>
export class Provider extends Component {

  static propTypes = {
    store: PropTypes.object.isRequired
  }

  // 声明向子组件提供哪些context数据
  static childContextTypes = {
    store: PropTypes.object.isRequired
  }

  // 为子组件提供包含store的context
  getChildContext() {
    // 返回的就是context对象
    return {
      store: this.props.store
    }
  }
  render() {
    return this.props.children // 将所有子标签返回
  }
}
```
# connect函数
我们在使用其他组件时，往往使用connect方法将UI组件生成容器组件。
```javascript
export default connect(state => ({}), {})(Xxx)
```
第一个参数是函数, 用来确定一般属性。第二个参数是对象, 用来确定函数(内部会使用dispatch方法)属性。
并且connect函数调用后会返回一个函数，在被传入UI组件，最终返回一个容器组件
我们先将connect结构创建出来
```javascript
// connect 函数
// export default connect(state => ({}), {})(Xxx)
// mapStateToProps: 函数, 用来确定一般属性
// mapDispatchToProps: 对象, 用来确定函数(内部会使用dispatch方法)属性
export function connect(mapStateToProps, mapDispatchToProps) {
  // 返回一个函数(接收一个组件)
  return (WrapComponent) => {
    // 返回一个容器组件
    return class ConnectComponent extends Component {
      
    }
  }
}
```
WrapComponent组件(即传入的UI组件)必须获得Redux中的store和actions，然后最终被渲染。
```javascript
return class ConnectComponent extends Component {
  render() {
      return <WrapComponent {...this.state} {...this.dispatchProps}/> // this.state: Redux中的store, this.dispatchProps: Redux中的actions
  }
}
```
使this.state得到Redux中的store, this.dispatchProps得到Redux中的actions。
```javascript
// mapStateToProps: 函数, 用来确定一般属性
// mapDispatchToProps: 对象, 用来确定函数(内部会使用dispatch方法)属性
return class ConnectComponent extends Component {
      // 声明获取context数据
      static contextTypes = {
        store: PropTypes.object.isRequired
      }

      constructor (props, context) {
        super(props, context)
        // 得到store
        const store = context.store
        // 包含一般属性的对象
        const stateProps = mapStateToProps(store.getState())
        // 包含函数属性的对象
        const dispatchProps = this.bindActionCreators(mapDispatchToProps) // 根据mapDispatchToProps返回一个包含分发action的函数的对象
        // 将所有的一般属性都保存到state中
        this.state = {
          ...stateProps // count msgs
        }
        // 将所有函数属性的对象保存组件对象
        this.dispatchProps = dispatchProps
      }

      render() {
        return <WrapComponent {...this.state} {...this.dispatchProps}/>
      }
    }
```
定义bindActionCreators函数，用于根据mapDispatchToProps返回一个包含分发action的函数的对象。
```javascript
// mapStateToProps: 函数, 用来确定一般属性
// mapDispatchToProps: 对象, 用来确定函数(内部会使用dispatch方法)属性
return class ConnectComponent extends Component {
      // 声明获取context数据
      static contextTypes = {
        store: PropTypes.object.isRequired
      }

      constructor (props, context) {
        ...
      }

      // 根据mapDispatchToProps返回一个包含分发action的函数的对象
      bindActionCreators = (mapDispatchToProps) => {
        return Object.keys(mapDispatchToProps).reduce((preDispatchToProps, key) => {
          // 添加一个包含dispatch语句的方法
          preDispatchToProps[key] =  (...args) => { // 透传: 将函数接收到参数, 原样传内部函数调用
            // 分发action
            this.context.store.dispatch(mapDispatchToProps[key](...args))
          }
          return preDispatchToProps
        }, {})
      }

      render() {
        return <WrapComponent {...this.state} {...this.dispatchProps}/>
      }
    }
```
最后不要忘了为每个组件订阅监听。
```javascript
return class ConnectComponent extends Component {
  // 声明获取context数据
  static contextTypes = {
    store: PropTypes.object.isRequired
  }

  constructor (props, context) {
    ...
  }

  // 根据mapDispatchToProps返回一个包含分发action的函数的对象
  bindActionCreators = (mapDispatchToProps) => {
    ...
  }

  componentDidMount() {
    // 得到store
    const store = this.context.store
    // 订阅监听
    this.context.store.subscribe(() => {
      // redux中产生了一个新的state
      // 更细当前组件的状态
      this.setState(mapStateToProps(store.getState()))
    })
  }

  render() {
    return <WrapComponent {...this.state} {...this.dispatchProps}/>
  }
}
```
# 最终代码
```javascript
/*
* react-redux模块
*   Provider
*   connect
* */

import React, {Component} from 'react'
import PropTypes from 'prop-types'
import {bindActionCreators} from "redux";

// Provider组件类
// 为所有容器子组件提供store
// <Provider store={store}><App/></Provider>
export class Provider extends Component {

  static propTypes = {
    store: PropTypes.object.isRequired
  }

  // 声明向子组件提供哪些context数据
  static childContextTypes = {
    store: PropTypes.object.isRequired
  }

  // 为子组件提供包含store的context
  getChildContext() {
    // 返回的就是context对象
    return {
      store: this.props.store
    }
  }
  render() {
    return this.props.children // 将所有子标签返回
  }
}

// connect 函数
// export default connect(state => ({}), {})(Xxx)
// mapStateToProps: 函数, 用来确定一般属性
// mapDispatchToProps: 对象, 用来确定函数(内部会使用dispatch方法)属性
export function connect(mapStateToProps, mapDispatchToProps) {
  // 返回一个函数(接收一个组件)
  return (WrapComponent) => {
    // 返回一个容器组件
    return class ConnectComponent extends Component {
      // 声明获取context数据
      static contextTypes = {
        store: PropTypes.object.isRequired
      }

      constructor (props, context) {
        super(props, context)
        // 得到store
        const store = context.store
        // 包含一般属性的对象
        const stateProps = mapStateToProps(store.getState())
        // 包含函数属性的对象
        const dispatchProps = this.bindActionCreators(mapDispatchToProps)
        // 将所有的一般属性都保存到state中
        this.state = {
          ...stateProps // count msgs
        }
        // 将所有函数属性的对象保存组件对象
        this.dispatchProps = dispatchProps
      }

      // 根据mapDispatchToProps返回一个包含分发action的函数的对象
      bindActionCreators = (mapDispatchToProps) => {
        return Object.keys(mapDispatchToProps).reduce((preDispatchToProps, key) => {
          // 添加一个包含dispatch语句的方法
          preDispatchToProps[key] =  (...args) => { // 透传: 将函数接收到参数, 原样传内部函数调用
            // 分发action
            this.context.store.dispatch(mapDispatchToProps[key](...args))
          }
          return preDispatchToProps
        }, {})
      }

      componentDidMount() {
        // 得到store
        const store = this.context.store
        // 订阅监听
        this.context.store.subscribe(() => {
          // redux中产生了一个新的state
          // 更细当前组件的状态
          this.setState(mapStateToProps(store.getState()))
        })
      }

      render() {
        return <WrapComponent {...this.state} {...this.dispatchProps}/>
      }
    }
  }
}

```