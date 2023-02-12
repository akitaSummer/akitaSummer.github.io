---
title: webpack4概念及常用配置
date: 2020-06-07 19:56:25
tags:
	- Webpack
categories: 学习笔记
---

# webpack核心概念

## Entry
入口(Entry)指示webpack以哪个文件为入口起点开始打包，分析构建内部依赖图。

## Output
输出(Entry)指示webpack打包后的资源输出到哪里去，以及如何命名。

## Loader
Loader让webpack能够去处理那些非JavaScript的文件。

## Plugins
插件(Plugins)可以用于执行范围内更广的任务，如打包优化和压缩，重新定义环境中变量等。

# webpack基本配置

## 准备
首先我们先安装webpack
```
npm install webpack webpack-cli --save-dev
```
or 
```
yarn add webpack webpack-cli --dev
```
在我们日常开发的时候，会有开发环境和生产环境，不同的环境下，打包的配置也必然有相同点和不同点。
所以我们在创建配置文件时，使用`webpack.common.js`、`webpack.dev.js`和`webpack.prod.js`分别存放公用配置、开发环境打包配置和生产环境打包配置。

`webpack.common.js`
```javascript
// 基础打包配置
const path = require('path')

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
    plugins: []
}
```
`webpack.dev.js`
```javascript
// 开发环境配置
const merge = require('webpack-merge')
const path = require('path')
const baseConfig = require('./webpack.base.conf')

module.exports = merge(baseConfig, {
    mode: 'development', // 设置为开发环境
})
```
`webpack.prod.js`
```javascript
// 生产环境配置
const merge = require('webpack-merge')
const path = require('path')
const baseConfig = require('./webpack.base.conf')
const { CleanWebpackPlugin } = require('clean-webpack-plugin')

module.exports = merge(baseConfig, {
    mode: 'production', // 生产环境
})
```
并在`package.json`中配置好
```
"scripts": {
    "start": "webpack --config webpack.dev.js",
    "build": "webpack --env.production --config webpack.pord.js",
  },
```
这样我们就可以分别使用`yarn start`和`yarn build`运行开发和生产环境打包了
## entry
entry可以以单个文件作为入口
```javascript
entry: './src/js/entry.js'
```
也可以以多个文件为入口
```javascript
entry: {
      lodash: './src/js/lodash.js',
      main: './src/js/entry.js',
  }
```

## output
output常用配置如下
```javascript
 output: {
        path: path.resolve(__dirname, 'dist/js'), // 输出文件目录
        publicPath: "/", // script中src的前缀，可用于cdn，将前缀改为http://cdn.com.cn等
        filename: '[name]_[hash].[ext]', // 输出文件名称
        filename: 'bundle.js', // 使用contenthash，只有当源代码改变时，才会改变输出名，但在热更新情况下不能使用，可以做到重新打包代码上线的时候，用户只需要更新有变化的代码，而没有变化的代码，用户可以直接使用本地的缓存
        chunkFilename: "[name].chunk.js", // 非入口chunk名称，
        libraryTarget: 'umd', // 在制作库时需要，保证任何引用方式都能正确引用
        library: 'library', // 可以通过src标签引入，并且使用library作为全局变量，如果libraryTarget为this，则library会挂载到全局this
    },
```
## module
module是loader放置的位置，我们前端一般会需要解析css文件，图片文件，和字体文件
module常用配置如下
```javascript
 module: {
        rules: [{
                test: /\.(png|jpg|gif)$/,
                use: [{
                    loader: 'url-loader',
                    options: {
                        limit: 8192,
                        // placeholder：占位符
                        name: '[name].[ext]', // 打包文件后，文件名为原文件名
                        outputPath: 'images/'
                    }
                }]
            },
            {
                test: /\.(eot|ttf|svg|woff|woff2)$/,
                use: {
                    loader: 'file-loader', // 打包字体文件
                    force: 'pre', // 强制loader优于其他loader
                }
            },
            {
            test: /\.css$/,
            use: [
                'style-loader',
                'css-loader'
            ]
        }
        ]
    }
```
其中所有的loader都需要使用`yarn add --dev`安装，
当use存在多个loader时，执行顺序是从下到上。
## plugins
plugins可以在webpack运行到某个时刻的时候，帮你做一些事情,在我们通常情况下，会用到html-webpack-plugin和clean-webpack-plugin这两个plugin，他们分别用于自动生成html和每次打包时清除之前打包的文件
首先先安装两个plugin
```
yarn add html-webpack-plugin clean-webpack-plugin --dev
```
然后我们在根目录下创建一个`index.html文件`作为html-webpack-plugin的html模板
在`webpack.common.js`中配置
```javascript
const HtmlWebpackPlugin = require('html-webpack-plugin'); //自动生成html文件的插件
const { CleanWebpackPlugin } = require('clean-webpack-plugin'); //清除之前打包的文件


plugins: [ // plugins可以在webpack运行到某个时刻的时候，帮你做一些事情
        new HtmlWebpackPlugin({ template: './index.html' }),
        new CleanWebpackPlugin({
            root: path.resolve(__dirname, './'), // 设置CleanWebpackPlugin的根目录
        }),
    ]
```

