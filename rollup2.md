> 原文地址：https://juejin.im/post/5bf532546fb9a049e129d529

# 前言
上一篇教程中，为大家介绍了rollup.js的入门技巧，没有读过的小伙伴可以点击[这里](https://juejin.im/post/5bed8b26e51d4560336ca5b3)，本次我们将继续对rollup.js的进阶技巧进行探讨，想直接看结论的小伙伴可以直接看最后一章。

# rollup.js插件
rollup.js的插件采用可拔插设计，它帮助我们增强了rollup.js的基础功能，下面我将重点讲解四个rollup.js最常用的插件。

## resolve插件

### 为什么需要resolve插件？
上一篇教程中，我们打包的对象是本地的js代码和库，但实际开发中，不太可能所有的库都位于本地，我们会通过npm下载远程的库。这里我专门准备了一些测试库，供大家学习rollup.js使用，首先下载测试库：
```bash
npm i -S sam-test-data
```
sam-test-data库默认提供了一个UMD模块，对外暴露了两个变量a和b以及一个random函数，a是0到9之间的一个随机整数，b是0到99之间的一个随机整数，random函数的参数是一个整数，如传入100，则返回一个0到99之间的随机整数，在本地创建测试插件代码的文件夹：
```bash
mkdir src/plugin
```
创建测试代码：
```bash
touch src/plugin/main.js
```
写入以下代码：
```js
import * as test from 'sam-test-data'
console.log(test)
export default test.random
```
先不使用rollup.js打包，直接通过babel-node尝试运行代码：
```bash
babel-node
> require('./src/plugin/main.js')
{ a: 1, b: 17, random: [Function: random] }
{ default: [Function: random] }
> require('./src/plugin/main.js').default(100)
41
```
可以看到代码可以正常运行，下面我们尝试通过rollup.js打包代码，添加一个新的配置文件：
```bash
touch rollup.plugin.config.js
```
写入以下内容：
```js
import { comment } from './comment-helper-es'

export default {
  input: './src/plugin/main.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs',
    banner: comment('welcome to imooc.com', 'this is a rollup test project'),
    footer: comment('powered by sam', 'copyright 2018')
  }, {
    file: './dist/index-plugin-es.js',
    format: 'es',
    banner: comment('welcome to imooc.com', 'this is a rollup test project'),
    footer: comment('powered by sam', 'copyright 2018')
  }]
}
```
这里我提供了一个comment-helper-es模块，暴露了一个comment方法，自动读取我们的参数，并帮助生成注释，同时会在注释上方和下方添加等长的分隔符，感兴趣的小伙伴可以直接拿去用：
```js
export function comment() {
  if (arguments.length === 0) {
    return // 如果参数为0直接返回
  }
  let maxlength = 0
  for (let i = 0; i < arguments.length; i++) {
    const length = arguments[i].toString().length
    maxlength = length > maxlength ? length : maxlength // 获取最长的参数
  }
  maxlength = maxlength === 0 ? maxlength : maxlength + 1 // 在最长参数长度上再加1，为了美观
  let seperator = ''
  for (let i = 0; i < maxlength; i++) {
    seperator += '=' // 根据参数长度生成分隔符
  }
  const c = []
  c.push('/**\n') // 添加注释头
  c.push(' * ' + seperator + '\n') // 添加注释分隔符
  for (let i = 0; i < arguments.length; i++) {
    c.push(' * ' + arguments[i] + '\n') // 加入参数内容
  }
  c.push(' * ' + seperator + '\n') // 添加注释分隔符
  c.push(' **/') // 添加注释尾
  return c.join('') // 合并参数为字符串
}
```
通过rollup.js打包：
```bash
$ rollup -c rollup.plugin.config.js 

./src/plugin/main.js → ./dist/index-plugin-cjs.js, ./dist/index-plugin-es.js...
(!) Unresolved dependencies
https://rollupjs.org/guide/en#warning-treating-module-as-external-dependency
sam-test-data (imported by src/plugin/main.js)
created ./dist/index-plugin-cjs.js, ./dist/index-plugin-es.js in 13ms
```
可以看到代码生成成功了，但是sam-test-data被当做一个外部的模块被引用，我们查看dist/index-plugin-es.js源码：
```js
/**
 * ==============================
 * welcome to imooc.com
 * this is a rollup test project
 * ==============================
 **/
import * as test from 'sam-test-data';
import { random } from 'sam-test-data';

console.log(test);

var main = random;

export default main;
/**
 * ===============
 * powered by sam
 * copyright 2018
 * ===============
 **/
```
和我们原本写的代码几乎没有区别，只是通过es6的解构赋值将random函数单独从sam-test-data获取，然后赋给变量main并暴露出来。大家试想，如果我们正在编写一个Javascript类库，用户在引用我们库的时候，还需要手动去下载这个库所有的依赖，这是多么糟糕的体验。为了解决这个问题，将我们编写的源码与依赖的第三方库进行合并，rollup.js为我们提供了resolve插件。

### resolve插件的使用方法
首先，安装resolve插件：
```bash
npm i -D rollup-plugin-node-resolve
```
修改配置文件rollup.plugin.config.js：
```js
import resolve from 'rollup-plugin-node-resolve'

export default {
  input: './src/plugin/main.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }, {
    file: './dist/index-plugin-es.js',
    format: 'es'
  }],
  plugins: [
    resolve()
  ]
}
```
重新打包：
```bash
$ rollup -c rollup.plugin.config.js 

./src/plugin/main.js → ./dist/index-plugin-cjs.js, ./dist/index-plugin-es.js...
created ./dist/index-plugin-cjs.js, ./dist/index-plugin-es.js in 28ms
```
可以看到警告消除了，我们重新查看dist/index-plugin-es.js源码：
```js
const a = Math.floor(Math.random() * 10);
const b = Math.floor(Math.random() * 100);
function random(base) {
  if (base && base % 1 === 0) {
    return Math.floor(Math.random() * base) 
  } else {
    return 0
  }
}

var test = /*#__PURE__*/Object.freeze({
  a: a,
  b: b,
  random: random
});

console.log(test);

var main = random;

export default main;
```
很明显sam-test-data库的源码已经与我们的源码集成了。

### tree-shaking
下面我们修改src/plugin/main.js的源码：
```js
import * as test from 'sam-test-data'
export default test.random
```
源码中去掉了`console.log(test)`，重新打包：
```bash
rollup -c rollup.plugin.config.js
```
再次查看dist/index-plugin-es.js源码：
```js
function random(base) {
  if (base && base % 1 === 0) {
    return Math.floor(Math.random() * base) 
  } else {
    return 0
  }
}

var main = random;

export default main;
```
我们发现关于变量a和b的定义没有了，因为源码中并没有用到这两个变量。这就是ES模块著名的**tree-shaking**机制，它动态地清除没有被使用过的代码，使得代码更加精简，从而可以使得我们的类库获得更快的加载速度（容量小了，自然加载速度变快）。

### external属性
有些场景下，虽然我们使用了resolve插件，但我们仍然某些库保持外部引用状态，这时我们就需要使用external属性，告诉rollup.js哪些是外部的类库，修改rollup.js的配置文件：
```js
import resolve from 'rollup-plugin-node-resolve'

export default {
  input: './src/plugin/main.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }, {
    file: './dist/index-plugin-es.js',
    format: 'es'
  }],
  plugins: [
    resolve()
  ],
  external: ['sam-test-data']
}
```
重新打包：
```bash
rollup -c rollup.plugin.config.js
```
查看dist/index-plugin-es.js源码：
```js
import { random } from 'sam-test-data';

var main = random;

export default main;
```
可以看到虽然使用了resolve插件，sam-test-data库仍被当做外部库处理

## commonjs插件
### 为什么需要commonjs插件？
rollup.js默认不支持CommonJS模块，这里我编写了一个CommonJS模块用于测试，该模块的内容与sam-test-data完全一致，差异仅仅是前者采用了CommonJS规范，先安装这个模块：
```bash
npm i -S sam-test-data-cjs
```
新建一个代码文件：
```bash
touch src/plugin/main-cjs.js
```
写入如下代码：
```js
import test from 'sam-test-data-cjs'
console.log(test)
export default test.random
```
这段代码非常简单，接下来修改rollup.js的配置文件：
```js
import resolve from 'rollup-plugin-node-resolve'

export default {
  input: './src/plugin/main-cjs.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }, {
    file: './dist/index-plugin-es.js',
    format: 'es'
  }],
  plugins: [
    resolve()
  ]
}
```
执行打包：
```bash
rollup -c rollup.plugin.config.js 

./src/plugin/main-cjs.js → ./dist/index-plugin-cjs.js, ./dist/index-plugin-es.js...
[!] Error: 'default' is not exported by node_modules/_sam-test-data-cjs@0.0.1@sam-test-data-cjs/index.js
https://rollupjs.org/guide/en#error-name-is-not-exported-by-module-
src/plugin/main-cjs.js (1:7)
1: import test from 'sam-test-data-cjs'
          ^
```
可以看到默认情况下，rollup.js是无法识别CommonJS模块的，此时我们需要借助commonjs插件来解决这个问题。

### commonjs插件的使用方法

首先安装commonjs插件：
```bash
npm i -D rollup-plugin-commonjs
```
修改rollup.js的配置文件：
```js
import resolve from 'rollup-plugin-node-resolve'
import commonjs from 'rollup-plugin-commonjs'

export default {
  input: './src/plugin/main-cjs.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }, {
    file: './dist/index-plugin-es.js',
    format: 'es'
  }],
  plugins: [
    resolve(),
    commonjs()
  ]
}
```
重新执行打包：
```bash
rollup -c rollup.plugin.config.js
```
打包成功后，我们查看dist/index-plugin-es.js源码：
```js
const a = Math.floor(Math.random() * 10);
const b = Math.floor(Math.random() * 100);
function random(base) {
  if (base && base % 1 === 0) {
    return Math.floor(Math.random() * base) 
  } else {
    return 0
  }
}
var _samTestDataCjs_0_0_1_samTestDataCjs = {
  a, b, random
};

console.log(_samTestDataCjs_0_0_1_samTestDataCjs);
var mainCjs = _samTestDataCjs_0_0_1_samTestDataCjs.random;

export default mainCjs;
```
可以看到CommonJS模块被集成到代码中了，通过babel-node尝试执行打包后的代码：
```bash
babel-node 
> require('./dist/index-plugin-es')
{ a: 7, b: 45, random: [Function: random] }
{ default: [Function: random] }
> require('./dist/index-plugin-es').default(1000)
838
```
代码执行成功，说明我们的代码打包成功了。

### CommonJS与tree-shaking

我们修改src/plugin/main-cjs.js的源码，验证一下CommonJS模块是否支持tree-shaking特性：
```js
import test from 'sam-test-data-cjs'
export default test.random
```
与resolve中tree-shaking的案例一样，我们去掉`console.log(test)`，重新执行打包后，再查看打包源码：
```js
const a = Math.floor(Math.random() * 10);
const b = Math.floor(Math.random() * 100);
function random(base) {
  if (base && base % 1 === 0) {
    return Math.floor(Math.random() * base) 
  } else {
    return 0
  }
}
var _samTestDataCjs_0_0_1_samTestDataCjs = {
  a, b, random
};

var mainCjs = _samTestDataCjs_0_0_1_samTestDataCjs.random;

export default mainCjs;
```
可以看到源码中仍然定义了变量a和b，说明CommonJS模块不能支持tree-shaking特性，所以建议大家使用rollup.js打包时，尽量使用ES模块，以获得更精简的代码。

### UMD与tree-shaking

UMD模块与CommonJS类似，也是不能够支持tree-shaking特性的，这里我提供了一个UMD测试模块sam-test-data-umd，感兴趣的小伙伴可以自己验证一下。有的小伙伴可能会问，sam-test-data也是一个UMD模块，为什么它能够支持tree-shaking？我们打开sam-test-data的package.json一探究竟：
```json
{
  "name": "sam-test-data",
  "version": "0.0.4",
  "description": "provide test data",
  "main": "dist/sam-test-data.js",
  "module": "dist/sam-test-data-es.js"
}
```
可以看到main属性指向dist/sam-test-data.js，这是一个UMD模块，但是module属性指向dist/sam-test-data-es.js，这是一个ES模块，rollup.js默认情况下会优先寻找并加载module属性指向的模块。所以sam-test-data的ES模块被优先加载，从而能够支持tree-shaking特性。我们看一下rollup.js官方的说明：

> 在 package.json 文件的 main 属性中指向当前编译的版本。如果你的 package.json 也具有 module 字段，像 Rollup 和 webpack 2 这样的 ES6 感知工具(ES6-aware tools)将会直接导入 ES6 模块版本。

## babel插件
### 为什么需要babel插件？
在src/plugin目录下创建一个新文件main-es.js：
```bash
touch src/plugin/main-es.js
```
写入如下代码：
```js
import { a, b, random } from 'sam-test-data-es'

console.log(a, b, random)
export default (base) => {
  return random(base)
}
```
代码中采用了ES6的新特性：箭头函数，修改配置文件：
```js
import resolve from 'rollup-plugin-node-resolve'
import commonjs from 'rollup-plugin-commonjs'

export default {
  input: './src/plugin/main-es.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }, {
    file: './dist/index-plugin-es.js',
    format: 'es'
  }],
  plugins: [
    resolve(),
    commonjs()
  ]
}
```
重新执行打包：
```bash
rollup -c rollup.plugin.config.js 
```
查看dist/index-plugin-es.js源码：
```js
var mainEs = (base) => {
  return random(base)
};

export default mainEs;
```
可以看到箭头函数被保留下来，这样的代码在不支持ES6的环境下将无法运行。我们期望在rollup.js打包的过程中就能使用babel完成代码转换，因此我们需要babel插件。

### babel插件的使用方法
首先安装babel插件：
```bash
npm i -D rollup-plugin-babel
```
修改配置文件，增加babel插件的引用：
```js
import resolve from 'rollup-plugin-node-resolve'
import commonjs from 'rollup-plugin-commonjs'
import babel from 'rollup-plugin-babel'

export default {
  input: './src/plugin/main-es.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }, {
    file: './dist/index-plugin-es.js',
    format: 'es'
  }],
  plugins: [
    resolve(),
    commonjs(),
    babel()
  ]
}
```
重新打包：
```bash
rollup -c rollup.plugin.config.js
```
再次查看dist/index-plugin-es.js源码：
```js
var mainEs = (function (base) {
  return random(base);
});

export default mainEs;
```
可以看到箭头函数被转换为了function，babel插件正常工作。

## json插件
### 为什么需要json插件
在src/plugin下创建一个新文件main-json.js：
```bash
touch src/plugin/main-json.js
```
把package.json当做一个模块来引入，并打印package.json中的name和main属性：
```js
import json from '../../package.json'

console.log(json.name, json.main)
```
使用bable-node尝试执行main-json.js：
```bash
$ babel-node src/plugin/main-json.js 
rollup-test index.js
```
可以看到name和main字段都被打印出来了，babel-node可以正确识别json模块。下面修改rollup.js的配置文件：
```js
import resolve from 'rollup-plugin-node-resolve'
import commonjs from 'rollup-plugin-commonjs'
import babel from 'rollup-plugin-babel'

export default {
  input: './src/plugin/main-json.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }, {
    file: './dist/index-plugin-es.js',
    format: 'es'
  }],
  plugins: [
    resolve(),
    commonjs(),
    babel()
  ]
}
```
重新打包：
```bash
$ rollup -c rollup.plugin.config.js 

./src/plugin/main-json.js → ./dist/index-plugin-cjs.js, ./dist/index-plugin-es.js...
[!] Error: Unexpected token (Note that you need rollup-plugin-json to import JSON files)
```
可以看到默认情况下rollup.js不支持导入json模块，所以我们需要使用json插件来支持。

### json插件的使用方法
下载json插件：
```bash
npm i -D rollup-plugin-json
```
修改配置文件：
```js
import resolve from 'rollup-plugin-node-resolve'
import commonjs from 'rollup-plugin-commonjs'
import babel from 'rollup-plugin-babel'
import json from 'rollup-plugin-json'

export default {
  input: './src/plugin/main-json.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }, {
    file: './dist/index-plugin-es.js',
    format: 'es'
  }],
  plugins: [
    resolve(),
    commonjs(),
    babel(),
    json()
  ]
}
```
重新打包：
```bash
rollup -c rollup.plugin.config.js
```
查看dist/index-plugin-cjs.js源码，可以看到json文件被解析为一个对象进行处理：
```js
var name = "rollup-test";
var version = "1.0.0";
var description = "";
var main = "index.js";
var scripts = {
	test: "echo \"Error: no test specified\" && exit 1"
};
var author = "";
var license = "ISC";
var devDependencies = {
	"@babel/core": "^7.1.6",
	"@babel/plugin-external-helpers": "^7.0.0",
	"@babel/preset-env": "^7.1.6",
	rollup: "^0.67.3",
	"rollup-plugin-babel": "^4.0.3",
	"rollup-plugin-commonjs": "^9.2.0",
	"rollup-plugin-json": "^3.1.0",
	"rollup-plugin-node-resolve": "^3.4.0"
};
var dependencies = {
	epubjs: "^0.3.80",
	loadsh: "^0.0.3",
	"sam-test-data": "^0.0.4",
	"sam-test-data-cjs": "^0.0.1",
	"sam-test-data-es": "^0.0.1",
	"sam-test-data-umd": "^0.0.1"
};
var json = {
	name: name,
	version: version,
	description: description,
	main: main,
	scripts: scripts,
	author: author,
	license: license,
	devDependencies: devDependencies,
	dependencies: dependencies
};

console.log(json.name, json.main);
```
## uglify插件
uglify插件可以帮助我们进一步压缩代码的体积，首先安装插件：
```bash
npm i -D rollup-plugin-uglify
```
修改rollup.js的配置文件：
```js
import resolve from 'rollup-plugin-node-resolve'
import commonjs from 'rollup-plugin-commonjs'
import babel from 'rollup-plugin-babel'
import json from 'rollup-plugin-json'
import { uglify } from 'rollup-plugin-uglify'

export default {
  input: './src/plugin/main.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }],
  plugins: [
    resolve(),
    commonjs(),
    babel(),
    json(),
    uglify()
  ]
}
```
这里要注意的是uglify插件不支持ES模块和ES6语法，所以只能打包成非ES格式的代码，如果碰到ES6语法则会出现报错：
```bash
$ rollup -c rollup.plugin.config.js 

./src/plugin/main.js → ./dist/index-plugin-cjs.js, ./dist/index-plugin-es.js...
  19 | var main = random;
  20 | 
> 21 | export default main;
     |       ^ Unexpected token: keyword (default)
[!] (uglify plugin) Error: Unexpected token: keyword (default)
```
所以这里我们采用sam-test-data进行测试，因为这个模块采用了babel进行编译，其他几个模块uglify都不支持（因为其他几个模块使用了const，const也是ES6特性，uglify不能支持），所以大家在自己编写类库的时候要注意使用babel插件进行编译。配置完成后重新打包：
```bash
$ rollup -c rollup.plugin.config.js 

./src/plugin/main.js → ./dist/index-plugin-cjs.js...
created ./dist/index-plugin-cjs.js in 679ms
```
查看dist/index-plugin-cjs.js源码：
```js
"use strict";var a=Math.floor(10*Math.random()),b=Math.floor(100*Math.random());function random(a){return a&&a%1==0?Math.floor(Math.random()*a):0}var test=Object.freeze({a:a,b:b,random:random});console.log(test);var main=random;module.exports=main;
```
可以看到代码被最小化了，体积也减小了不少。

# rollup.js watch
## 命令行模式
rollup.js的watch模式支持监听代码变化，一旦修改代码后将自动执行打包，非常方便，使用方法是在打包指令后添加`--watch`即可：
```bash
$ rollup -c rollup.plugin.config.js  --watch

rollup v0.67.1
bundles ./src/plugin/main-json.js → dist/index-plugin-cjs.js, dist/index-plugin-es.js...
created dist/index-plugin-cjs.js, dist/index-plugin-es.js in 24ms

[2018-11-20 22:26:24] waiting for changes...
```

## API模式
rollup.js支持我们通过API来启动watch模式，在项目根目录下创建以下文件：
- rollup-watch-input-options.js：输入配置
- rollup-watch-output-options.js：输出配置
- rollup-watch-options.js：监听配置
- rollup-watch.js：调用rollup.js的API启动watch模式

为了让node能够执行我们的程序，所以采用CommonJS规范，rollup-watch-input-options.js代码如下：
```js
const json = require('rollup-plugin-json')
const resolve = require('rollup-plugin-node-resolve')
const commonjs = require('rollup-plugin-commonjs')
const babel = require('rollup-plugin-babel')
const uglify = require('rollup-plugin-uglify').uglify

module.exports = {
  input: './src/plugin/main.js',
  plugins: [
    json(),
    resolve({
      customResolveOptions: {
        moduleDirectory: 'node_modules' // 仅处理node_modules内的库
      }
    }),
    babel({
      exclude: 'node_modules/**' // 排除node_modules
    }),
    commonjs(),
    uglify() // 代码压缩
  ]
}
```
rollup-watch-output-options.js代码如下：
```js
module.exports = [{
  file: './dist/index-cjs.js',
  format: 'cjs',
  name: 'sam-cjs'
}]
```
rollup-watch-options.js代码如下：
```js
module.exports = {
  include: 'src/**', // 监听的文件夹
  exclude: 'node_modules/**' // 排除监听的文件夹
}
```
rollup-watch.js代码如下：
```js
const rollup = require('rollup')
const inputOptions = require('./rollup-watch-input-options')
const outputOptions = require('./rollup-watch-output-options')
const watchOptions = require('./rollup-watch-options')

const options = {
  ...inputOptions,
  output: outputOptions,
  watchOptions
} // 生成rollup的options

const watcher = rollup.watch(options) // 调用rollup的api启动监听

watcher.on('event', event => {
  console.log('重新打包中...', event.code)
}) // 处理监听事件

// watcher.close() // 手动关闭监听
```
通过node直接启动监听：
```bash
$ node rollup-watch.js 
重新打包中... START
重新打包中... BUNDLE_START
重新打包中... BUNDLE_END
重新打包中... END
```
之后我们再修改src/plugin/main.js的源码，rollup.js就会自动对代码进行打包。

# 总结
本教程详细讲解了rollup.js的插件、tree-shaking机制和watch模式，涉及知识点整理如下：
- rollup.js插件
	- resolve插件：集成外部模块
	- commonjs插件：支持CommonJS模块
	- babel插件：编译ES6语法，使低版本浏览器可以识别
	- json插件：支持json模块
	- uglify：代码最小化打包（不支持ES模块）
- tree-shaking：只有ES模块才支持，大幅精简代码量
- watch模式：支持命令行和API模式，实时监听代码变更
