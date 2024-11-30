---
title: webpack
date: 2019-10-23 14:54:32
tags:
  - webpack
---

## 概念

> 本质上，webpack 是一个现代 JavaScript 应用程序的静态模块打包工具。当 webpack 处理应用程序时，它会在内部构建一个**依赖图(dependency graph)**，此依赖图会映射项目所需的每个模块，并生成一个或多个 bundle。

上面一段话摘自官方文档，由此引申出下面几个问题

1. 什么是依赖图？依赖图是怎么映射项目所需的模块？
2. bundle 是什么？

<!-- more -->

### 依赖图

webpack 开箱即用，可以无需使用任何配置文件。然而，webpack 会假定项目的入口起点为工程目录 `src/index`，然后会在 `dist/main.js` 输出结果，并且在生产环境开启压缩和优化。

也就是说，在没有任何配置文件和命令行传参的情况下，会有一个默认的入口起点 `src/index` (如果自己配置了入口就会使用配置好的)，直接执行

```cmd
webpack
```

webpack 会将 `mode` 的默认值设置为 `production`并开始打包，从 入口起点 开始，webpack 递归地构建一个依赖图，这个依赖图包含着应用程序所需的每个模块，然后将所有这些模块打包为少量的 bundle - 通常只有一个 - 可由浏览器加载。

所以依赖图的生成是首先取决于入口文件，当然，在入口文件中如果也引入了其它文件，那么其它文件也会变成依赖图一部分。

### bundle

bundle 就是 webpack 生成的文件，bundle 里包含多个 chunk - 代码块，可能多个 bundle 会存在相同的代码块，所以需要用代码分离来共享相同代码块部分

### process.env.NODE_ENV

在 wepback4 中，存在两种模式，`development`、`production`，webpack 自动会引入`definePlugin`，这个插件会决定在开发环境与生产环境(dev-vs-prod)下，server tools(服务期工具)、build scripts(构建脚本) 和 client-side libraries(客户端库) `process.env.NODE_ENV`的值，然而，在 `webpack.config.js` 中是不会存在这个值的，要在配置文件中使用`process.env.NODE_ENV`，有以下两种方法

- 命令行中传入`--env.NODE_ENV production|development`， `webpack.config.js` 中导出函数

```javascript
// webpack.config.js
module.exports = env => {
  console.log(env.NODE_ENV); // production|development
};
```

- `cross-env`传参

```javascript
// 安装cross-env
yarn add -D cross-env

// package.json
cross-env NODE_ENV=production webpack ./webpack.config.js

module.exports = {
  console.log(process.env.NODE_ENV) // production|development
}
```

## 入口 entry

入口的写法，

```javascript
// webpack.config.js
module.exports = {
  // normal
  entry: './path/to/my/entry/file.js'
  // 对象语法
  // 使用 optimization.splitChunks 为页面间共享的应用程序代码创建 bundle。
  // 由于入口起点增多，多页应用能够复用入口起点之间的大量代码/模块
  // 如果输出为library 可以使用 array 语法暴露所有模块
  // 参考 https://github.com/webpack/webpack/tree/master/examples/multi-part-library
  entry: {
    main: './path/to/my/entry/file.js'
  }
  // 数组
  // 这样传会合并多个文件为一个bundle文件
  // 如果输出为library时只会暴露最后一个模块
  entry: ['path1', 'path2']
  // 动态入口 可以接收远程服务器的数据来决定入口，感觉没什么用
  entry: () => './demo'
  // 或
  entry: () => new Promise((resolve) => resolve(['./demo', './demo2']))
};


```

## 输出 output

输出主要是告诉 webpack 打包后的 bundle 放在哪里，以及如何命名这些文件

```javascript
const path = require("path");

module.exports = {
  // ...
  // normal
  output: {
    path: path.resolve(__dirname, "dist"), // 打包后的目录 URL以HTML页面为基准
    filename: "my-first-webpack.bundle.js", // 文件命名
    publicPath: "assets/", // 相对于 html 页面
    publicPath: "/assets/" // 相对于服务器根目录
  },
  // 多个入口起点 使用占位符
  output: {
    filename: "[name].js"
    // 输出文件 ./dist/name1.js  ./dist/name2.js
  },
  // cdn hash
  output: {
    path: "/home/proj/cdn/assets/[hash]", // 此处打包后会生成到硬盘根路径
    publicPath: "http://cdn.example.com/assets/[hash]/" // cdn路径
  }
};
```