## externals
externals是用于配置source-map的，source-map默认开启，是打包文件对原文件的映射，通过source-map，我们就可以知道源文件哪行出错
在`webpack.dev.js`中配置
```javascript
devtool: 'cheap-module-eval-source-map'
```
inline- 开头：将.map文件存至js文件最后一行 
cheap- 开头：简化提示，只告诉哪行出错 
module- 开头： 包含module的提示 
eval-是通过eval储存映射内容
在`weback.prod.js`中配置
```javascript
devtool: 'cheap-module-source-map'
```
不同的环境下，我们需要的source-map也不同，这里需要注意

## resolve
resolve是解析模块的规则，我们在这里可以进行配置解析的路径等功能
```javascript
resolve: {
        extensions: ['.js', '.jsx'], // 省略后缀，从左到右查找
        mainFile: ['index', 'child'], // 未写文件名，则优先加载的文件名
        alias: {
            path: path.resolve(__dirname, './src/') // 路径别名
        }
    }
```

# 常见需求配置
## 本地启动开发服务器进行开发
像react和vue使用yarn start时，启动一个本地服务器，我们通常需要安装`webpack-dev-server`
```
yarn add webpack-dev-server --dev
```
然后修改我们的`package.json`文件
```
"scripts": {
    ...
    "start": "webpack-dev-server --config webpack.dev.js",
    ...
  }
```
并在`webpack.dev.js`中配置devServer
```javascript
devServer: {
        contentBase: path.resolve(__dirname, '../dist'),
        open: true, // 打开浏览器
    }
```
也需要在plugins下添加`webpack.HotModuleReplacementPlugin`
```javascript
plugins: [
        new webpack.HotModuleReplacementPlugin(), // 热加载插件
    ]
```
但这样是不能完成的，我们还需要在js文件中添加如下代码
```javascript
if (module.hot) {
  module.hot.accept('./content', () => {
    // ./content.js 文件改变后，进行一些事情，例如更新 HMR热更新引入后才有用
  })
}
```
在react和vue中，因为他们的loader内部集成了这样的代码，所以无需写入上面的代码，也能实现热更新
当然，你可以配置更多选项，让你的本地服务器更加完善，常用配置如下
```javascript
devServer: {
        contentBase: './dist', // 打包文件位置
        open: true, // 自动打开浏览器
        // historyApiFallback: true, // 解决单页面路由browser后端问题
        historyApiFallback: {
            rewrites: [{
                from: /abc.html/,
                to: '/views/landing.html' // 也可传入function（content） {} 结合逻辑动态返回路径
            }]
        },
        proxy: {
            // '/api': 'http://localhost:3000', // 配置跨域代理
            '/api': {
                target: 'http://loaclhost:3000',
                secure: false, // 对https网址的请求转发
                pathRewrite: {
                    'header.json': 'demo.json' // 请求header.json文件会返回demo.json文件
                },
                bypass: function (req, res, proxyOptions) { // 根据特定请求，返回特定内容
                    if (req.header.accept.indexOf('html')) {
                        return '/index.html'
                    }
                },
                changeOrigin: true, // 突破Origin限制
                overlay: true, // eslint检测出错时，会弹出页面提醒
            }
        },
        // port: 8888, // 配置端口
        hot: true, // 热模块加载
        hotOnly: true， // 即使hot未生效，也不刷新页面
        clientLogLevel: 'none', // 不需要显示启动服务器日志信息
        quiet: true, // 除了一些基本的配置以外，其他的内容都不要显示
        overlay: false, // 如果出错了，不需要全局显示
    }
```
配置完毕后，在控制台输入`yarn start`就可以启动了

