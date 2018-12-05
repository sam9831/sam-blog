> 原文地址：https://juejin.im/post/5bf822c2f265da61307486c7

> 本教程是rollup.js系列教程的最后一篇，我将基于Vue.js框架，深度分析Vue.js源码打包过程，让大家深入理解复杂的前端框架是如何利用rollup.js进行打包的。通过这一篇教程的学习，相信大家可以更好地应用rollup.js为自己的项目服务。

# 前置学习——基础知识
要理解`Vue.js`的打包源码，需要掌握以下知识点：
- `fs`模块：`Node.js`内置模块，用于本地文件系统处理；
- `path`模块：`Node.js`内置模块，用于本地路径解析；
- `buble`模块：用于`ES6+`语法编译；
- `flow`模块：用于`Javascript`源码静态检查；
- `zlib`模块：`Node.js`内置模块，用于使用`gzip`算法进行文件压缩；
- `terser`模块：用于`Javascript`代码压缩和美化。

我将这些基础知识点整理成一篇前置学习教程：[《10分钟快速精通rollup.js——前置学习之基础知识篇》](https://juejin.im/post/5bf82319f265da614b11a621)，感兴趣的小伙伴可以看看。

# 前置学习——rollup.js插件
`rollup.js`进阶教程中讲解了`rollup.js`的部分常用插件：
- `rollup-plugin-resolve`：集成外部模块代码；
- `rollup-plugin-commonjs`：支持`CommonJS`模块；
- `rollup-plugin-babel`：编译`ES6+`语法为`ES2015`；
- `rollup-plugin-json`：支持`json`模块；
- `rollup-plugin-uglify`：代码压缩（不支持`ES`模块）；

为了理解`Vue.js`的打包源码，我们还需要学习以下`rollup.js`插件及知识：
- `rollup-plugin-buble`插件：编译`ES6+`语法为`ES2015`，无需配置，比`babel`更轻量；
- `rollup-plugin-alias`插件：替换模块路径中的别名；
- `rollup-plugin-flow-no-whitespace`插件：去除`flow`静态类型检查代码；
- `rollup-plugin-replace`插件：替换代码中的变量为指定值；
- `rollup-plugin-terser`插件：代码压缩，取代`uglify`，支持`ES`模块。
- `intro`和`outro`配置：在代码块内添加代码注释。

我为还不熟悉这些插件的小伙伴准备了另一篇前置学习教程：[《10分钟快速精通rollup.js——前置学习之rollup.js插件篇》](https://juejin.im/post/5bf823b96fb9a049e93c61a8)。

# Vue.js源码打包
`Vue.js`的打包过程并不复杂，首先要将`Vue.js`源码clone到本地：
```bash
git clone https://github.com/vuejs/vue.git
```
安装依赖：
```bash
cd vue
npm i
```
打开package.json查看scripts：
```json
"scripts": {
  "build": "node scripts/build.js",
  "build:ssr": "npm run build -- web-runtime-cjs,web-server-renderer",
  "build:weex": "npm run build -- weex",
}
```
我们先通过build指令进行打包：
```bash
$ npm run build

> vue@2.5.17-beta.0 build /Users/sam/WebstormProjects/vue
> node scripts/build.js

dist/vue.runtime.common.js 209.20kb
dist/vue.common.js 288.22kb
dist/vue.runtime.esm.js 209.18kb
dist/vue.esm.js 288.20kb
dist/vue.runtime.js 219.55kb
dist/vue.runtime.min.js 60.24kb (gzipped: 21.62kb)
dist/vue.js 302.27kb
dist/vue.min.js 85.19kb (gzipped: 30.86kb)
packages/vue-template-compiler/build.js 121.88kb
packages/vue-template-compiler/browser.js 228.17kb
packages/vue-server-renderer/build.js 220.73kb
packages/vue-server-renderer/basic.js 304.00kb
packages/vue-server-renderer/server-plugin.js 2.92kb
packages/vue-server-renderer/client-plugin.js 3.03kb
```
打包成功后会在dist目录下创建下列打包文件：
![Vue.js打包文件](https://user-gold-cdn.xitu.io/2018/11/23/1674147a71937e10?w=660&h=444&f=png&s=54922)
以上就是使用build指令对Vue.js源码进行打包的过程，除此之外，Vue.js还提供了另外两种打包方式：`build:ssr`和`build:weex`，先尝试`build:ssr`指令：
```bash
$ npm run build:ssr

> vue@2.5.17-beta.0 build:ssr /Users/sam/WebstormProjects/vue
> npm run build -- web-runtime-cjs,web-server-renderer


> vue@2.5.17-beta.0 build /Users/sam/WebstormProjects/vue
> node scripts/build.js "web-runtime-cjs,web-server-renderer"

dist/vue.runtime.common.js 209.20kb
packages/vue-server-renderer/build.js 220.73kb
packages/vue-server-renderer/basic.js 304.00kb
packages/vue-server-renderer/server-plugin.js 2.92kb
packages/vue-server-renderer/client-plugin.js 3.03kb
```
再尝试`build:weex`：
```bash
$ npm run build:weex

> vue@2.5.17-beta.0 build:weex /Users/sam/WebstormProjects/vue
> npm run build -- weex


> vue@2.5.17-beta.0 build /Users/sam/WebstormProjects/vue
> node scripts/build.js "weex"

packages/weex-vue-framework/factory.js 193.79kb
packages/weex-vue-framework/index.js 5.68kb
packages/weex-template-compiler/build.js 109.11kb
```
通过命令行日志可以看出这两个指令和build指令没有本质区别，都是通过`node`执行scripts/build.js源码，只是附带的参数不同：
```bash
node scripts/build.js # build
node scripts/build.js "web-runtime-cjs,web-server-renderer" # build:ssr
node scripts/build.js "weex" # build:weex
```
可见scripts/build.js是解读`Vue.js`源码打包的关键。下面我们就来分析`Vue.js`的源码打包流程。

# Vue.js打包流程分析
`Vue.js`源码打包基于`rollup.js`的API，大致可分为五步，如下图所示：
![Vue.js源码打包流程](https://user-gold-cdn.xitu.io/2018/11/23/1674147a9e775fcc?w=1696&h=430&f=png&s=90014)
- 第一步：创建dist目录。检查是否存在dist目录，如果不存在，则进行创建；
- 第二步：生成`rollup`配置文件。通过scripts/config.js生成`rollup`的配置文件；
- 第三步：`rollup`配置文件过滤。根据传入的参数，对`rollup`配置文件的内容进行过滤，排除不必要的打包项目。
- 第四步：遍历配置打包，生成打包源码。遍历配置文件项目，通过`rollup`的API进行打包，并生成打包后的源码。
- 第五步：源码输出文件，gzip压缩测试。如果输出的是最终产品，则通过`terser`进行最小化压缩并通过`zlib`进行gzip压缩测试，并在控制台输出测试结果，最后将源码内容输出到指定文件中，完成打包。

# Vue.js打包源码分析
下面我们将深入`Vue.js`打包源码，解析打包的原理和细节。
> 友情提示：建议阅读源码之前先将之前提供的四份教程全部看完：
> - [《10分钟快速入门rollup.js》](https://juejin.im/post/5bed8b26e51d4560336ca5b3)
> - [《10分钟快速进阶rollup.js》](https://juejin.im/post/5bf532546fb9a049e129d529)
> - [《10分钟快速精通rollup.js——前置学习之基础知识篇》](https://juejin.im/post/5bf82319f265da614b11a621)
> - [《10分钟快速精通rollup.js——前置学习之rollup.js插件篇》](https://juejin.im/post/5bf823b96fb9a049e93c61a8)

## 创建dist目录
执行`npm run build`时，会从scripts/build.js开始执行：
```js
// scripts/build.js
const fs = require('fs')
const path = require('path')
const zlib = require('zlib')
const rollup = require('rollup')
const terser = require('terser')

if (!fs.existsSync('dist')) {
  fs.mkdirSync('dist')
}
```
前5行分别导入了5个模块，这5个模块的用途在前置学习教程中已经详细过。第7行通过同步方法判断dist目录是否存在，如果不存在则通过同步方法创建dist目录。

## 生成rollup配置
生成dist目录后，通过以下代码生成了`rollup`的配置文件：
```js
// scripts/build.js
let builds = require('./config').getAllBuilds()
```
代码虽然只有短短一句，但是做了很多事情。首先它加载了scripts/config.js模块，然后调用其中的`getAllBuilds()`方法。下面我们来分析scripts/config.js的加载过程，加载config.js时先执行了以下内容：
```js
// scripts/config.js
const path = require('path')
const buble = require('rollup-plugin-buble')
const alias = require('rollup-plugin-alias')
const cjs = require('rollup-plugin-commonjs')
const replace = require('rollup-plugin-replace')
const node = require('rollup-plugin-node-resolve')
const flow = require('rollup-plugin-flow-no-whitespace')
```
这些插件的用途和用法在进阶教程和前置教程中都有介绍。
```js
const version = process.env.VERSION || require('../package.json').version
const weexVersion = process.env.WEEX_VERSION || require('../packages/weex-vue-framework/package.json').version
```
上述代码是从package.json中获取`Vue`的版本号和`Weex`的版本号。
```js
const banner =
  '/*!\n' +
  ` * Vue.js v${version}\n` +
  ` * (c) 2014-${new Date().getFullYear()} Evan You\n` +
  ' * Released under the MIT License.\n' +
  ' */'
```
上述代码生成了banner文本，在`Vue`代码打包后，会写在文件顶部。
```js
const weexFactoryPlugin = {
  intro () {
    return 'module.exports = function weexFactory (exports, document) {'
  },
  outro () {
    return '}'
  }
}
```
上述代码仅用于打包`weex-factory`源码时使用：
```js
// Weex runtime factory
'weex-factory': {
  weex: true,
  entry: resolve('weex/entry-runtime-factory.js'),
  dest: resolve('packages/weex-vue-framework/factory.js'),
  format: 'cjs',
  plugins: [weexFactoryPlugin]
}
```
接下来导入了scripts/alias.js模块：
```js
const aliases = require('./alias')
```
alias.js模块输出了一个对象，这个对象中定义了所有的别名及其对应的绝对路径：
```js
// scripts/alias.js
const path = require('path')

const resolve = p => path.resolve(__dirname, '../', p)

module.exports = {
  vue: resolve('src/platforms/web/entry-runtime-with-compiler'),
  compiler: resolve('src/compiler'),
  core: resolve('src/core'),
  shared: resolve('src/shared'),
  web: resolve('src/platforms/web'),
  weex: resolve('src/platforms/weex'),
  server: resolve('src/server'),
  entries: resolve('src/entries'),
  sfc: resolve('src/sfc')
}
```
这个模块中定义了`resolve()`方法，用于生成绝对路径：
```js
const resolve = p => path.resolve(__dirname, '../', p)
```
`__dirname`为当前模块对应的路径，即`scripts/`目录，`../`表示上一级目录，即项目的根目录，然后通过`path.resolve()`方法将项目的根目录与传入的相对路径结合起来形成最终结果。回到scripts/config.js模块，我们继续向下执行：
```js
// scripts/config.js
const resolve = p => {
  // 获取路径的别名
  const base = p.split('/')[0]
  // 查找别名是否存在
  if (aliases[base]) { 
    // 如果别名存在，则将别名对应的路径与文件名进行合并
    return path.resolve(aliases[base], p.slice(base.length + 1)) 
  } else {
    // 如果别名不存在，则将项目根路径与传入路径进行合并
    return path.resolve(__dirname, '../', p) 
  }
}
```
config.js也定义了一个`resolve()`方法，该方法接收一个路径参数p，假设p为`web/entry-runtime.js`，则第一步获取的base为web，然后到alias模块输出的对象aliases中寻找对应的别名是否存在，web模块对应的别名是存在的，它的值为：
```js
web: resolve('src/platforms/web')
```
所以会将别名的实际路径与文件名进行拼接，获取文件的真实路径。文件名的获取方法是：
```js
p.slice(base.length + 1)
```
如果传入的路径为：`dist/vue.runtime.common.js`，则会查找别名dist，该别名是不存在的，所以会执行另外一条路径，将项目根路径与传入的参数路径进行拼接，即执行下面这段代码：
```js
return path.resolve(__dirname, '../', p)
```
这与scripts/alias.js模块的实现是类似的。接下来config.js模块中定义了builds变量，代码节选如下：
```js
const builds = {
  // Runtime only (CommonJS). Used by bundlers e.g. Webpack & Browserify
  'web-runtime-cjs': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.common.js'),
    format: 'cjs',
    banner
  }
}
```
这个变量中调用`resolve()`方法生成了文件的真实路径，由于配置项采用的是`rollup.js`老版本的配置名称，在新版本中已经被废弃，所以紧接着config.js模块又定义了一个`genConfig(name)`方法来解决这个问题：
```js
function genConfig (name) {
  const opts = builds[name]
  const config = {
    input: opts.entry,
    external: opts.external,
    plugins: [
      replace({
        __WEEX__: !!opts.weex,
        __WEEX_VERSION__: weexVersion,
        __VERSION__: version
      }),
      flow(),
      buble(),
      alias(Object.assign({}, aliases, opts.alias))
    ].concat(opts.plugins || []),
    output: {
      file: opts.dest,
      format: opts.format,
      banner: opts.banner,
      name: opts.moduleName || 'Vue'
    },
    onwarn: (msg, warn) => {
      if (!/Circular/.test(msg)) {
        warn(msg)
      }
    }
  }

  if (opts.env) {
    config.plugins.push(replace({
      'process.env.NODE_ENV': JSON.stringify(opts.env)
    }))
  }

  Object.defineProperty(config, '_name', {
    enumerable: false,
    value: name
  })

  return config
}
```
这个方法的用途是将老版本的`rollup.js`配置转为新版本的格式。对于插件部分，每一个打包项目都会采用`replace`、`flow`、`buble`和`alias`插件，其余自定义的插件会合并到plugins中，通过以下代码实现：
```js
plugins: [].concat(opts.plugins || []),
```
`genConfig()`方法还判断了环境变量`NODE_ENV`是否需要被替换：
```js
if (opts.env) {
    config.plugins.push(replace({
      'process.env.NODE_ENV': JSON.stringify(opts.env)
    }))
  }
```
上述代码判断了传入的opts中是否存在env参数，如果存在，则会将代码中的`process.env.NODE_ENV`部分替换为`JSON.stringify(opts.env)`：
，如传入的env值为development，则生成的结果为带双引号的development
```bash
"development"
```
除此之外，`genConfig()`方法还将builds对象的key保存在config对象中：
```js
Object.defineProperty(config, '_name', {
    enumerable: false,
    value: name
  })
```
如果builds的key为`web-runtime-cjs`，则生成的config为：
```js
config = {
  '_name': 'web-runtime-cjs'
}
```
最后config.js模块定义了`getAllBuilds()`方法：
```js
if (process.env.TARGET) {
  module.exports = genConfig(process.env.TARGET)
} else {
  exports.getBuild = genConfig
  exports.getAllBuilds = () => Object.keys(builds).map(genConfig)
}
```
该方法首先判断环境变量TARGET是否定义，在build的三种方法中没有定义TARGET环境变量，所以会执行else中的逻辑，else逻辑中会暴露一个`getBuild()`方法和`getAllBuilds()`方法，`getAllBuilds()`方法会获取builds对象的key数组，进行遍历并调用`genConfig()`方法生成配置对象，这样`rollup`的配置就生成了。

## rollup配置过滤
我们回到scripts/build.js模块，配置生成完毕后，将对配置项进行过滤，因为每一种打包模式都将输出不同的结果，过滤部分的源码如下：
```js
// scripts/build.js
// filter builds via command line arg
if (process.argv[2]) {
  const filters = process.argv[2].split(',')
  builds = builds.filter(b => {
    return filters.some(f => b.output.file.indexOf(f) > -1 || b._name.indexOf(f) > -1)
  })
} else {
  // filter out weex builds by default
  builds = builds.filter(b => {
    return b.output.file.indexOf('weex') === -1
  })
}
```
首先分析build命令，该命令实际执行指令为：
```bash
node scripts/build.js
```
所以process.argv的内容为：
```bash
[ '/Users/sam/.nvm/versions/node/v11.2.0/bin/node',
  '/Users/sam/WebstormProjects/vue/scripts/build.js' ]
```
不存在process.argv[2]，所以会执行else中的内容：
```js
builds = builds.filter(b => {
  return b.output.file.indexOf('weex') === -1
})
```
这段代码的用途是排除`weex`的代码打包，通过output.file是否包含`weex`字符串判断是否为`weex`代码。`build:ssr`命令实际执行指令为：
```bash
node scripts/build.js "web-runtime-cjs,web-server-renderer"
```
此时process.argv的值为：
```bash
[ '/Users/sam/.nvm/versions/node/v11.2.0/bin/node',
  '/Users/sam/WebstormProjects/vue/scripts/build.js',
  'web-runtime-cjs,web-server-renderer' ]
```
process.argv[2]的值为`web-runtime-cjs,web-server-renderer`，所以会执行if中的逻辑：
```js
const filters = process.argv[2].split(',')
  builds = builds.filter(b => {
    return filters.some(f => b.output.file.indexOf(f) > -1 || b._name.indexOf(f) > -1)
```
这个方法首先将参数通过逗号分隔为一个filters数组，然后遍历builds数组，寻找output.file或_name中任一个包含filters中任一个的配置项。比如filters的第一个元素为：web-runtime-cjs，则会寻找output.file或_name中包含`web-runtime-cjs`的配置项，_name之前分析过，它指向配置项的key，此时会找到下面的配置项符合条件：
```js
'web-runtime-cjs': {
  entry: resolve('web/entry-runtime.js'),
  dest: resolve('dist/vue.runtime.common.js'),
  format: 'cjs',
  banner
}
```
那么该配置就会被保留，并最终被打包。

## rollup打包
配置过滤完之后就会调用打包函数：
```js
build(builds)
```
build函数定义如下：
```js
function build (builds) {
  let built = 0 // 当前打包项序号
  const total = builds.length // 需要打包的总次数
  const next = () => {
    buildEntry(builds[built]).then(() => {
      built++ // 打包完成后序号加1
      if (built < total) {
        next() // 如果打包序号小于打包总次数，则继续执行next()函数
      }
    }).catch(logError) // 输出错误信息
  }

  next() // 调用next()函数
}
```
`build()`函数接收builds参数，进行遍历，并调用`buildEntry()`函数执行实际的打包逻辑，`buildEntry()`函数返回一个Promise对象，如果出错，会调用`logError(e)`函数打印报错信息：
```js
function logError (e) {
  console.log(e)
}
```
打包的核心函数是`buildEntry(config)`
```js
function buildEntry (config) {
  const output = config.output // 获取config的output配置项
  const { file, banner } = output // 获取output中的file和banner
  const isProd = /min\.js$/.test(file) // 判断file中是否以min.js结尾，如果是则标记isProd为true
  return rollup.rollup(config) // 执行rollup打包
    .then(bundle => bundle.generate(output)) // 将打包的结果生成源码
    .then(({ code }) => { // 获取打包生成的源码
      if (isProd) { // 判断是否为isProd
        const minified = (banner ? banner + '\n' : '') + terser.minify(code, { // 执行代码最小化打包，并在代码标题处手动添加banner，因为最小化打包会导致注释被删除
          output: { 
            ascii_only: true // 只支持ascii字符
          },
          compress: {
            pure_funcs: ['makeMap'] // 过滤makeMap函数
          }
        }).code // 获取最小化打包的代码
        return write(file, minified, true) // 将代码写入输出路径
      } else {
        return write(file, code) // 将代码写入输出路径
      }
    })
}
```
如果理解了`rollup`的原理及`terser`的使用方法，理解上述代码并不难，这里与我们之前使用`rollup`打包不同之处在于采用了手动添加banner注释和手动输出代码文件，而之前都是`rollup`自动输出。之前我们采用的方法为：
```js
const bundle = await rollup.rollup(input) // 获取打包对象bundle
bundle.write(output) // 将打包对象输出到文件
```
而`Vue.js`采用的方法是：
```js
const bundle = await rollup.rollup(input) // 获取打包对象bundle
const { code, map } = await bundle.generate(output) // 根据bundle生成源码和source map
```
通过bundle获取源码，然后手动输出到文件中。

## 源码输出

源码输出主要是调用`write()`函数，这里需要提供3个参数：
- dest：输出文件的绝对路径，通过output.file获取；
- code：源码字符串，通过`bundle.generate()`获取；
- zip：是否需要进行gzip压缩测试，如果isProd为true，则zip为true，反之为false。
```js
function write (dest, code, zip) {
  return new Promise((resolve, reject) => {
    function report (extra) { // 输出日志函数
      console.log(blue(path.relative(process.cwd(), dest)) + ' ' + getSize(code) + (extra || '')) // 打印文件名称、文件容量和gzip压缩测试结果
      resolve()
    }

    fs.writeFile(dest, code, err => {
      if (err) return reject(err) // 如果报错则直接调用reject()方法
      if (zip) { // 如果isProd则进行gzip测试
        zlib.gzip(code, (err, zipped) => { // 通过gzip对源码进行压缩测试
          if (err) return reject(err)
          report(' (gzipped: ' + getSize(zipped) + ')') // 测试成功后获取gzip字符串长度并输出gizp容量
        })
      } else {
        report() // 输出日志
      }
    })
  })
}
```
这里有几个细节需要注意，第一是获取当前命令行路径到最终生成文件的相对路径：
```js
path.relative(process.cwd(), dest)
```
第二是调用`blue()`函数生成命令行蓝色的文本：
```js
function blue (str) {
  return '\x1b[1m\x1b[34m' + str + '\x1b[39m\x1b[22m'
}
```
第三是获取文件容量的方法：
```js
function getSize (code) {
  return (code.length / 1024).toFixed(2) + 'kb'
}
```
这三个方法不难理解，但是都非常实用，大家在开发过程中可以多多借鉴。

# 总结
大家可以发现当我们具备了基础知识后，再分析`Vue.js`的源码打包过程并不复杂，所以建议大家工作中可以借鉴这种学习方式，将基础知识点先抽离出来，单独搞明白后再攻克复杂的源码。`rollup.js`10分钟系列教程到此完结，对本教程有任何建议非常欢迎大家给我留言，教程内容较多，谢谢大家耐心看完。