### 在运行时设置 publicPath

所谓运行时，即在打包完成后运行应用程序的时候。一般在 output 中配置的 publicPath 是固定的，但是，我们可能需要在运行的时候动态加载 publicPath,webpack 暴露了一个名为 **webpack_public_path** 的全局变量，通过改变这个变量的值达到我们的目的。

1. 创建一个文件`public_path.js`

```javascript
__webpack_public_path__ = "http://some.cdn.com/some";
```

2. 在入口文件中引入

```javascript
// entry.js
import "public_path.js";
import "./app.js";
```

> 如果在 entry 文件中使用 ES2015 module import，则会在 import 之后进行 **webpack_public_path** 赋值。在这种情况下，你必须将 public path 赋值移至一个专用模块中，然后将它的 import 语句放置到 entry.js 最上面

### chunkFilename

定义非入口 chunk 文件的名称。这个在动态导入时可以设置分出来的文件名

如下，在文件中使用 import()时，webpack 会在打包时将 a.js 分离出去，成为一个新的文件，这个文件在没有设置 chunkFilename 时会自动使用`[模块id].js`命名(如果注释名存在则使用注释名), 当然，使用注释命名也可以

```javascript
// entry.js
import('./a.js') // 输出文件 0.js
import(/* webpackChunkName: "chunk1" */'./a.js') // 输出文件 chunk1.js

// 设置chunkFilename后
output: {
  chunkFilename: '[name].[chunkhash].js' // name 一般是模块id 如果有注释名则使用注释名
},

// 输出
// 0.[chunkhash].js
// chunk1.[chunkhash].js
```

### crossOriginLoading | jsonpScriptType | chunkLoadTimeout

crossOriginLoading，只用于 `target` 是 `web`，使用了通过 script 标签的 JSONP 来按需加载 chunk。通过加载资源的 origin 信息来判断是否跨域，比如在 cdn 加载 chunk 的时候肯定是跨域的，那么此设置就会生效

jsonpScriptType 设置 jsonp 中 script 的 type 属性

chunkLoadTimeout 设置 script 中超时时间，默认 120s

```javascript
if (script.src.indexOf(window.location.origin + "/") !== 0) {
  script.crossOrigin = "anonymous";
}
```

### filename 中的 chunkhash contenthash

chunkhash 和 contenthash 的区别在于，都是 chunk 内容，不过 contenthash 是通过`ExtractTextWebpackPlugin`提取出来的，如果 js 文件改变打包后 css 内容即使没变 css hash 也会改变

### libraryTarget

配置如何暴露 library。

1. **var**. （默认值）当 library 加载完成，入口起点的返回值将分配给 library 变量，会覆盖掉已经定义过的全局变量（谨慎使用）

```javascript
output.library = "someLibName";
// 打包后，加载完库后会把库对象分配给全局变量 someLibName
var someLibName = module.exports; // 输出结果，如果在之前存在全局变量someLibName会覆盖
```

2. **assign**. 比 `'var'`少了个 var，可以说没区别

```javascript
someLibName = module.exports; // 输出结果
```

3. **this**.

- output.library 没有赋值，webpack 将把 library 对象上所有的属性挂载到浏览器的 this 上，也就是 window

```javascript
(function(e, a) { for(var i in a) e[i] = a[i]; }(this, module.exports)
// 遍历exports对象并挂载到this
```

- `output.library = 'someLibName'`则会将对象挂载到`this['someLibName']`

```javascript
this["someLibName"] = module.exports;
```

4. **window** 同上

```javascript
window["someLibName"] = module.exports;
```

5. **global** 分配给 global 对象

```javascript
global["someLibName"] = module.exports;
```

6. **commonjs** 分配给 exports 对象。这个名称也意味着，模块用于 CommonJS 环境

```javascript
exports["someLibName"] = module.exports;

require("someLibName").doSomething();
```

7. **commonjs2** 模块定义系统.用于`CommonJS`系统，入口起点的返回值将分配给 `module.exports` 对象

与`commonjs`的区别是不用指定 output.library