## 解析ES6语法
为了解析ES6及以上语法，需要安装`babel-loader`和`@babel/perset-env`
然后在`webpack.common.js`中添加配置
```javascript
module: {
        rules: [
            ......
            {
                test: /\.js$/,
                exclude: /node_modules/, // 第三方库除外
                use: [{
                        loader: 'babel-loader', // 配置babel-loader
                        query: {
                            presets: [
                                ["@babel/preset-env", { // es6语法转换es5语法
                                    useBuiltIns: "usage", // 根据业务代码，添加@babel/polyfill兼容的内容
                                    targets: {
                                        chrome: "67", // 运行的环境
                                    }
                                }],
                            ]
                        }
                    }
                ],
            },
            ......
        ]
    }
```
babel只负责语法转换，比如将ES6的语法转换成ES5。但如果有些对象、方法，浏览器本身不支持，比如：
- 全局对象：Promise、WeakMap 等。
- 全局静态函数：Array.from、Object.assign 等。
- 实例方法：比如 Array.prototype.includes 等。
此时，需要引入babel-polyfill来模拟实现这些对象、方法。
所以，在query里面配置了useBuiltIns，需要安装`@babel/polyfill`。
并且要在你的入口文件中引入
```javascript
import '@babel/polyfill' // 引入babel/polyfill，用于兼容低版本浏览器
```
但是`@babel/polyfill`会污染全局环境，制作库时，需要使用`@babel/plugin-transform-runtime`
`@babel/plugin-transform-runtime`的配置较为简单，只需要在query中配置plugins
当然，你也可以在其中配置`dynamic-import-webpack`支持动态import
```javascript
module: {
        rules: [
            ......
            {
                test: /\.js$/,
                exclude: /node_modules/, // 第三方库除外
                use: [{
                        loader: 'babel-loader', // 配置babel-loader
                        query: {
                            presets: [
                                ["@babel/preset-env", { // es6语法转换es5语法
                                    useBuiltIns: "usage", // 根据业务代码，添加@babel/polyfill兼容的内容
                                    targets: {
                                        chrome: "67", // 运行的环境
                                    }
                                }],
                            ],
                            plugins: [
                                ["@babel/plugin-transform-runtime", { // @babel/polyfill会污染全局环境，制作库时，需要使用@babel/plugin-transform-runtime
                                    "corejs": 2,
                                    "helpers": true,
                                    "regenerator": true,
                                    "useESModules": false,
                                }],
                                "dynamic-import-webpack", // 动态import
                            ]
                        }
                    }
                ],
            },
            ......
        ]
    }
```
babel的配置有很多，详情可去官网查询，但是如果配置过多，我们可以将其单独抽取出来，在根目录下新建一个.babelrc文件，用于储存babel的配置
```json
{
  "plugins": [
    [
      "@babel/plugin-transform-runtime", {
      "corejs": 2,
      "helpers": true,
      "regenerator": true,
      "useESModules": false
    }],
    "@babel/plugin-transform-react-constant-elements"
  ]
}
```

