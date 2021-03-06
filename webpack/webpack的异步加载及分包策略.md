## webpack 异步加载原理

`webpack ensure` 有人称它为异步加载，也有人称为代码切割，他其实就是将 js 模块给独立导出一个.js 文件，然后使用这个模块的时候，再创建一个 `script` 对象，加入到 `document.head` 对象中，浏览器会自动帮我们发起请求，去请求这个 js 文件，然后写个回调函数，让请求到的 js 文件做一些业务操作。

### 举个例子

需求：`main.js` 依赖两个 js 文件：`A.js` 是点击 aBtn 按钮后，才执行的逻辑，`B.js` 是点击 bBtn 按钮后，才执行的逻辑。

`webpack.config.js`，我们先来写一下 `webpack` 打包的配置的代码

```js
const path = require('path') // 路径处理模块
const HtmlWebpackPlugin = require('html-webpack-plugin')
const { CleanWebpackPlugin } = require('clean-webpack-plugin') // 引入CleanWebpackPlugin插件

module.exports = {
  entry: {
    index: path.join(__dirname, '/src/main.js'),
  },
  output: {
    path: path.join(__dirname, '/dist'),
    filename: 'index.js',
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: path.join(__dirname, '/index.html'),
    }),
    new CleanWebpackPlugin(), // 所要清理的文件夹名称
  ],
}
```

`index.html` 代码如下

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>webpack</title>
  </head>
  <body>
    <div id="app">
      <button id="aBtn">按钮A</button>
      <button id="bBtn">按钮B</button>
    </div>
  </body>
</html>
```

入口文件 `main.js` 如下

```js
import A from './A'
import B from './B'

document.getElementById('aBtn').onclick = function () {
  alert(A)
}

document.getElementById('bBtn').onclick = function () {
  alert(B)
}
```

`A.js` 和 `B.js` 的代码分别如下

```js
// A.js
const A = 'hello A'
module.exports = A

// B.js
const B = 'hello B'
module.exports = B
```

此时，我们对项目进行 `npm run build`， 打包出来的只有两个文件

- index.html
- index.js

由此可见，此时 `webpack` 把 `main.js` 依赖的两个文件都同时打包到同一个 js 文件，并在 index.html 中引入。但是 `A.js` 和 `B.js` 都是点击相应按钮才会执行的逻辑，如果用户并没有点击相应按钮，而且这两个文件又是比较大的话，这样是不是就导致首页默认加载的 js 文件太大，从而导致首页渲染较慢呢？那么有能否实现当用户点击按钮的时候再加载相应的依赖文件呢？

`webpack.ensure` 就解决了这个问题。

### require.ensure 异步加载

下面我们将 `main.js` 改成异步加载的方式

```js
document.getElementById('aBtn').onclick = function () {
  //异步加载A
  require.ensure([], function () {
    let A = require('./A.js')
    alert(A)
  })
}

document.getElementById('bBtn').onclick = function () {
  //异步加载b
  require.ensure([], function () {
    let B = require('./B.js')
    alert(B)
  })
}
```

此时，我们再进行一下打包，发现多了 `1.index.js` 和 `2.index.js` 两个文件。而我们打开页面时只引入了 `index.js` 一个文件，当点击按钮 A 的时候才引入 `1.index.js` 文件，点击按钮 B 的时候才引入 `2.index.js` 文件。这样就满足了我们按需加载的需求。

`require.ensure` 这个函数是一个代码分离的分割线，表示回调里面的 `require` 是我们想要进行分割出去的，即 `require('./A.js')`，把 A.js 分割出去，形成一个 `webpack` 打包的单独 js 文件。它的语法如下

```js
require.ensure(dependencies: String[], callback: function(require), chunkName: String)
```

我们打开 `1.index.js` 文件，发现它的代码如下

```js
;(window.webpackJsonp = window.webpackJsonp || []).push([
  [1],
  [
    ,
    function (o, n) {
      o.exports = 'hello A'
    },
  ],
])
```

由上面的代码可以看出：

1. 异步加载的代码，会保存在一个全局的 `webpackJsonp` 中。
2. `webpackJsonp.push` 的的值，两个参数分别为异步加载的文件中存放的需要安装的模块对应的 id 和异步加载的文件中存放的需要安装的模块列表。
3. 在满足某种情况下，会执行具体模块中的代码。

### import() 按需加载

webpack4 官方文档提供了模块按需切割加载，配合 es6 的按需加载 `import()` 方法，可以做到减少首页包体积，加快首页的请求速度，只有其他模块，只有当需要的时候才会加载对应 js。

`import()`的语法十分简单。该函数只接受一个参数，就是引用包的地址，并且使用了 `promise` 式的回调，获取加载的包。在代码中所有被 `import()`的模块，都将打成一个单独的包，放在 `chunk` 存储的目录下。在浏览器运行到这一行代码时，就会自动请求这个资源，实现异步加载。

下面我们将上述代码改成 `import()`方式。

```js
document.getElementById('aBtn').onclick = function () {
  //异步加载A
  import('./A').then((data) => {
    alert(data.A)
  })
}

