---
title: webpack4配置vue开发环境
date: 2019-11-5 16:50:25
tags:
	- Webpack
	- Vue
categories: 学习笔记
---

# 1.初始化项目

创建好项目文件夹，如webpack-vue
然后我们要初始化项目：
```
npm init -y
```
之后在文件夹内分别创建dist(存放打包后项目)、src(代码源文件)、src/components和bulid(webpack打包相关配置)

在根目录下创建`index.html`作为模板
```html
<!DOCTYPE html>
<html lang="zh-CN">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Webpack-Vue</title>
</head>

<body>
    <div id="app"></div>
</body>

</html>
```
最后在src下创建入口文件`index.js`
# 配置Webpack
首先我们先安装webpack
```
npm install webpack webpack-cli -D
```
然后我们进入build文件夹，创建以下文件：
- webpack.base.conf.js (基础打包文件)
- webpack.dev.conf.js (开发环境配置)
- webpack.prod.conf.js (生产环境配置)
- build.js (Node打包脚本)

## webpack.base.conf.js
先安装HtmlWebpackPlugin插件
```
npm install clean-webpack-plugin -D
```
然后将基本配置信息写入
```javascript
// 基础打包配置
const path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')

module.exports = {
    entry: {
        bundle: path.resolve(__dirname, '../src/index.js')
    },
    output: {
        path: path.resolve(__dirname, '../dist'),
        filename: '[name].[hash].js'
    },
    module: {
        rules: []
    },
    plugins: [
        new HtmlWebpackPlugin({ // 生成创建html入口文件的插件
            template: path.resolve(__dirname, '../index.html')
        })
    ]
}
```
## webpack.dev.conf.js 
先安装webpack-merge用来合并两个配置和webpack-dev-server创建本地运行环境
```
npm install webpack-merge webpack-dev-server -D
```
然后写入开发环境配置
```javascript
// 开发环境配置
const merge = require('webpack-merge')
const path = require('path')
const baseConfig = require('./webpack.base.conf')

module.exports = merge(baseConfig, {
    mode: 'development', // 设置为开发环境
    devtool: 'inline-source-map',
    devServer: {
        contentBase: path.resolve(__dirname, '../dist'),
        open: true // 打开浏览器
    }
})
```
## webpack.prod.conf.js
为了删除生产环境多余的文件，先安装clean-webpack-plugin
```
npm install clean-webpack-plugin -D
```
然后写入生产环境配置
```javascript
// 生产环境配置
const merge = require('webpack-merge')
const path = require('path')
const baseConfig = require('./webpack.base.conf')
const { CleanWebpackPlugin } = require('clean-webpack-plugin')

module.exports = merge(baseConfig, {
    mode: 'production', // 生产环境
    devtool: 'source-map',
    module: {
        rules: []
    },
    plugins: [
        new CleanWebpackPlugin({ // 打包之前将文件先清除
            cleanOnceBeforeBuildPatterns: ['dist/'], 
            verbose: true, // 控制台打印日志
            dry: false, // 删除文件夹
            root: path.resolve(__dirname, '../'),
        })
    ]
})
```
## build.js
写入打包脚本
```javascript
// Node打包用脚本
const webpack = require('webpack')
const config = require('./webpack.prod.conf')
webpack(config, (error, stats) => {
    if (error || stats.hasErrors) {
        console.error(error)
        return
    }
    console.log(stats.toString({
        chunks: false, //构建过程中没有输出
        colors: true // 控制台显示颜色
    }))
})
```
## package.json
在`package.json`中script属性中加入如下命令，使我们更方便使用打包
```json
    "scripts": {
        "test": "echo \"Error: no test specified\" && exit 1",
        "build": "node build/build.js",
        "dev": "webpack-dev-server --inline --progress --config build/webpack.dev.conf.js"
    },
```
## 测试
在`src/index.js`文件中写入
```javascript
console.log('hello world')
```
命令行中执行
```
npm run dev
```
然后我们在命令行中执行
```
npm run build
```
进行构建
# 引入Loader
 >Loader 让 webpack 能够去处理那些非 JavaScript 文件（webpack 自身只理解 JavaScript）。loader 可以将所有类型的文件转换为 webpack 能够处理的有效模块，然后你就可以利用 webpack 的打包能力，对它们进行处理。