## Tree Shaking
Tree Shaking 是一个术语，通常用于描述移除javascript上下文中的未引用代码。
Tree Shaking 在production模式下是默认开启的，但是在development模式下，就需要我们主动开启。
在`webpack.dev.js`中添加
```javascript
optimization: {
        usedExports: true, // development 开发环境下，设置Tree shaking
    }
```
需要注意的是Tree shaking, 只支持ES Module，即静态引用，js文件中不要使用commonjs引入方法。
并且当你引入css文件时，会写如下代码
```javascript
import '../css/test.css'
```
这样的代码Tree Shaking会默认将其移除，所以我们需要在`package.json`中添加配置
```json
"sideEffects": [
    "*.css"
  ]
```
这样就可以防止css文件被Tree Shaking默认移除了。

## 代码分割(Code Splitting)
代码分割可以将代码分割到不同的打包文件，代码分割可以控制资源加载优先级，如果使用合理，会极大影响加载时间。
如果想要开启代码分割，则可以在`webpack.common.js`中配置
```javascript
optimization: {
        splitChunks: {
            chunks: "all", // 代码分割
            minSize: 30000, // 对代码分割的最小文件大小
            maxSize: 0, // 代码分割文件最大大小
            minChunks: 1, // 代码分割文件至少被chunk或作为chunk引入的次数
            maxAsyncRequests: 5, // 总共最多分割成多少代码分割文件
            maxInitialRequests: 3, // 入口文件进行分割时最多分割成多少文件
            automaticNameDelimiter: "~", // 默认情况下，webpack将使用块的来源和名称生成名称, automaticNameDelimiter用于修改连接符
            name: true, // 拆分模块名称，true会根据模块和缓存键值对自动选择一个名称
            cacheGroups: { // 决定拆分分割模块分割至哪个文件中，缓存分组
                vendors: {
                    test: /[\\/]node_modules[\\/]/, // 模块保存的分割代码
                    priority: -10, // 打包到此文件的优先级
                    filename: 'vendors.js' // 模块名字
                },
                // default: false, // 默认分割文件放置地址
                default: {
                    priority: -20,
                    reuseExistingChunk: true, // 是否复用之前打包过的模块
                    filename: 'common.js'
                }
            }
        }
    }
```
这只是同步代码的分割，当你进行了3.2中的`dynamic-import-webpack`配置后，你的动态引用都会自动进行代码分割。

## css的代码分割
css文件也可以进行代码分割，我们需要安装`mini-css-extract-plugin`和`optimize-css-assets-webpack-plugin`这个两个插件，但是由于热加载目前不支持`mini-css-extract-plugin`，所以我们需要在`webpack.prod.js`中配置
```javascript
const MiniCssExtractPlugin = require('mini-css-extract-plugin') // 将css从js中抽离出来，单独作为css文件引入，HMR目前不支持
const OptimizeCSSAssetsPlugin = require('optimize-css-assets-webpack-plugin') // css文件压缩

module.exports = merge(baseConfig, {
  mode: 'production', // 生产环境
  devtool: 'cheap-module-source-map',
  module: {
      rules: [
      {
          test: /\.css$/,
          use: [
          MiniCssExtractPlugin.loader, // 将style-loader更换为MiniCssExtractPlugin.loader
          'css-loader'
          ]
      },
      {
          test: /\.scss$/,
          use: [
          MiniCssExtractPlugin.loader,
          {
              loader: "css-loader", // 若要配置loader，需要使用对象写法
              options: {
              importLoaders: 2, // 通过import引入的css文件需要再走下面两个loader
              modules: true, // 模块化css，通过style.xxx定义class
              }
          },
          'sass-loader', // webpack在打包时是有顺序的，由下至上，由右至左
          'postcss-loader', // 为CSS添加-webkit等兼容开头
          ]
      },
      ]
  },
  plugins: [
      new MiniCssExtractPlugin({
          filename: '[name].css',
          chunkFilename: '[name].chunk.css'
      }),
  ],
  optimization: {
      minimizer: [
      new OptimizeCSSAssetsPlugin({})
      ]
  }
})
```
这样就完成了css的代码分割。