> 模块定义系统会使 `bundle` 带有更多的头部处理，以便兼容各种模块系统

```javascript
module.exports = _entry_return_;

require("MyLibrary").doSomething();
```

8. **amd** 将 library 导出为 AMD 模块

可以由 RequireJS 或任何兼容的模块加载器加载。直接加载会报错。

```javascript
// MyLibrary.js
define("MyLibrary", [], function() {
  return _entry_return_;
});

// 浏览器 使用前需要先引入RequireJS
require(["MyLibrary"], function(MyLibrary) {
  // 使用 library 做一些事……
});
```

9. **umd** 将 library 导出为所有的模块定义下都可运行的方式。既可以在 CommonJS, AMD 环境下运行，也可以在浏览器环境下且无需 requireJS 的情况下运行。

```javascript
// webpack配置
module.exports = {
  //...
  output: {
    library: "MyLibrary", // 如果不设置的话，webpack会把exports对象上的所有属性挂载到全局变量上
    libraryTarget: "umd"
  }
};
// 也可以给每个导出环境配置不同的名称
module.exports = {
  //...
  output: {
    library: {
      root: "MyLibrary",
      amd: "my-library",
      commonjs: "my-common-library"
    },
    libraryTarget: "umd"
  }
};

// MyLibrary.js
(function webpackUniversalModuleDefinition(root, factory) {
  if (typeof exports === "object" && typeof module === "object")
    module.exports = factory();
  else if (typeof define === "function" && define.amd) define([], factory);
  else if (typeof exports === "object") exports["MyLibrary"] = factory();
  else root["MyLibrary"] = factory();
})(typeof self !== "undefined" ? self : this, function() {
  return _entry_return_; // 此模块返回值，是入口 chunk 返回的值
});
```

10. **jsonp** 将导出结果包裹在以 library 变量作为函数名的容器中

```javascript
library: "MyLibrary";

// MyLibrary.js
MyLibrary(_entry_return_);
```

### exports、module.exports 和 export、export default

> require: node 和 es6 都支持的引入
> export / import : 只有 es6 支持的导出引入
> module.exports / exports: 只有 node 支持的导出

#### node 模块

- commonjs 导入导出 nodejs 支持，浏览器不支持（引用 requirejs 也可以支持）。**在 webpack 打包时，如果使用了 module.exports 作为最终输出时，在浏览器中运行是获取不到模块中的变量的**

Node 里面的模块系统遵循的是 CommonJS 规范。CommonJS 定义的模块分为: 模块标识(module)、模块定义(exports) 、模块引用(require)。
当 Node 执行一个文件时，会为文件生成一个 exports 和 module 对象，而 module 对象的 exports 属性和 exports 指向同一个内存地址。

```javascript
exports = module.exports = {};
```

当 Node 导入某个文件模块时，实际上是导入文件的 module.exports 属性。重新给 exports 属性赋一个对象会导致 exports 属性与 module.exports 断开连接。

- es6 导入导出 主要用于浏览器加载模块，当然 nodejs 也支持
  1.  export 与 export default 均可用于导出常量、函数、文件、模块等
  2.  在一个文件或模块中，export、import 可以有多个，export default 仅有一个
  3.  通过 export 方式导出，在导入时要加{ }，export default 则不需要
  4.  export 能直接导出变量表达式，export default 不行。

```javascript
// testEs6Export.js
"use strict";
//导出变量
export const a = "100";

//导出方法
export const dogSay = function() {
  console.log("wang wang");
};

//导出方法第二种
function catSay() {
  console.log("miao miao");
}
export { catSay };

//export default导出
const m = 100;
export default m;

//
// index.js

import { dogSay, catSay } from "./testEs6Export"; //导出了 export 方法
import m from "./testEs6Export"; //导出了 export default

import * as testModule from "./testEs6Export"; //as 集合成对象导出
console.log(testModule.m); // undefined , 因为  as 导出是 把 零散的 export 聚集在一起作为一个对象，而export default 是导出为 default属性。
console.log(testModule.default); // 100
```

#### commonjs vs commonjs2

那么 webpack 打包 library 时 commonjs 与 commonjs2 的区别就是 commonjs 必须赋值一个变量作为 exports 的属性，commonjs2 则是直接导出为 `module.exports` 的对象