接下来我们配置一些Loader，好让我们的代码能兼容更多的环境
## babel-loader
babel-loader可以让代码兼容不同版本的浏览器
首先先安装babel-loader、babel-core和babel-preset-env
```
npm i babel-loader@7 babel-core babel-preset-env -D
```
需要注意的是`babel-core`必须比`babel-loader`低一大版本，否则会报错。
然后在`webpack.base.conf.js`中的`module.rules`里添加
```javascript
{
     test: /\.js$/,
     use: 'babel-loader',
     exclude: /node_modules/
}
```
然后在根路径下创建`.babellrc`文件作为配置文件
```json
{
    "presets": [
        [
            "env",
            {
                "modules": false,
                "targets": {
                    "browsers": ["> 1%", "last 2 versions", "not ie <= 8"]
                }
            }
        ]
    ]
}
```
以上代码主要是为了兼容最新两个版本浏览器且不兼容IE8
## file-loader
file-loader是常用的将字体、图片文件模块化的loader
首先先安装`file-loader`
```
npm install file-loader -D
```
然后在`webpack.base.conf.js`中的`module.rules`里添加
```javascript
{
    test: /\.(png|svg|jpg|gif|woff|woff2|eot|ttf|otf)$/,
    use: ['file-loader']
}
```
## vue-loader
为了能识别Vue文件，我们需要使用`vue-loader`
首先安装`vue-loader`、`css-loader`、`vue-style-loader`和`vue-template-compiler`
```
npm install vue-loader css-loader vue-style-loader vue-template-compiler -D
```
然后在`webpack.base.conf.js`中的`module.rules`里添加
```javascript
{
  test: /\.vue$/,
  loader: 'vue-loader'
},
{
  test: /\.css$/,
  use: ['vue-style-loader', 'css-loader']
}
```
然后配置插件
```javascript
const VueLoaderPlugin = require('vue-loader/lib/plugin')

...

plugins: [
        ...
        new VueLoaderPlugin(),
        ...
    ]
```
最后我们配置别名
```javascript
resolve: {
    alias: {
         'vue$': 'vue/dist/vue.esm.js', // 引入vue更方便
         '@': path.resolve(__dirname, '../src'),
    },
    extensions: ['*', '.js', '.json', '.vue'], //省略后缀
}
```
最重要的，别忘了安装Vue
```
npm install Vue --S
```
修改`index.js`
```
import Vue from 'vue'
import App from './App'

new Vue({
    el: '#app',
    render: h => h(App)
})
```
然后创建一个`App.vue`文件
```HTML
<template>
  <h1>Hello World!</h1>
</template>

<script>
  export default {
    name: 'App'
  }
</script>

<style>
  html, body {
    padding: 0;
    margin: 0;
    box-sizing: border-box;
    font-size: 16px;
  }
</style>
```
最后我们运行下`npm run dev`测试下就可以了
# CSS代码优化
我们使用postcss的autoprefixer插件来优化代码，使其自动添加前缀来适应不同的浏览器
首先先安装
```
npm install postcss-loader autoprefixer -D
```
然后在`webpack.base.conf.js`中的`module.rules`里修改配置
```javascript
{
  test: /\.css$/,
  use: ['vue-style-loader', 'css-loader', 'postcss-loader']
}
```
最后在根目录下创建配置文件`postcss.config.js`
```javascript
module.exports = {
    plugins: [
        require('autoprefixer'),
    ]
}
```
# 开启热更新
Webpack4开启热更新较为容易
在`webpack.dev.conf.js`中修改
```javascript
devServer: {
    contentBase: path.resolve(__dirname, '../dist'),
    open: true, // 打开浏览器
    hot: true, // 开启热更新
}
```
并添加插件
```javascript
const webpack = require(webpack)

...

plugins: [
    new webpack.HotModuleReplacementPlugin()
]
```
为使javascript模块也能实现热更新，需要在`index.js`文件底部加入
```javascript
if (module.hot) {
    module.hot.accept()
}
```
# 使用sass-loader
先下载`sass-loader`和`node-sass`
```
npm install sass-loader node-sass -D
```
然后在`module.rule`中加入
```javascript
{
    test: /\.scss$/,
    use: [{
            loader: "style-loader" // 将 JS 字符串生成为 style 节点
        }, {
            loader: "css-loader" // 将 CSS 转化成 CommonJS 模块
        }, {
            loader: "sass-loader" // 将 Sass 编译成 CSS
        }]
}
```
# 优化代码
我们可以将第三方库打包一遍，以后打包时没必要再打包一次，减少打包时间。
可以使用`autodll-webpack-plugin`插件
首先安装
```
npm install autodll-webpack-plugin -D
```
然后在`webpack.base.conf.js`中添加
```javascript
const AutoDllPlugin = require('autodll-webpack-plugin')

...

plugins: [
    ...
    new AutoDllPlugin({
        inject: true, // 自动将打包好的第三方库插入HTML
        debug: true,
        filename: '[name]_[hash],js',
        path: './dll', // 打包后的路径
        entry: {
            vendor: ['vue', 'vue-router', 'vuex'] // 要打包的库
        }
    }),
    ...
]
```
我们可以使用Webpack自带的splitChucksPlugin插件来提取公共代码
在`webpack.base.conf.js`中添加
```javascript
plugins: [
    ...
    new webpack.optimize.SplitChunksPlugin(), // 提取公共代码
    ...
]
```
# 提取CSS到单独文件
使用`mini-css-extract-plugin`
首先安装
```
npm install mini-css-extract-plugin -D
```
在生产环境打包时，将`vue-style-loader`改为MiniCssExtractPlugin.loader
最后修改plugins选项，插入
```javascript
new MiniCssExtractPlugin({
  filename: "[name].css",
  chunkFilename: "[id].css"
})
```