## 环境变量
我们可以在`package.json`中通过--来配置打包时的环境变量
```json
"scripts": {
    ...
    "start": "webpack-dev-server --config webpack.common.js",
    "build": "webpack --env.production --config webpack.common.js",
    ...
  },
```
这样我们可以在`webpack.common.js`中获得这个变量并进行一定的代码逻辑，例如打包时我们就可以根据环境变量来判断是什么环境
```javascript
module.exports = (env) => {
    if (env && env.production) {
        return merge(commonConfig, prodConfig)
    } else {
        return merge(commonConfig, devConfig)
    }
}
```
个人还是推荐使用`webpack.common.js`、`webpack.dev.js`和`webpack.prod.js`分别存放公用配置、开发环境打包配置和生产环境打包配置。

## typescript
typescript配置较为简单，首先需要安装typescript，并运行`tsc --init`创建`tsconfig.json`来配置ts，
然后安装`ts-loader`让webpack能配置ts文件。
```javascript
module: {
        rules: [
            ...
            {
                test: /\.tsx?$/,
                use: 'ts-loader',
                exclude: /node_nodules/
            }
            ...
        ]
    },
```

## ESLint
ESLint是一个插件化的javascript代码检测工具，使用ESLint可以让我们在协作开发时大家的代码更加规范化。
首先我们先要下载eslint
```
yarn add eslint --dev
```
然后我们运行`npx eslint --init`,进行eslint的基础配置。
当配置完成后会有`.eslintrc.js`文件
```javascript
module.exports = {
  "extends": 'airbnb',
  "parser": "babel-eslint",
  "rules": { // 配置自己的规范
    "react/jsx-filename-extension": 0
  },
  globals: { // 设置可用全局变量
    document: false,
  }
}
```
最后不要忘记在webpack中加上eslint
```javascript
module: {
        rules: [
            ...
            {
                test: /\.js$/,
                exclude: /node_modules/, // 第三方库除外
                use: [
                    ...
                    {
                        loader: "eslint-loader", // 配置eslint在打包时检测
                        options: {
                            fix: true // 简单问题自动修复
                        },
                    }
                ],
            },
            ...
        ]
    }
```

## webpack打包速度优化
1. 跟上技术的迭代(webpack/node/npm/yarn/...)
尽可能使用新版本的工具，因为新版本中做了更多的优化
2. 在尽可能少的模块上应用 loader
通过配置 exclude / include 减少 loader 的使用
3. Plugin 尽可能精简并确保可靠
尽量使用官方推荐的插件，官方的优化的更好
4. resolve 参数合理配置
合理的配置可减少查找匹配的次数，降低性能损耗
5. 控制包文件大小
不要引入一些未使用的模块包
配置 Tree-Shaking，打包时，不打包一些引入但未使用的模块
配置 splitChunks，对代码进行合理拆分，将大文件拆成小文件打包
6. thread-loader/parallel-webpack/happypack 多进程打包
webpack 默认是通过 nodeJs 来运行的，是单进程的打包
可以使用 thread-loader / parallel-webpack / happypack 这些技术，配置多进程打包
7. 合理使用 sourceMap
打包生成的 sourceMap 越详细，打包的速度就越慢，可根据不同的环境配置不同的 sourceMap
8. 结合 stats 分析打包结果
根据打包分析的结果，做对应的优化
9. 开发环境内存编译
开发环境使用 webpack-dev-server，启动服务后，会将编译生成的文件放到内存中，而内存的读取速度远远高于硬盘的读取速度，可以让我们在开发环境中，webpack 性能得到很大的提升
10. 开发环境无用插件剔除
例如：开发环境无需对代码进行压缩等