#### 缓存

利用缓存技术可以合理利用浏览器缓存减少请求，加快网站的加载速度。

```javascript
module.exports = {
  output: {
    // 改为contenthash 通过内容来映射hash,内容变化则hash变，内容不变hash不变
    filename: "[name].[contenthash].js"
  },
  optimization: {
    // 分离runtime
    runtimeChunk: "single",
    // 将第三方库提取到单独的vendor文件中
    splitChunks: {
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: "vendors",
          chunks: "all"
        }
      }
    },
    plugins: [
      new webpack.HashedModuleIdsPlugin() // 保持内容不变的情况下hash也不变
    ]
  }
};
```

### output.umdNamedDefine

当使用了 `libraryTarget: "umd"`，设置：

```javascript
module.exports = {
  //...
  output: {
    umdNamedDefine: true
  }
};

// 打包后
if (typeof define === "function" && define.amd)
  define("someLibName" /*这里会加上library字段的值*/, [], factory);
```

### output.pathinfo

开启后多了下面的注释部分，会导致造成垃圾回收性能压力，建议还是关闭

```javascript
/***/ "tjUo":
/*!**********************!*\
  !*** ./src/index.js ***!
  \**********************/
/*! exports provided: a */
/***/
```

## 模式 mode

> 值有：`none`, `development`, `production`（默认）。设置 `NODE_ENV` 并不会自动地设置 mode。

### 用法

```javascript
// 配置文件
module.exports = {
  mode: 'development'
}

// cli传参
webpack --mode=development
```

| 选项        | 描述                                                                                                                                                                                                                                       |
| ----------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| development | 会将 DefinePlugin 中 process.env.NODE_ENV 的值设置为 development。启用 NamedChunksPlugin 和 NamedModulesPlugin。                                                                                                                           |
| production  | 会将 DefinePlugin 中 process.env.NODE_ENV 的值设置为 production。启用 FlagDependencyUsagePlugin, FlagIncludedChunksPlugin, ModuleConcatenationPlugin, NoEmitOnErrorsPlugin, OccurrenceOrderPlugin, SideEffectsFlagPlugin 和 TerserPlugin。 |
| none        | 退出任何默认优化选项.                                                                                                                                                                                                                      |

#### `mode: development`

```javascript
// webpack.development.config.js
module.exports = {
+ mode: 'development'
- devtool: 'eval',
- cache: true, // 缓存生成的 webpack 模块和 chunk，来改善构建速度。缓存默认在观察模式(watch mode)启用
- performance: {
-   hints: false // false | "error" | "warning" 是否开启打包后文件过大的性能提示 false不开启，warning 展示警告 error展示错误(开发环境中会展示在浏览器的工作台中)。文件大小可以限制可以自由配置
- },
- output: {
-   pathinfo: true
- },
- optimization: {
-   namedModules: true,
-   namedChunks: true,
-   nodeEnv: 'development',
-   flagIncludedChunks: false,
-   occurrenceOrder: false,
-   sideEffects: false,
-   usedExports: false,
-   concatenateModules: false,
-   splitChunks: {
-     hidePathInfo: false,
-     minSize: 10000,
-     maxAsyncRequests: Infinity,
-     maxInitialRequests: Infinity,
-   },
-   noEmitOnErrors: false,  // 设置为true 则会在编译出错时跳过生成阶段，避免生成错误打包文件
-   checkWasmTypes: false,
-   minimize: false,
- },
- plugins: [
-   new webpack.NamedModulesPlugin(),
-   new webpack.NamedChunksPlugin(),
-   new webpack.DefinePlugin({ "process.env.NODE_ENV": JSON.stringify("development") }),
- ]
}
```

## optimization

webpack 4 特有的优化选项，可以进行压缩代码，分包等操作

### minimize

