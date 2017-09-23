---
title: vue-webpack-boilerplate源码简单分析
date: 2017-09-23 21:50:33
tags: vue
---
[2]:/images/170923/2.png

这篇文章会告诉你

* 什么是[vue-webpack-boilerplate](https://github.com/vuejs-templates/webpack)及用途
* 即当通过vue-cli创建一个webpack模块，通过npm启动项目的大致流程

它不会告诉你

* 什么是vue，webpack

<!--more-->

在创建vue项目时，官方推荐使用vue-cli这个命令行工具。原文是这么说的

```javascript
# 全局安装 vue-cli
$ npm install --global vue-cli
# 创建一个基于 webpack 模板的新项目
$ vue init webpack my-project
# 安装依赖，走你
$ cd my-project
$ npm install
$ npm run dev;           
```

执行npm run dev 会运行webpack。大致会编译vue格式文件，热加载，eslint语法检测等功能。开发时过程带给我们非常省时省力的体验。

下面讲下当执行npm run dev后都发生了什么。

首先当通过vue-cli创建vue项目。执行的vue init webpack my-project

实际拷贝的是 https://github.com/vuejs-templates/webpack 中template提供的配置文件模版。他给我们提供了vue项目所需的架子并整合了webpack相关配置。

我们熟悉的build, src, config等目录都里面。

通过 package.json 可以看出有四个命令。

开发时要运行 npm run dev 或 npm run start 或 npm start

线上及生产环境要运行 npm run build

检测JS语法运行 npm run lint
```javascript
  "scripts": {
    "dev": "node build/dev-server.js",
    "start": "node build/dev-server.js",
    "build": "node build/build.js",
    "lint": "eslint --ext .js,.vue src"
  },         
```


npm run dev实际执行的就是 node build/dev-server.js

打开dev-server.js，第一行是 require('./check-versions')()  先分析这个文件

### check-versions.js

大致就是将当前运行的node和npm环境与package.json中的engines中node和npm要求的最低版本进行对比，看是否满足需求。


```javascript
// 不错的node模块，使输出内容带前景色和背景色。
var chalk = require('chalk')
// 语义化版本，详见http://semver.org/lang/zh-CN/
// https://www.npmjs.com/package/semver
var semver = require('semver')
var packageConfig = require('../package.json')
// 顾名思义，提供了一些和shell命令名字功能都一样的方法
var shell = require('shelljs')
function exec (cmd) {
  return require('child_process').execSync(cmd).toString().trim()
}

var versionRequirements = [
  {
    name: 'node',
    // semver.clean('  =v1.2.3   ') => '1.2.3'
    currentVersion: semver.clean(process.version),
    versionRequirement: packageConfig.engines.node
  },
]

if (shell.which('npm')) {
  versionRequirements.push({
    name: 'npm',
    currentVersion: exec('npm --version'),
    versionRequirement: packageConfig.engines.npm
  })
}

module.exports = function () {
  var warnings = []
  for (var i = 0; i < versionRequirements.length; i++) {
    var mod = versionRequirements[i]
    if (!semver.satisfies(mod.currentVersion, mod.versionRequirement)) {
      warnings.push(mod.name + ': ' +
        chalk.red(mod.currentVersion) + ' should be ' +
        chalk.green(mod.versionRequirement)
      )
    }
  }

  if (warnings.length) {
    console.log('')
    console.log(chalk.yellow('To use this template, you must update following to modules:'))
    console.log()
    for (var i = 0; i < warnings.length; i++) {
      var warning = warnings[i]
      console.log('  ' + warning)
    }
    console.log()
    process.exit(1)
  }
}       
```
打开package.json，你可以将
```
"engines": {
  "node": ">= 4.0.0",
  "npm": ">= 3.0.0"
},
```
改为`"npm": ">= 6.0.0"`

然后运行 npm run dev 就会输出类似的错误
![param][2]

### config/index.js

接着是var config = require('../config')

这里定义了一些开发环境和生产环境中不同的环境参数以及供后面的webpack配置文件使用的参数。

举个例子，为什么运行npm run dev会自动打开浏览器并且地址端口是8080。就是在这里面定义的。

更详细的参见：
https://github.com/vuejs-templates/webpack/blob/develop/docs/env.md

开发过程中可能需要调用后端的接口，如果要设置代理。

参见：
https://github.com/vuejs-templates/webpack/blob/develop/docs/proxy.md

本质上用到一个叫 http-proxy-middleware http代理中间件的node模块

### env.js

dev.env.js 和 prod.env.js 里面放一些配置变量

比如我添加了一个KEY_ID

```javascript
module.exports = merge(prodEnv, {
  NODE_ENV: '"development"',
  KEY_ID: '123456'
})
```

在vue文件里就可以用process.env.KEY_ID 输出 123456

为什么? 原理是用到了webpack提供的 DefinePlugin 插件

```javascript
  plugins: [
    new webpack.DefinePlugin({
      'process.env': config.dev.env
    }),
```

注意字符串要写两个引号。
回到dev-server.js，接着分析webpack配置文件。这类的文章很多了，简要分析下

### webpack.dev.conf

```javascript
// 工具方法
var utils = require('./utils')
var webpack = require('webpack')
var config = require('../config')
// webpack官方提供的一个webpack配置文件合并工具
var merge = require('webpack-merge')
// 开发时和发布时公用的配置文件信息
var baseWebpackConfig = require('./webpack.base.conf')
// 一个webpack插件，生成入口html文件，最强大的功能是动态自动插入webpack生成的css和js资源文件
// 一般的单页面应用只有一个主html文件比如index.html，webpack打包生成js和css文件后，
// 我们当然可以在手动写标签引入这些资源文件，但有时候生成后的文件带有hash。如app-3d31retr.js。这时需要自动引入，并替换html。
var HtmlWebpackPlugin = require('html-webpack-plugin')
// 不解释，自行去github查文档
var FriendlyErrorsPlugin = require('friendly-errors-webpack-plugin')

// add hot-reload related code to entry chunks
Object.keys(baseWebpackConfig.entry).forEach(function (name) {
  baseWebpackConfig.entry[name] = ['./build/dev-client'].concat(baseWebpackConfig.entry[name])
})

module.exports = merge(baseWebpackConfig, {
  module: {
    rules: utils.styleLoaders({ sourceMap: config.dev.cssSourceMap })
  },
  // cheap-module-eval-source-map is faster for development
  devtool: '#cheap-module-eval-source-map',
  plugins: [
    new webpack.DefinePlugin({
      'process.env': config.dev.env
    }),
    // https://github.com/glenjamin/webpack-hot-middleware#installation--usage
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NoEmitOnErrorsPlugin(),
    // https://github.com/ampedandwired/html-webpack-plugin
    new HtmlWebpackPlugin({
      filename: 'index.html',
      template: 'index.html',
      inject: true
    }),
    new FriendlyErrorsPlugin()
  ]
})
```

### webpack.base.conf

主要讲下resolve部分，webpack把各种类型资源的文件当做module来处理，并通过相应的loader编译成js可运行的代码。

在vueLoaderConfig里已经定义了各种style loader了。

为什么在vue中的style中定义 lang="stylus" 或  lang="less"  ?
答：就能直接写相应的预处理语言啦。因为上面的webpack.base中已经把各种样式loader都定义好啦

为什么可以在vue类型类型里用 `<template>`, `<script>` 和 `<style>` ?
答：这是vue-loader的功劳。当然也离不开webpack。
(详见)[https://vue-loader.vuejs.org/zh-cn/start/spec.html]


### dev-server.js

```javascript
// 检测是否满足版本需要
require('./check-versions')()
// 引入外层config中的index.js，分为运行时和开发时的变量信息
// 我们可以自己添加一些变量
var config = require('../config')
if (!process.env.NODE_ENV) {
  // 最终变成 process.env.NODE_ENV = JSON.parse('"development"')
  process.env.NODE_ENV = JSON.parse(config.dev.env.NODE_ENV)
}

// 第三方模块。可调用系统默认的程序打开图片，地址等，如 opn('http://localhost:8080'); open('./demo.jpg')
var opn = require('opn')
// node内置的核心模块
var path = require('path')
// node框架，在这里用来启动webserver
var express = require('express')
// 不解释，核心编译工具
// 官方文档中不推荐全局安装webpack，然后执行全局变量的方式
// 这里在nodejs里引入webpack，用提供的api来配置并启动
var webpack = require('webpack')
// http代理中间件，可转发api等
var proxyMiddleware = require('http-proxy-middleware')
var webpackConfig = require('./webpack.dev.conf')
```

### 最终的webpack.dev.conf.js
```javascript
{
  entry: {
    app: ['./build/dev-client', './src/main.js']
  },
  output: {
    path: 'D:\\projects\\Vue\\vue-music-player\\dist',
    filename: '[name].js',
    publicPath: '/'
  },
  resolve: {
    extensions: ['.js', '.vue', '.json'],
    alias: {
      'vue$': 'vue/dist/vue.esm.js',
      '@': 'D:\\projects\\Vue\\vue-music-player\\src',
      common: 'D:\\projects\\Vue\\vue-music-player\\src\\common',
      components: 'D:\\projects\\Vue\\vue-music-player\\src\\components',
      base: 'D:\\projects\\Vue\\vue-music-player\\src\\base',
      api: 'D:\\projects\\Vue\\vue-music-player\\src\\api'
    }
  },
  module: {
    rules: [{
      test: /\.vue$/,
      loader: 'vue-loader',
      options: {
        loaders: {
          css: ['vue-style-loader', {
            loader: 'css-loader',
            options: {
              minimize: false,
              sourceMap: false
            }
          }],
          postcss: ['vue-style-loader', {
            loader: 'css-loader',
            options: {
              minimize: false,
              sourceMap: false
            }
          }],
          less: ['vue-style-loader', {
            loader: 'css-loader',
            options: {
              minimize: false,
              sourceMap: false
            }
          },
          {
            loader: 'less-loader',
            options: {
              sourceMap: false
            }
          }],
          sass: ['vue-style-loader', {
            loader: 'css-loader',
            options: {
              minimize: false,
              sourceMap: false
            }
          },
          {
            loader: 'sass-loader',
            options: {
              indentedSyntax: true,
              sourceMap: false
            }
          }],
          scss: ['vue-style-loader', {
            loader: 'css-loader',
            options: {
              minimize: false,
              sourceMap: false
            }
          },
          {
            loader: 'sass-loader',
            options: {
              sourceMap: false
            }
          }],
          stylus: ['vue-style-loader', {
            loader: 'css-loader',
            options: {
              minimize: false,
              sourceMap: false
            }
          },
          {
            loader: 'stylus-loader',
            options: {
              sourceMap: false
            }
          }],
          styl: ['vue-style-loader', {
            loader: 'css-loader',
            options: {
              minimize: false,
              sourceMap: false
            }
          },
          {
            loader: 'stylus-loader',
            options: {
              sourceMap: false
            }
          }]
        }
      }
    },
    {
      test: /\.js$/,
      loader: 'babel-loader',
      include: ['D:\\projects\\Vue\\vue-music-player\\src', 'D:\\projects\\Vue\\vue-music-player\\test']
    },
    {
      test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
      loader: 'url-loader',
      options: {
        limit: 10000,
        name: 'static/img/[name].[hash:7].[ext]'
      }
    },
    {
      test: /\.(woff2?|eot|ttf|otf)(\?.*)?$/,
      loader: 'url-loader',
      options: {
        limit: 10000,
        name: 'static/fonts/[name].[hash:7].[ext]'
      }
    },
    {
      test: /\.css$/,
      use: ['vue-style-loader', {
        loader: 'css-loader',
        options: {
          minimize: false,
          sourceMap: false
        }
      }]
    },
    {
      test: /\.postcss$/,
      use: ['vue-style-loader', {
        loader: 'css-loader',
        options: {
          minimize: false,
          sourceMap: false
        }
      }]
    },
    {
      test: /\.less$/,
      use: ['vue-style-loader', {
        loader: 'css-loader',
        options: {
          minimize: false,
          sourceMap: false
        }
      },
      {
        loader: 'less-loader',
        options: {
          sourceMap: false
        }
      }]
    },
    {
      test: /\.sass$/,
      use: ['vue-style-loader', {
        loader: 'css-loader',
        options: {
          minimize: false,
          sourceMap: false
        }
      },
      {
        loader: 'sass-loader',
        options: {
          indentedSyntax: true,
          sourceMap: false
        }
      }]
    },
    {
      test: /\.scss$/,
      use: ['vue-style-loader', {
        loader: 'css-loader',
        options: {
          minimize: false,
          sourceMap: false
        }
      },
      {
        loader: 'sass-loader',
        options: {
          sourceMap: false
        }
      }]
    },
    {
      test: /\.stylus$/,
      use: ['vue-style-loader', {
        loader: 'css-loader',
        options: {
          minimize: false,
          sourceMap: false
        }
      },
      {
        loader: 'stylus-loader',
        options: {
          sourceMap: false
        }
      }]
    },
    {
      test: /\.styl$/,
      use: ['vue-style-loader', {
        loader: 'css-loader',
        options: {
          minimize: false,
          sourceMap: false
        }
      },
      {
        loader: 'stylus-loader',
        options: {
          sourceMap: false
        }
      }]
    }]
  },
  devtool: '#cheap-module-eval-source-map',
  plugins: [
    DefinePlugin {
      definitions: {
        'process.env': {
          NODE_ENV: '"development"'
        }
      }
    },
    HotModuleReplacementPlugin {
      multiStep: undefined,
      fullBuildTimeout: 200
    },
    NoEmitOnErrorsPlugin {},
    HtmlWebpackPlugin {
      options: {
        template: 'index.html',
        filename: 'index.html',
        hash: false,
        inject: true,
        compile: true,
        favicon: false,
        minify: false,
        cache: true,
        showErrors: true,
        chunks: 'all',
        excludeChunks: [],
        title: 'Webpack App',
        xhtml: false
      }
    },
    FriendlyErrorsWebpackPlugin {}
  ]
}

```


### 总结

开发用的webpack配置文件是 webpack.dev.conf.js + webpack.base.conf.js合并后的结果
主要实现热加载，友好的错误输出，方便调试等功能

生产环境用到的是webpack.prod.conf.js + webpack.base.conf.js合并后的结果
主要实现打包压缩css和js，不输出警告和错误
参见：https://vue-loader.vuejs.org/zh-cn/workflow/production.html


### 参考
https://github.com/vuejs-templates/webpack/blob/develop/docs/README.md
https://vue-loader.vuejs.org/zh-cn/
https://cn.vuejs.org/v2/guide/deployment.html