# 底层原理简单剖析
## 自己编写一个Loader
实际上 loader 就是一个函数，这个函数可接收一个参数，这个参数指的就是引入文件的源代码，但是这个函数不能写成箭头函数，因为要用到 this，webpack 在调用 loader 时，会把这个 this 做一些变更，变更之后，才能用 this 里面的方法，如果写成箭头函数，this 指向就会有问题
假如我们有个`index.js`文件需要被打包
```javascript
console.log('webpack-loader，akita')
```
我们编写自己的loader
```javascript
module.exports = function(source) { // loader 不能用箭头函数 source为导入的文件源代码
  return source.replace('akita', 'akitaSummer')
}
```
然后在webpack中配置
```javascript
module: {
    rules: [
      {
        test: /\.js/,
        use: [{
          loader: path.resolve(__dirname, './loaders/replaceLoader.js')
        }
        ]
      }
    ]
  },
```
这样我们就完成了一个简单地loader。
不光可以使用`return`来返回处理后的文件源代码，我们也可以使用`this.callback`来处理返回后的源代码，通过`this.async()`我们也可以进行异步处理。
```javascript
module.exports = function(source) { // loader 不能用箭头函数 source为导入的文件源代码
// return source.replace('akita', 'akitaSummer')
  const result = source.replace('akitaSummer', 'akitaSummer')

  this.callback(null, result) // 可以用callback进行处理

  // const callback = this.async()
  // setTimeout(() => {
  //   callback(result) // 异步处理
  // }, 1000)

  // this.callback(
//   err: Error | null,
//   content: string | Buffer,
//   sourceMap?: SourceMap,
//   meta?: any
// )
}
```
通常情况下，webpack中可以传递option参数，如：
```javascript
module: {
    rules: [
      {
        test: /\.js/,
        use: [{
          loader: path.resolve(__dirname, './loaders/replaceLoader.js'),
          options: {
            name: 'akita'
          }
        }
        ]
      }
    ]
  },
```
我们在自己的loader中，就需要使用`this.query`来获得option。
webpack中推荐使用`loader-utils`来获得option。
```javascript
const loaderUtils = require('loader-utils') // 官方推荐的query获取方法

module.exports = function(source) { // loader 不能用箭头函数 source为导入的文件源代码
  // console.log(this.query) // this.query 是loader中option传递的参数
  const options = loaderUtils.getOptions(this) // loader-utils会自动将this中query传递到options中
  // return source.replace('akita', options.name)
  const result = source.replace('akitaSummer', options.name)

  this.callback(null, result) // 可以用callback进行处理

  // const callback = this.async()
  // setTimeout(() => {
  //   callback(result) // 异步处理
  // }, 1000)
}

// this.callback(
//   err: Error | null,
//   content: string | Buffer,
//   sourceMap?: SourceMap,
//   meta?: any
// )
```
最后，如果我们需要在配置中不用丑陋的`path.resolve(__dirname, './loaders/replaceLoader.js')`的话，我们可以配置`resolveLoader`
```javascript
resolveLoader:  {
    modules: ['node_modules', './loaders'] // 未写路径时，loader查找的目录
  },
  module: {
    rules: [
      {
        test: /\.js/,
        use: [{
          // loader: path.resolve(__dirname, './loaders/replaceLoader.js'),
          loader: 'replaceLoader',
          options: {
            name: 'akita'
          }
        }
        ]
      }
    ]
  }
```