document.getElementById('bBtn').onclick = function () {
  //异步加载b
  import('./B').then((data) => {
    alert(data.B)
  })
}
```

此时打包出来的文件和 `webpack.ensure` 方法是一样的。

## 路由懒加载

为什么需要懒加载？

像 vue 这种单页面应用，如果没有路由懒加载，运用 webpack 打包后的文件将会很大，造成进入首页时，需要加载的内容过多，出现较长时间的白屏，运用路由懒加载则可以将页面进行划分，需要的时候才加载页面，可以有效的分担首页所承担的加载压力，减少首页加载用时。

vue 路由懒加载有以下三种方式

- vue 异步组件
- ES6 的 `import()`
- webpack 的 `require.ensure()`

### vue 异步组件

这种方法主要是使用了 `resolve` 的异步机制，用 `require` 代替了 `import` 实现按需加载

```js
export default new Router({
  routes: [
    {
      path: '/home',',
      component: (resolve) => require(['@/components/home'], resolve),
    },
    {
      path: '/about',',
      component: (resolve) => require(['@/components/about'], resolve),
    },
  ],
})
```

### require.ensure

这种模式可以通过参数中的 `webpackChunkName` 将 js 分开打包。

```js
export default new Router({
  routes: [
    {
      path: '/home',
      component: (resolve) => require.ensure([], () => resolve(require('@/components/home')), 'home'),
    },
    {
      path: '/about',
      component: (resolve) => require.ensure([], () => resolve(require('@/components/about')), 'about'),
    },
  ],
})
```

### ES6 的 import()

`vue-router` 在官网提供了一种方法，可以理解也是为通过 `Promise` 的 `resolve` 机制。因为 `Promise` 函数返回的 `Promise` 为 `resolve` 组件本身，而我们又可以使用 `import` 来导入组件。

```js
export default new Router({
  routes: [
    {
      path: '/home',
      component: () => import('@/components/home'),
    },
    {
      path: '/about',
      component: () => import('@/components/home'),
    },
  ],
})
```

## webpack 分包策略

在 webpack 打包过程中，经常出现 `vendor.js`， `app.js` 单个文件较大的情况，这偏偏又是网页最先加载的文件，这就会使得加载时间过长，从而使得白屏时间过长，影响用户体验。所以我们需要有合理的分包策略。

### CommonsChunkPlugin

在 Webapck4.x 版本之前，我们都是使用 `CommonsChunkPlugin` 去做分离

```js
plugins: [
  new webpack.optimize.CommonsChunkPlugin({
    name: 'vendor',
    minChunks: function (module, count) {
      return (
        module.resource &&
        /\.js$/.test(module.resource) &&
        module.resource.indexOf(path.join(__dirname, './node_modules')) === 0
      )
    },
  }),
  new webpack.optimize.CommonsChunkPlugin({
    name: 'common',
    chunks: 'initial',
    minChunks: 2,
  }),
]
```

我们把以下文件单独抽离出来打包

- `node_modules` 文件夹下的，模块
- 被 3 个 入口 `chunk` 共享的模块

### optimization.splitChunks

webpack 4 最大的改动就是废除了 `CommonsChunkPlugin` 引入了 `optimization.splitChunks`。如果你的 `mode` 是 `production`，那么 webpack4 就会自动开启 `Code Splitting`。

它内置的代码分割策略是这样的：

- 新的 chunk 是否被共享或者是来自 `node_modules` 的模块
- 新的 chunk 体积在压缩之前是否大于 30kb
- 按需加载 chunk 的并发请求数量小于等于 5 个
- 页面初始加载时的并发请求数量小于等于 3 个

虽然在 webpack4 会自动开启 `Code Splitting`，但是随着项目工程的最大，这往往不能满足我们的需求，我们需要再进行个性化的优化。

### 应用实例

我们先找到一个优化空间较大的项目来进行操作。这是一个后台管理系统项目，大部分内容由 3-4 个前端开发，平时开发周期较短，且大部分人没有优化意识，只是写好业务代码完成需求，日子一长，造成打包出来的文件较大，大大影响性能。

我们先用 `webpack-bundle-analyzer` 分析打包后的模块依赖及文件大小，确定优化的方向在哪。

<img src="../img/21.png">

然后我们再看下打包出来的 js 文件

<img src="../img/31.png">

看到这两张图的时候，我内心是崩溃的，槽点如下

- 打包后生成多个将近 1M 的 js 文件，其中不乏 `vendor.js` 首页必须加载的大文件
- `xlsx.js` 这样的插件没必要使用，导出 excel 更好的方法应该是后端返回文件流格式给前端处理
- `echart` 和 `iview` 文件太大，应该使用 cdn 引入的方法

吐槽完之后我们就要开始做正事了。正是因为有这么多槽点，我们才更好用来验证我们优化方法的可行性。

#### 抽离 echart 和 iview

由上面分析可知，`echart` 和 `iview` 文件太大，此时我们就用到 webpack4 的 `optimization.splitChunks` 进行代码分割了，把他们单独抽离打包成文件。(为了更好地呈现优化效果，我们先把 xlsx.js 去掉)

`vue.config.js` 修改如下：

```js
chainWebpack: config => {
    config.optimization.splitChunks({
      chunks: 'all',
      cacheGroups: {
        vendors: {
          name: 'chunk-vendors',
          test: /[\\/]node_modules[\\/]/,
          priority: 10,
          chunks: 'initial'
        },
        iview: {
          name: 'chunk-iview',
          priority: 20,
          test: /[\\/]node_modules[\\/]_?iview(.*)/
        },
        echarts: {
          name: 'chunk-echarts',
          priority: 20,
          test: /[\\/]node_modules[\\/]_?echarts(.*)/
        },
        commons: {
          name: 'chunk-commons',
          minChunks: 2,
          priority: 5,
          chunks: 'initial',
          reuseExistingChunk: true
        }
      }
    })
  },
