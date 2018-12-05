> 原文地址：https://juejin.im/post/5bed8b26e51d4560336ca5b3

# 为什么要学习rollup.js
rollup.js是Javascript的ES模块打包器，我们熟知的Vue、React等诸多知名框架或类库都通过rollup.js进行打包。与Webpack偏向于应用打包的定位不同，rollup.js更专注于Javascript类库打包（虽然rollup.js也可以提供资源打包，但显然这不是它的强项）。在我们学习Vue和React等框架源码或者自己编写Javascript类库时，rollup.js是一条必经之路。

# rollup.js的工作原理
rollup.js可以将我们自己编写的Javascript代码（通过插件可以支持更多语言，如Tyepscript）与第三方模块打包在一起，形成一个文件，该文件可以是一个库（Library）或者一个应用（App），在打包过程中可以应用各类插件实现特定功能。下图揭示了rollup.js的运行机制：
![rollup.js运行机制](https://user-gold-cdn.xitu.io/2018/11/15/16717e7848beb2ff?w=898&h=323&f=png&s=37289)
rollup.js默认采用ES模块标准，我们可以通过rollup-plugin-commonjs插件使之支持CommonJS标准。

# 安装rollup.js
> rollup.js的安装依赖于nodejs，之前的手记中我曾详细介绍如何通过nvm管理nodejs版本，需要了解的小伙伴可以点击[这里](https://juejin.im/post/5bd6d7145188253682167e54)

## 全局安装rollup.js
首先全局安装rollup：
```bash
npm i rollup -g
```

## rollup.js打包实例
安装成功后，我们尝试使用rollup做一个简单的案例，创建src目录：
```bash
mkdir src
```
在src目录下创建a.js：
```bash
vim src/a.js
```
写入如下代码，这个模块非常简单，仅仅对外暴露一个变量a：
```javascript
const a = 1
export default a
```
在src目录下再创建main.js：
```bash
vim src/main.js
```
写入如下代码，这个模块会引入模块a，并对外暴露一个function：
```javascript
import a from './a.js'
  
export default function() {
  console.log(a)
}
```
通过rollup指令，我们可以快速地预览打包后的源码，这点和babel非常类似：
```bash
$ rollup src/main.js -f es

src/main.js  stdout...
const a = 1;

function main() {
  console.log(a);
}

export default main;
created stdout in 26ms
```
需要注意的是rollup必须带有`-f`参数，否则会报错：
```bash
$ rollup src/main.js

src/main.js  stdout...
[!] Error: You must specify output.format, which can be one of 'amd', 'cjs', 'system', 'esm', 'iife' or 'umd'
https://rollupjs.org/guide/en#output-format-f-format
```
rollup的报错提示非常棒，非常有利于我们定位错误和修复问题。通过上面的错误提示，我们了解到`-f`的值可以为'amd'、'cjs'、'system'、'esm'（'es'也可以）、'iife'或'umd'中的任何一个。`-f`参数是`--format`的缩写，它表示生成代码的格式，amd表示采用AMD标准，cjs为CommonJS标准，esm（或es）为ES模块标准。接着我们把这段代码输出到一个文件中：
```bash
$ rollup src/main.js -f es -o dist/bundle.js

src/main.js  dist/bundle.js...
created dist/bundle.js in 29ms
```
参数`-o`指定了输出的路径，这里我们将打包后的文件输出到dist目录下的bundle.js，这个文件内容与我们之前预览的内容是完全一致的。我们再输出一份CommonJS格式的代码：
```bash
$ rollup src/main.js --format cjs --output.file dist/bundle-cjs.js

src/main.js  dist/bundle-cjs.js...
created dist/bundle-cjs.js in 27ms
```
参数`--output.file`是`-o`的全称，它们是等价的，输出后我们在dist目录下会多一个bundle-cjs.js文件，查看这个文件的内容：
```javascript
'use strict';
  
const a = 1;

function main() {
  console.log(a);
}

module.exports = main;
```
可以看到代码采用CommonJS标准编写，并且将a.js和main.js两个文件进行了融合。

## 验证rollup.js打包结果
在打包成功后，我们尝试运行dist/bundle-cjs.js代码：
```bash
$ node
> const m = require('./dist/bundle-cjs.js')
> m()
1 
```
我们接着尝试运行之前输出的ES标准代码dist/bundle.js，由于nodejs并不支持ES标准，直接运行会报错：
```bash
$ node
> require('./dist/bundle.js')()
/Users/sam/Desktop/rollup-test/dist/bundle.js:7
export default main;
^^^^^^

SyntaxError: Unexpected token export
```
babel为我们提供了一个工具：babel-node，它可以在运行时将ES标准的代码转换为CommonJS格式，从而使得运行ES标准的代码成为可能，首先全局安装babel-node及相关工具，@babel/node包含babel-node，@babel/cli包含babel，而这两个工具都依赖@babel/core，所以建议都安装：
```bash
npm i @babel/core @babel/node @babel/cli -g
```
这里要注意的是babel 7改变了npm包的名称，之前的babel-core和babel-cli已经被弃用，所以安装老版本babel的同学建议先卸载：
```bash
npm uninstall babel-cli babel-core -g
```
然后到代码的根目录下，初始化项目：
```bash
npm init
```
一路回车后，在代码根目录下创建babel的配置文件.babelrc，写入如下配置
```bash
{
  "presets": ["@babel/preset-env"]
}
```
完成babel配置后安装babel的依赖：
```bash
npm i -D @babel/core @babel/preset-env
```
尝试通过babel编译代码：
```bash
$ babel dist/bundle.js 
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.default = void 0;
var a = 1;

function main() {
  console.log(a);
}

var _default = main;
exports.default = _default;
```
可以看到ES模块代码被编译成了CommonJS格式，下面通过babel-node运行代码：
```bash
$ babel-node 
> require('./dist/bundle.js')
{ default: [Function: main] }
> require('./dist/bundle.js').default()
1
```
注意babel会认为`export default function()`是一个名称为default的函数，如果想更改这个函数名称，可以修改main.js：
```javascript
import a from './a.js'
  
export function test() {
  console.log(a)
}
```
重写打包后通过babel-node运行：
```bash
$ rollup -f es --file dist/bundle.js src/main.js 

src/main.js  dist/bundle.js...
created dist/bundle.js in 26ms

$ babel-node 
> require('./dist/bundle.js').test()
1
```
注意这里的`--file`定价于`-o`和`--output.file`，通过上述案例，我们完成了rollup打包的基本操作，并验证了打包结果。但很多时候我们不会这样操作，因为直接使用命令行功能单一，而且无法使用插件，所以我们需要借助配置文件来操作。

# rollup.js配置文件
首先在代码根目录下创建rollup.config.js文件：
```bash
touch rollup.config.js
```
写入如下配置：
```js
export default {
  input: './src/main.js',
  output: [{
    file: './dist/index-cjs.js',
    format: 'cjs',
    banner: '// welcome to imooc.com',
    footer: '// powered by sam'
  }, {
    file: './dist/index-es.js',
    format: 'es',
    banner: '// welcome to imooc.com',
    footer: '// powered by sam'
  }]
}
```
rollup的配置文件非常容易理解，这里有几点需要说明：
- rollup的配置文件需要采用ES模块标准编写
- input表示入口文件的路径（老版本为entry，已经废弃）
- output表示输出文件的内容，它允许传入一个对象或一个数组，当为数组时，依次输出多个文件，它包含以下内容：
	- output.file：输出文件的路径（老版本为dest，已经废弃）
	- output.format：输出文件的格式
	- output.banner：文件头部添加的内容
	- output.footer：文件末尾添加的内容

通过`rollup -c`指令进行打包，rollup.js会自动寻找名称为rollup.config.js的配置文件：
```bash
$ rollup -c

./src/main.js  ./dist/index-cjs.js, ./dist/index-es.js...
created ./dist/index-cjs.js, ./dist/index-es.js in 13ms
```
查看dist/index-es.js文件：
```javascript
// welcome to imooc.com
const a = 1;

function test() {
  console.log(a);
}

export { test };
// powered by sam
```
代码的内容与命令行生成的无异，但头部和末尾添加了自定义的注释信息。接着我们修改配置文件的名称，并通过`-c`参数指定配置文件进行打包：
```bash
$ mv rollup.config.js rollup.config.dev.js
$ rollup -c rollup.config.dev.js 

./src/main.js  ./dist/index-cjs.js, ./dist/index-es.js...
created ./dist/index-cjs.js, ./dist/index-es.js in 13ms
```

# rollup.js api打包

## 编写rollup.js配置
很多时候命令行和配置文件的打包方式无法满足需求，我们需要更加个性化的打包方式，这时我们可以考虑通过rollup.js的api进行打包，创建rollup-input-options.js，这是输入配置，我们单独封装一个模块，提高复用性和可扩展性：
```bash
touch rollup-input-options.js
```
在输入配置文件中加入以下内容，需要注意的是这个文件必须为CommonJS格式，因为需要使用nodejs来执行：
```js
module.exports = {
  input: './src/main.js'
}
```
再添加一个输出配置文件：
```bash
touch rollup-output-options.js
```
在输出配置文件我们仍然使用一个数组，实现多种文件格式的输出，需要注意的是umd格式必须指定模块的名称，通过name属性来实现：
```javascript
module.exports = [{
  file: './dist/index-cjs.js',
  format: 'cjs',
  banner: '// welcome to imooc.com',
  footer: '// powered by sam'
}, {
  file: './dist/index-es.js',
  format: 'es',
  banner: '// welcome to imooc.com',
  footer: '// powered by sam',
}, {
  file: './dist/index-amd.js',
  format: 'amd',
  banner: '// welcome to imooc.com',
  footer: '// powered by sam',
}, {
  file: './dist/index-umd.js',
  format: 'umd',
  name: 'sam-umd', // 指定文件名称
  banner: '// welcome to imooc.com',
  footer: '// powered by sam',
}]
```

## 编写rollup.js build代码
接下来我们要在当前项目中安装rollup库：
```bash
npm i -D rollup
```
创建一个rollup-build文件，通过这个文件来调用rollup的api：
```bash
touch rollup-build.js
```
rollup-build的源码如下：
```js
const rollup = require('rollup')
const inputOptions = require('./rollup-input-options')
const outputOptions = require('./rollup-output-options')

async function rollupBuild(input, output) {
  const bundle = await rollup.rollup(input) // 根据input配置进行打包
  console.log(`正在生成：${output.file}`)
  await bundle.write(output) // 根据output配置输出文件
  console.log(`${output.file}生成成功！`)
}

(async function () {
  for (let i = 0; i < outputOptions.length; i++) {
    await rollupBuild(inputOptions, outputOptions[i])
  }
})()
```
代码的核心有两点：
- 通过`rollup.rollup(input)`得到打包对象
- 通过`bundle.write(output)`输出打包文件

这里我们还可以通过async和await实现同步操作，因为`bundle.write(output)`是异步的，会返回Promise对象，我们可以借助async机制实现按配置顺序依次打包。执行rollup-build文件：
```bash
$ node rollup-build.js 
正在生成：./dist/index-cjs.js
./dist/index-cjs.js生成成功！
正在生成：./dist/index-es.js
./dist/index-es.js生成成功！
正在生成：./dist/index-amd.js
./dist/index-amd.js生成成功！
正在生成：./dist/index-umd.js
./dist/index-umd.js生成成功！
```
查看dist/index-umd.js文件：
```js
(function (global, factory) {
  typeof exports === 'object' && typeof module !== 'undefined' ? factory(exports) :
  typeof define === 'function' && define.amd ? define(['exports'], factory) :
  (factory((global['sam-umd'] = {})));
}(this, (function (exports) {
	// ...
}
```
可以看到index-umd.js文件中在global全局变量中添加了sam-umd属性，这就是我们之前需要在umd配置中添加name属性的原因。

# 总结
本文向大家介绍了rollup.js的三种打包方式：命令行、配置文件和API，在下一篇教程中我将继续为大家介绍更多rollup.js的特性，如Tree-shaking、watch等，还会详细演示各种插件的用途及用法，敬请关注。