## 自己编写一个Plugin
Plugin是一个类，所以我们的plugin用一个类来表示。
```javascript
class CoyprightWebpackPlugin { // 插件是一个类
    constructor(options) { // options是webpack中配置的参数的参数
        console.log('plugin 被使用了', option.name)
    }
    apply(compiler) { // webpack会调用apply方法 compiler是webpack实例
    }
}

module.exports = CoyprightWebpackPlugin
```
webpack配置较为简单，我们只需引入即可
```javascript
const CopyRightWebpackPlugin = require('./plugins/coypright-webpack-plugin')

module.exports = {
  .....
  plugins: [
    new CopyRightWebpackPlugin({name: 'akitaSummer'})
  ]
}
```
`apply`中`compiler`提供了打包时刻，你可以在不同的打包时刻进行一定的操作
```javascript
class CoyprightWebpackPlugin { // 插件是一个类
    constructor(options) {
        console.log('plugin 被使用了')
    }
    apply(compiler) { // webpack会调用apply方法 compiler是webpack实例
        compiler.hooks.emit.tapAsync('CoyprightWebpackPlugin', (compilation, callback) => { //emit 是在文件被放置于dist目录时调用
            // compilation 存放此次打包内容， compiler存放所有打包内容
            console.log(compilation.assets) // assets中存放了打包结果
            compilation.assets['copyright.txt'] = {
                source: function() { // 内容
                    return 'copyright'
                },
                size: function() { // 大小
                    return 9
                }
            }
            callback() // 必须调用
        })
        compiler.hooks.compile.tap('CoyprightWebpackPlugin', (compilation) => {
            // compile 是同步函数
            console.log('compile')
        })
    }
}

module.exports = CoyprightWebpackPlugin
```
其它的打包时刻
- done (异步时刻)表示打包完成
- compile (同步时刻)
- 同步时刻是 tap 且没有 callback
- 异步时刻是 tapAsync 有 callback