```

此时我们再用 `webpack-bundle-analyzer` 分析一下

<img src="../img/23.png">

打包出来的 js 文件

<img src="../img/33.png">

从这里可以看出我们已经成功把 `echart` 和 `iview` 单独抽离出来了，同时 `vendor.js` 也相应地减小了体积。此外，我们还可以继续抽离其他更多的第三方模块。

#### CDN 方式

虽然第三方模块是单独抽离出来了，但是在首页或者相应路由加载时还是要加载这样一个几百 kb 的文件，还是不利于性能优化的。这时，我们可以用 CDN 的方式引入这样插件或者 UI 组件库。

1. 在 `index.html` 引入相应 cdn 链接

```html
<head>
  <link rel="stylesheet" href="https://cdn.bootcdn.net/ajax/libs/iview/3.5.4/styles/iview.css" />
</head>
<body>
  <div id="app"></div>
  <script src="https://cdn.bootcss.com/vue/2.6.8/vue.min.js"></script>
  <script src="https://cdn.bootcdn.net/ajax/libs/iview/3.5.4/iview.min.js"></script>
  <script src="https://cdn.bootcdn.net/ajax/libs/xlsx/0.16.8/xlsx.mini.min.js"></script>
  <script src="https://cdn.bootcdn.net/ajax/libs/xlsx/0.16.8/cpexcel.min.js"></script>
</body>
```

2. `vue.config.js` 配置 `externals`

```js
configureWebpack: (config) => {
  config.externals = {
    vue: 'Vue',
    xlsx: 'XLSX',
    iview: 'iView',
    iView: 'ViewUI',
  }
}
```

3. 删除之前的引入方式并卸载相应 npm 依赖包

```
npm uninstall vue iview echarts xlsx --save
```

此时我们在来看一下打包后的情况

<img src="../img/25.png">

打包出来的 js 文件

<img src="../img/35.png">

well done ! 这时基本没有打包出大文件了，首页加载需要的 `vendor.js` 也只有几十 kb，而且我们还可以进一步优化，就是把 vue 全家桶的一些模块再通过 cdn 的方法引入，比如 `vue-router`，`vuex`，`axios` 等。这时页面特别是首页加载的性能就得到大大地优化了。