开启后使用[TerserPlugin](https://webpack.docschina.org/plugins/terser-webpack-plugin/)压缩。`mode:production`时自动开启

### minimizer

测试后只在`mode: production`时有效，可以配置 terserPlugin 的参数

### splitChunks

用于分割代码块，提取出公用代码块。在多页面项目或者动态导入模块的时候非常有用，能减少初始加载代码的大小，提升网页首屏的加载速度。

`splitChunks`默认只影响按需块，当然也可以通过设置 `chunks: 'initial'`来拆分公用初始代码块。

> `splitChunks` 总是会提取按需块

先看看 webpack 中默认的`splitChunks`设置

```javascript
module.exports = {
  //...
  optimization: {
    splitChunks: {
      // 可设置的值有 initial, async, all
      // all 最佳配置，当分离的块既有同步又有按需引入时，共享代码
      // initial 当分离的块既有同步又有按需引入时，不共享代码
      // async 只拆分按需引入块
      chunks: "async",
      // 当代码块大于这个值的时候就会被拆分出来
      minSize: 30000,
      // 当代码块大于这个值的时候会继续拆分（如果还可以拆分的话） 0表示不拆分
      maxSize: 0,
      // 当代码块被引用的次数超过这个数的时候才会拆分
      minChunks: 1,
      // 最多能拆分的按需块 >= 1
      maxAsyncRequests: 5,
      // 最多能拆分的初始块 >= 1  如果设置了maxSize，并且能拆分，可能会拆分出更多的块
      maxInitialRequests: 3,
      // 块文件名分隔符
      automaticNameDelimiter: "~",
      // 分割块的名字。如果传入 true 将会自动生成一个基于块组和缓存组键的名称
      // 也可以用函数生成名称
      // name (module, chunks, cacheGroupKey) {
      //   // generate a chunk name...
      //   return; //...
      // },
      name: true,
      cacheGroups: {
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          priority: -10
        },
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

#### 多页面分割代码

[栗子地址](https://github.com/kingRywen/webpack-test/tree/splitChunk)

```javascript
let entry = {
  // 多个入口
  index: "./src/index.js",
  pageA: "./src/a.js",
  pageB: "./src/b.js",
  pageC: "./src/c.js",
  pageD: "./src/d.js"
};
let _chunks = {};

// 添加分离出来的runtime和node_modules中的库，以及当前页的chunk到chunk映射变量_chunks中
Object.keys(entry).forEach(key => {
  _chunks[key] = [key, "runtime", "vendors"];
});

const htmls = Object.keys(entry).map(
  name =>
    new HtmlWebpackPlugin({
      filename: name + ".html",
      template: "template.pug",
      title: name,
      chunks: _chunks[name]
    })
);

module.exports = {
  mode: "production",
  entry,
  // ...
  output: {
    path: path.resolve(__dirname, "dist"),
    crossOriginLoading: "anonymous",
    filename: "[name].[contenthash].js",
    pathinfo: false
  },
  performance: {
    hints: false
  },
  optimization: {
    minimize: true,
    runtimeChunk: "single",
    splitChunks: {
      chunks: "all",
      minSize: 30000,
      maxSize: 0,
      minChunks: 2,
      maxAsyncRequests: 20,
      maxInitialRequests: 20,
      automaticNameDelimiter: "~",
      // 关键命名函数，将引入次数2次以上的公共业务代码分割出来，并命名
      // 命名的同时将chunk推入_chunks中，改变入口htmlwebpack中chunks的引入
      name(module, chunks, cacheGroupKey) {
        let name = chunks.map(el => el.name).join("~");
        for (let index = 0; index < chunks.length; index++) {
          const c = chunks[index];
          if (c.name && _chunks[c.name] && !~_chunks[c.name].indexOf(name)) {
            _chunks[c.name].push(name);
          }
        }
        return name;
      },
      cacheGroups: {
        // 分割库代码
        commons: {
          test: /[\\/]node_modules[\\/]/,
          name: "vendors",
          chunks: "all"
        }
      }
    }
  }
  // ...
};
```

### 模块(module)

webpack 模块是管理各种文件资源的途径，通过 loader 能解析非.js 文件

#### css

##### 普通 css

通过`import './style.css'`方法引入样式文件，在没有配置 module 的情况下 webpack 是不会正常解析 css 文件的，所以我们必须引入`style-loader`来将引入的 css 文件解析出来。当然也要引入`css-loader`来解析 css 文件的内容,css-loader 也能解析 css 中的引入：`@import`及`url()`（可以配置不解析某些资源）

```javascript
// 安装
yarn add -D style-loader
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          // 区分生产环境和开发环境,生产环境时提取css作为单独文件
          process.env.NODE_ENV == 'production' ? MiniCssExtractPlugin.loader : 'style-loader', {
            loader: 'css-loader',
            options:{
              modules: true, // css 模块化
              sourceMap: true // 开启 sourceMap
            }
          }
        ]
      }
    ]
  }
}
```

##### sass

```javascript
// 安装
yarn add sass-loader node-sass -D

// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.scss$/,
        use: [
          process.env.NODE_ENV == 'production' ? MiniCssExtractPlugin.loader : 'style-loader',
          {
            loader: 'css-loader',
            options: {
              sourceMap: true
            }
          },
          'sass-loader'
        ]
      }
    ]
  }
}
```

#####　 postcss

在[PostCSS](http://postcss.org/)官网有着这样的对 PostCSS 特性介绍，箭头后面是对应功能的插件及其 github 地址。

1. increase code readability （增加代码可读性，补全） → [Autoprefixer](https://github.com/postcss/autoprefixer)
2. Use tomorrow's CSS ,today!（使用下代 css） → [postcss-cssnext](https://github.com/MoOx/postcss-cssnext/)
3. The end of global CSS（模块化 css）→ [postcss-modules](https://github.com/css-modules/postcss-modules)
4. Avoid errors in your CSS（错误提示） → [stylelint](https://github.com/stylelint/stylelint)
5. Powerful grid CSS（栅格系统） → lost →[lost](https://github.com/peterramsing/lost)

##### webpack 中使用 postcss

`PostCSS Preset Env`可以将现代 CSS 转换为大多数浏览器可以理解的内容，并根据目标浏览器或运行时环境确定所需的 polyfill。

启用 sourceMap 支持，postcss-loader 将使用其他加载器提供的先前的 sourceMap 并进行相应的更新。除`style-loader`之外的 loader 都需要配置`sourceMap: true`

```javascript
// 安装
yarn add -D postcss-preset-env

// webpack.config.js
module.exports = {
  rules: [
    {
      test: /\.css$/,
      use: [
      'style-loader',
      {
        loader: "css-loader",
        options: {
          sourceMap: true,
          // 必须配置，影响@import导入的文件中的加载器的数量，
          // 如果不配置@import导入文件将不会应用后面的加载器
          importLoaders: 2
        }
      },
      {
        loader: 'postcss-loader',
        options: {
          sourceMap: true,
          indent: 'postcss',
          plugins: () => [
            // 使用 PostCSS Preset Env
            // postcss-preset-env已经预置autoprefixer，无需单独添加它
            postcssPresetEnv({
              stage: 3,
              // 支持任何标准的browserslist配置，可以是.browserslistrc文件，package.json中的browserslist或browserslist环境变量。
              browsers: 'last 2 versions'
            })
          ]
        }
      }]
    }
  ]
}
```

#### images 图像

```javascript
// 安装
yarn add -D url-loader

// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.(png|svg|jpg|gif)$/,
        // url-loader将少于limit的图像文件转化为dataUrl来减少请求
        loader: 'url-loader',
        options: {
          // 当图像大于limit时，会使用备用的loader
          fallback: {
            loader: 'file-loader',
            options: {
              name: 'img/[name].[hash:8].[ext]'
            }
          },
          limit: 4096
        }
      }
    ]
  }
}
```

#### fonts 字体

`file-loader` 和 `url-loader` 可以接收并加载任何文件，所以也可以用来加载 fonts 字体文件

```javascript
module.exports = {
  module: {
    rules: [
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/,
        loader: "url-loader",
        options: {
          fallback: {
            loader: "file-loader",
            options: {
              name: "img/[name].[hash:8].[ext]"
            }
          },
          limit: 4096
        }
      }
    ]
  }
};
```

#### 数据 json xml

加载的资源还有数据，如 JSON 文件，CSV、TSV 和 XML。类似于 NodeJS，JSON 支持实际上是内置的，也就是说 `import Data from './data.json'` 默认将正常运行

```javascript
+       {
+         test: /\.(csv|tsv)$/,
+         use: [
+           'csv-loader'
+         ]
+       },
+       {
+         test: /\.xml$/,
+         use: [
+           'xml-loader'
+         ]
+       }
```