## 自己编写一个Bundler
首先我们先理一下思路，如果需要编写一个打包文件，需要能解析文件，能遍历出所有的文件，能将所有的解析文件进行合并。
那我们就一步步完成这三个功能。
首先我们先创建需要打包的文件。
```javascript
// /src/word.js
export const word = 'hello'


// /src/message.js
import { word } from './word.js'
// 需要写 .js 后缀，因为没有使用 webpack

const message = `say ${word}`

export default message


// /src/index.js
import message from './message.js'

console.log(message)
```
接下来我们新建一个`bundler.js`作为我们的打包工具。
在解析文件时，我们需要`@babel/parser`和`@babel/traverse`来将文件源码转化成抽象语法树，并解析，最后在使用`@babel/core`将抽象语法树转化为代码
```javascript
const fs = require('fs')
const path = require('path')
const parser = require('@babel/parser') // 将文件源码转化为抽象语法树
const traverse = require('@babel/traverse').default // 默认是es6导出
const babel = require('@babel/core') // 转换代码

const moduleAnalyser = (filename) => {
    const content = fs.readFileSync(filename, 'utf-8') // 读取文件
    const ast = parser.parse(content, { // 将文件源码转化为抽象语法树
        sourceType: 'module' // es6引用时配置
    })
    const dependencies = {}
    traverse(ast, { // 接收抽象语法树并解析
        ImportDeclaration({ node }) { // 当节点为ImportDeclaration时
            const dirname = path.dirname(filename)
            const newFile = './' + path.join(dirname, node.source.value) // 处理文件路径
            dependencies[node.source.value] = newFile // 将import结点储存于对象中
        }
    })
    const { code } = babel.transformFromAst(ast, null, { // transformFromAst方法可将抽象语法树转化为代码
        presets: ['@babel/preset-env'] // es6 转 es5
    })
    return {
        filename,
        dependencies,
        code
    }
    // console.log(ast.program.body)
}
```
这样我们的第一步就完成了，接下来我们需要将所有的文件全部解析。
```javascript
const makeDependenciesGraph = (entry) => { // 依赖图谱
    const entryModule = moduleAnalyser(entry) // 得到入口文件的详情
    const graphArray = [entryModule]
    for (let i = 0; i < graphArray.length; i++) {
        const item = graphArray[i]
        const { dependencies } = item // 得到文件的依赖
        if (dependencies) {
            for (let j in dependencies) {
                graphArray.push(moduleAnalyser(dependencies[j])) // 将依赖分析后添加进graphArray里
            }
        }
    }
    const graph = {}
    graphArray.forEach(item => { // 转换数据结构
        graph[item.filename] = {
            dependencies: item.dependencies,
            code: item.code
        }
    })
    return graph
}
```
最后我们将所有解析完毕的文件进行打包整合
```javascript
const generateCode = (entry) => {
    const graph = JSON.stringify(makeDependenciesGraph(entry)) // 放置模板字符串会将对象转化为[Object Object]
    return `
    ;(function(graph) { // 使用闭包，防止污染全局
        function require(module) { // 自己构造require函数
            function localRequire(relativePath) { // 将相对路径转换成真实路径后调用
                return require(graph[module].dependencies[relativePath])
            }
            var exports = {} // 内部需要exports对象
            ;(function (require, exports, code) {// 使用闭包，防止污染全局
                eval(code) // graph[module].code 中有require函数，会递归调用，不断引入
            })(localRequire, exports, graph[module].code)
            return exports
        }
        require('${entry}')
    })(
        ${graph}
    )
    `
}
```
全部代码
```javascript
const fs = require('fs')
const path = require('path')
const parser = require('@babel/parser') // 将文件源码转化为抽象语法树
const traverse = require('@babel/traverse').default // 默认是es6导出
const babel = require('@babel/core') // 转换代码

const moduleAnalyser = (filename) => {
    const content = fs.readFileSync(filename, 'utf-8')
    const ast = parser.parse(content, {
        sourceType: 'module' // es6引用时配置
    })
    const dependencies = {}
    traverse(ast, { // 接收抽象语法树并解析
        ImportDeclaration({ node }) { // 当节点为ImportDeclaration时
            const dirname = path.dirname(filename)
            const newFile = './' + path.join(dirname, node.source.value) // 处理文件路径
            dependencies[node.source.value] = newFile // 将import结点储存于对象中
        }
    })
    const { code } = babel.transformFromAst(ast, null, { // transformFromAst方法可将抽象语法树转化为代码
        presets: ['@babel/preset-env'] // es6 转 es5
    })
    return {
        filename,
        dependencies,
        code
    }
    // console.log(ast.program.body)
}

const makeDependenciesGraph = (entry) => { // 依赖图谱
    const entryModule = moduleAnalyser(entry) // 得到入口文件的详情
    const graphArray = [entryModule]
    for (let i = 0; i < graphArray.length; i++) {
        const item = graphArray[i]
        const { dependencies } = item // 得到文件的依赖
        if (dependencies) {
            for (let j in dependencies) {
                graphArray.push(moduleAnalyser(dependencies[j])) // 将依赖分析后添加进graphArray里
            }
        }
    }
    const graph = {}
    graphArray.forEach(item => { // 转换数据结构
        graph[item.filename] = {
            dependencies: item.dependencies,
            code: item.code
        }
    })
    return graph
}

const generateCode = (entry) => {
    const graph = JSON.stringify(makeDependenciesGraph(entry)) // 放置模板字符串会将对象转化为[Object Object]
    return `
    ;(function(graph) { // 使用闭包，防止污染全局
        function require(module) { // 自己构造require函数
            function localRequire(relativePath) { // 将相对路径转换成真实路径后调用
                return require(graph[module].dependencies[relativePath])
            }
            var exports = {} // 内部需要exports对象
            ;(function (require, exports, code) {// 使用闭包，防止污染全局
                eval(code) // graph[module].code 中有require函数，会递归调用，不断引入
            })(localRequire, exports, graph[module].code)
            return exports
        }
        require('${entry}')
    })(
        ${graph}
    )
    `
}

const code = generateCode('./src/index.js')
console.log(code)
```