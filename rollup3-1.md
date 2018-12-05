> 原文地址：https://juejin.im/post/5bf82319f265da614b11a621

# 前言
本文是[《10分钟快速精通rollup.js——Vue.js源码打包过程深度分析》](https://juejin.im/post/5bf822c2f265da61307486c7)的前置学习教程，讲解的知识点以理解Vue.js打包源码为目标，不会做过多地展开。教程将保持rollup.js系列教程的一贯风格，大部分知识点都将提供可运行的代码案例和实际运行的结果，让大家通过教程就可以看到实现效果，省去亲自上机测试的时间。

# 1. fs的基本应用
> fs模块是Node.js提供一组文件操作API，用于对系统文件及目录进行读写操作。

## 判断文件夹是否存在
删除dist目录，创建src/vue/fs测试代码目录和index.js测试代码：
```bash
rm -rf dist
mkdir -p src/vue/fs
touch src/vue/fs/index.js
```
通过异步和同步两种方式判断dist目录是否存在：
```js
const fs = require('fs')

fs.exists('./dist', result => console.log(result))

const exists = fs.existsSync('./dist')
if (exists) {
  console.log('dist目录存在')
} else {
  console.log('dist目录不存在')
}
```
通过node执行代码：
```bash
$ node src/vue/fs/index.js

dist目录不存在
false
```
根据执行结果我们可以看到同步的任务先完成，而异步的任务会延后一些，但是同步任务会导致主线程阻塞，在实际应用过程中需要根据实际应用场景进行取舍。

## 创建文件夹
通过异步的方式创建dist目录：
```js
const fs = require('fs')

fs.exists('./dist', result => !result && fs.mkdir('./dist'))
```
通过同步的方式创建dist目录：
```js
const fs = require('fs')

if (!fs.existsSync('./dist')) {
  fs.mkdirSync('./dist')
}
```
检查dist目录是否生成：
```bash
$ ls -al
total 312
drwxr-xr-x    5 sam  staff    160 Nov 22 14:15 dist/
```

## 读取文件
我们先通过rollup.js打包代码，在dist目录下会生成index-cjs.js和index-es.js：
```bash
$ rollup -c
./src/plugin/main.js  ./dist/index-cjs.js, ./dist/index-es.js...
created ./dist/index-cjs.js, ./dist/index-es.js in 27ms
```
通过异步方式读取index-cjs.js的内容，注意读取到的文件file是一个Buffer对象，通过toString()方法可以获取到文件的文本内容：
```js
const fs = require('fs')

fs.readFile('./dist/index-cjs.js', (err, file) => {
  if (!err) console.log(file.toString()) // 打印文件内容 
}) // 通过异步读取文件内容
```
通过同步方式读取index-cjs.js的内容：
```js
const fs = require('fs')

const file = fs.readFileSync('./dist/index-cjs.js') // 通过同步读取文件内容
console.log(file.toString()) // 打印文件内容
```
运行代码，可以看到成功读取了文件内容：
```bash
$ node src/vue/fs/index.js 
/**
 * ==============================
 * welcome to imooc.com
 * this is a rollup test project
 * ==============================
 **/
'use strict';

var a = Math.floor(Math.random() * 10);
var b = Math.floor(Math.random() * 100);

# ...
```

## 写入文件

### 覆盖写入
通过异步方式读取src/vue/fs/index.js的内容，并写入dist/index.js：
```js
const fs = require('fs')

fs.readFile('./src/vue/fs/index.js', (err, file) => {
  if (!err) fs.writeFile('./dist/index.js', file, () => {
    console.log('写入成功') // 写入成功的回调
  }) // 通过异步写入文件
}) // 通过异步读取文件
```
通过同步方式实现与上面一样的功能：
```js
const fs = require('fs')

const code = fs.readFileSync('./src/vue/fs/index.js') // 同步读取文件
fs.writeFileSync('./dist/index.js', code) // 同步写入文件
```
需要注意的是writeFile()方法默认情况下会覆盖dist/index.js的内容，即先清空文件再写入。

### 追加写入
很多时候我们需要在文件末尾追加写入一些内容，可以增加flag属性进行标识，当flag的值为a时，表示追加写入：
```js
const fs = require('fs')

const code = fs.readFileSync('./src/vue/fs/index.js')
fs.writeFileSync('./dist/index.js', code, { flag: 'a' })
```
验证方法非常简单，大家可以自己尝试。

# 2. path的基本应用
> path模块是Node.js提供的用于处理文件路径的函数集合。

## 生成绝对路径
path.resolve()方法可以帮助我们生成绝对路径，创建src/vue/path测试代码路径和index.js测试代码：
```bash
mkdir -p src/vue/path
touch src/vue/path/index.js
```
写入如下测试代码：
```js
const path = require('path')

console.log(path.resolve('./dist/index.js'))
console.log(path.resolve('src', 'vue/path/index.js'))
console.log(path.resolve('/src', '/vue/path/index.js'))
console.log(path.resolve('/src', 'vue/path/index.js'))
```
测试代码执行结果：
```bash
$ node src/vue/path/index.js 

/Users/sam/WebstormProjects/rollup-test/dist/index.js
/Users/sam/WebstormProjects/rollup-test/src/vue/path/index.js
/vue/path/index.js
/src/vue/path/index.js
```
通过测试结果不难看出path.resolve()的工作机制：
- 从左往右依次拼装路径；
- 如果参数为相对路径，则会将相对路径拼接起来，再加上当前目录的绝对路径合并成一个完整路径；
- 如果参数为绝对路径，则会以最后一个参数的绝对路径为准；
- 如果参数既有绝对路径也有相对路径，则会按照从左向后的顺序进行拼接。

## 生成相对路径
在src/vue/path/index.js写入如下代码：
```js
const path = require('path')
const fs = require('fs')

const absolutePath = path.resolve('src', 'vue/path/index.js')
console.log(path.relative('./', absolutePath))
console.log(path.relative(absolutePath, './'))
```
执行代码：
```bash
$ node src/vue/path/index.js 

src/vue/path/index.js
../../../..
```
通过运行结果我们可以看到path.relative(a, b)方法提供了两个参数，返回的结果是第一个参数到第二个参数的相对路径，换句话说就是如何从第一个路径到达第二个路径：
- 第一个案例中：第一个参数是项目的根目录，第二个参数是./src/vue/path/index.js，所以第一个路径到达第二个路径的相对路径是src/vue/path/index.js
- 第二个案例中：第一个参数是./src/vue/path/index.js，第二个路径是项目的根目录，所以第一个路径到达第二个路径的相对路径是../../../..
# 3. buble的基本应用

## buble是什么？
> The blazing fast, batteries-included ES2015 compiler.

buble是一款类似babel的ES编译器，它的主要特性如下：
- 无配置，没有plugins和preset的概念，可扩展性较低，但简单易用。
- 相对较小，速度更快。
- 避免无法在ES2015中表达的代码，如`for...of`。buble不支持的功能列表：https://buble.surge.sh/guide/#unsupported-features

## buble命令行模式
全局安装buble：
```bash
npm i -g buble
```
创建buble的测试代码：
```bash
mkdir -p src/vue/buble
touch src/vue/buble/index.js
```
在src/vue/buble/index.js中写入以下内容：
```js
const a = 1 // 使用ES6新语法：const
let b = 2 // 使用ES6新语法：let
const c = () => a + b // 使用ES6新特性：箭头函数
console.log(a, b, c())
```
使用buble编译代码，并打印出结果：
```bash
$ buble src/vue/buble/index.js 

var a = 1
var b = 2
var c = function () { return a + b; }
console.log(a, b, c())
```
相比babel，buble使用起来更加简便，不再需要配置。但是bubble对某些语法是不支持的，比如`for...of`，我们修改src/vue/buble/index.js，写入如下代码：
```js
const arr = [1, 2, 3]
for (const value of arr) {
  console.log(value)
}
```
使用node运行代码：
```bash
$ node src/vue/buble/index.js 

1
2
3
```
代码可以正常运行，我们再通过buble编译代码：
```bash
buble src/vue/buble/index.js 
---
1 : const arr = [1, 2, 3]
2 : for (const value of arr) {
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
for...of statements are not supported. Use `transforms: { forOf: false }` to skip transformation and disable this error, or `transforms: { dangerousForOf: true }` if you know what you're doing (2:0)
```
可以看到buble提示`for...of statements are not supported`，所以使用buble之前一定要了解哪些语法不能被支持，以免出现编译报错。

## buble API模式
除了命令行之外，我们还可以通过API来进行编译，在代码中引入buble库：
```bash
npm i -D buble
```
在src/vue/buble目录下创建buble-build.js文件：
```bash
touch src/vue/buble/buble-build.js
```
我们在buble-build.js文件中写入以下代码，在这段代码中我们通过fs模块获取src/vue/buble/index.js文件内容，应用buble的API进行编译，编译的关键方法是buble.tranform(code)：
```js
const buble = require('buble')
const fs = require('fs')
const path = require('path')

const codePath = path.resolve('./src/vue/buble/index.js') // 获取代码的绝对路径
const file = fs.readFileSync(codePath) // 获取缓冲区文件内容
const code = file.toString() // 将缓冲区文件转为文本格式
const result = buble.transform(code) // 通过buble编译代码

console.log(result.code) // 打印buble编译的代码
```
通过node执行buble-build.js：
```bash
$ node src/vue/buble/buble-build.js 

var a = 1
var b = 2
var c = function () { return a + b; }
console.log(a, b, c())
```
编译成功！这里需要注意的是buble.transfomr()方法传入的参数必须是String类型，不能支持Buffer对象，如果将fs.readFileSync()获取的Buffer对象直接传入会引发报错。
```bash
$ node src/vue/buble/buble-build.js 
/Users/sam/WebstormProjects/rollup-test/node_modules/_magic-string@0.25.1@magic-string/dist/magic-string.cjs.js:187
        var lines = code.split('\n');
                         ^

TypeError: code.split is not a function
```

# 4. flow的基本应用
> Flow is a static checker for javascript.

flow是Javascript静态代码类型检查器，Vue.js应用flow进行类型检查。
## 应用flow静态类型检查
我们在代码中引入flow：
```bash
npm i -D flow-bin
```
修改package.json，在scripts中添加flow指令：
```json
{
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "flow": "flow"
  }
}
```
在代码根路径下执行以下指令，进行flow项目初始化：
```bash
$ npm run flow init

> rollup-test@1.0.0 flow /Users/sam/WebstormProjects/rollup-test
> flow "init"
```
此时会在项目根路径下生成.flowconfig文件，接下来我们尝试运行flow进行代码类型的静态检查：
```bash
$ npm run flow

> rollup-test@1.0.0 flow /Users/sam/WebstormProjects/rollup-test
> flow

Launching Flow server for /Users/sam/WebstormProjects/rollup-test
Spawned flow server (pid=24734)
Logs will go to /private/tmp/flow/zSUserszSsamzSWebstormProjectszSrollup-test.log
Monitor logs will go to /private/tmp/flow/zSUserszSsamzSWebstormProjectszSrollup-test.monitor_log
No errors!
```
接下来我们创建flow的测试文件：
```bash
mkdir -p src/vue/flow
touch src/vue/flow/index.js
```
先看一个官方提供的例子，在src/vue/flow/index.js中写入如下代码：
```js
/* @flow */ // 指定该文件flow检查对象
function square(n: number): number { // square的参数必须为number类型，返回值必须为number类型
  return n * n
}

console.log(square("2"))
```
flow只会检查代码顶部添加了`/* @flow */`或`// flow`的源码。这里square("2")方法传入的参数是string型，与我们定义的类型不相符，运行flow进行类型检查：
```bash
$ npm run flow

> rollup-test@1.0.0 flow /Users/sam/WebstormProjects/rollup-test
> flow

Error ┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈ src/vue/flow/index.js:6:8

Cannot call square with "2" bound to n because string [1] is
incompatible with number [2].
```
flow不仅检查出了错误，还能精确定位到出错位置。我们将代码修改为正确类型：
```js
/* @flow */
function square(n: number): number {
  return n * n
}

console.log(square(2))
```
此时我们尝试用node运行src/vue/flow/index.js：
```bash
$ node src/vue/flow/index.js 
/Users/sam/WebstormProjects/rollup-test/src/vue/flow/index.js:2
function square(n: number): number {
                 ^

SyntaxError: Unexpected token :
```
可以看到代码无法直接运行，因为node不能识别类型检查，这时我们可以通过babel-node来实现flow代码的运行，首先安装babel的flow插件：
```bash
npm i -D @babel/plugin-transform-flow-strip-types
```
修改.babelrc配置文件，增加flow插件的支持：
```json
{
  "presets": [
    "@babel/preset-env"
  ],
  "plugins": [
    "@babel/plugin-transform-flow-strip-types"
  ]
}
```
尝试babel-node运行代码：
```bash
$ babel-node src/vue/flow/index.js 
4
```
得到了正确的结果，这得益于babel的flow插件帮助我们消除flow检查部分的代码，使得代码可以正常运行。

## 自定义类型检查
flow的强大之处在于可以进行自定义类型检查，我们在项目的根目录下创建flow文件夹，并添加test.js文件：
```bash
mkdir -p flow
touch flow/test.js
```
在test.js中写入如下内容：
```js
declare type Test = {
  a?: number;
  b?: string;
  c: (key: string) => boolean;
}
```
`declare type`表示声明一个自定义类型，这个配置文件的具体含义如下：
- 对象中属性a的类型为number，该属性可以为空（`?`表示该属性可以为空）；
- 对象中属性b的类型为string，该属性可以为空；
- 对象中属性c的类型为function，只能包含一个参数，类型为string，返回值为boolean，注意c不能为空，也就是说如果指定一个对象的类型为Test，那么这个对象中**必须包含一个名称为c的属性**。flow强大之处在于不仅可以指定类型，还能规定属性的名称，确保代码的一致性。

接下来我们修改.flowconfig，在[libs]下添加flow，这样flow在初始化时会前往项目根目录下的flow文件夹中寻找并加载自定义类型：
```bash
[ignore]

[include]

[libs]
flow

[lints]

[options]

[strict]
```
接着我们在src/vue/flow下创建type.test.js文件：
```bash
touch src/vue/flow/type-test.js
```
写入如下代码，对自定义类型进行测试：
```js
/* @flow */
const obj : Test = {
  a: 1,
  b: 'b',
  c: (p) => {
	  return new String(p) instanceof String
  }
}
console.log(obj.c("c"))
```
通过flow指令进行静态检查，并通过babel-node运行代码：
```bash
$ npm run flow
> rollup-test@1.0.0 flow /Users/sam/WebstormProjects/rollup-test
> flow

No errors!

$ babel-node src/vue/flow/type-test.js 
true
```
如果代码中obj对象不定义c属性：
```js
/* @flow */
const obj : Test = {
  a: 1,
  b: 'b'
}
```
运行flow后会出现报错：
```bash
$ npm run flow

> rollup-test@1.0.0 flow /Users/sam/WebstormProjects/rollup-test
> flow

Please wait. Server is initializing (parsed files 3000): -^[[2A^[[Error ┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈ src/vue/flow/type-test.js:2:20

Cannot assign object literal to obj because property c is missing
in object literal [1] but exists in Test [2].
```

# 5. zlib的基本应用
> zlib是Node.js的内置模块，它提供通过Gzip和Deflate/Inflate实现的压缩功能。

Vue.js源码编译时仅用到zlib.gzip()方法，先了解一下gzip的用法：
```js
zlib.gzip(buffer[, options], callback)
```
参数的含义如下：
- buffer：需要压缩文件的buffer
- options：参数，可以为空
- callback：压缩成功后的回调函数

创建src/vue/zlib目录，并创建index.js文件，用于zlib测试：
```bash
mkdir -p src/vue/zlib
touch src/vue/zlib/index.js
```
尝试通过fs模块读取dist/index-cjs.js文件的内容，并通过gzip进行压缩：
```js
const fs = require('fs')
const zlib = require('zlib')

fs.readFile('./dist/index-cjs.js', (err, code) => {
  if (err) return
  console.log('原文件容量:' + code.length)
  zlib.gzip(code, (err, zipped) => {
    if (err) return
    console.log('gzip压缩后容量:' + zipped.length)
  })
})
```
通过node执行代码：
```bash
$ node src/vue/zlib/index.js 

原文件容量:657
gzip压缩后容量:329
```
值得注意的是，传入buffer进行压缩与传入string进行压缩的结果是完全一致的。我们修改代码：
```js
fs.readFile('./dist/index-cjs.js', (err, code) => {
  if (err) return
  console.log('原文件容量:' + code.toString().length)
  zlib.gzip(code.toString(), (err, zipped) => {
    if (err) return
    console.log('gzip压缩后容量:' + zipped.length)
  })
})
```
再次执行，可以看到同样的结果：
```bash
$ node src/vue/zlib/index.js 

原文件容量:657
gzip压缩后容量:329
```
所以结论是无论通过buffer还是string获取的length都是一致的，通过buffer和string进行gzip压缩后获得的结果也是一致的。

# 6. terser的基本应用
> A JavaScript parser, mangler/compressor and beautifier toolkit for ES6+.

## 为什么选择terser
terser是一个Javascript代码的压缩和美化工具，选择terser的原因有两点：
- uglify-es不再维护，而uglify-js不支持 ES6+，这一点在上一篇教程中我们已经看到了，rollup-plugin-uglify就是基于uglify-js，所以它不能够支持ES6语法；
- terser是uglify-es的一个分支，它保持了与uglify-es和uglify-js@3的API及CLI的兼容。

## terser命令行模式：
全局安装terser：
```bash
npm i -g terser
```
通过terser压缩文件：
```bash
terser dist/index-cjs.js
```
如果先输入参数再输入文件，建议增加双短划线（`--`）进行分割：
```bash
terser -c -m -o dist/index-cjs.min.js -- dist/index-cjs.js
```
各参数的含义如下：
- -c / --compress：对代码格式进行压缩
- -m / --mangle：对变量名称进行压缩
- -o / --output：指定输出文件路径

对比压缩结果，普通压缩：
```bash
$ terser dist/index-cjs.js

"use strict";var a=Math.floor(Math.random()*10);var b=Math.floor(Math.random()*100);function random(base){if(base&&base%1===0){return Math.floor(Math.random()*base)}else{return 0}}var test=Object.freeze({a:a,b:b,random:random});const a$1=1;const b$1=2;console.log(test,a$1,b$1);var main=random;module.exports=main;
```
可以看到代码间的空格被去除，代码结构更加紧凑。下面加入-c参数后再次压缩：
```bash
$ terser dist/index-cjs.js -c

"use strict";var a=Math.floor(10*Math.random()),b=Math.floor(100*Math.random());function random(base){return base&&base%1==0?Math.floor(Math.random()*base):0}var test=Object.freeze({a:a,b:b,random:random});const a$1=1,b$1=2;console.log(test,1,2);var main=random;module.exports=main;
```
加入-c后，产生如下几个变化：
- 变量定义更加紧凑；
- 将if判断改为了三目运算符；
- console.log时直接将变量替换为值。
 
下面我们使用-m参数再对比一下：
```bash
$ terser dist/index-cjs.js -m

"use strict";var a=Math.floor(Math.random()*10);var b=Math.floor(Math.random()*100);function random(a){if(a&&a%1===0){return Math.floor(Math.random()*a)}else{return 0}}var test=Object.freeze({a:a,b:b,random:random});const a$1=1;const b$1=2;console.log(test,a$1,b$1);var main=random;module.exports=main;
```
加入-m后，主要修改了变量的名称，如random函数中的形参base变成了a，同时加入-m和-c后代码变得更加精简：
```bash
$ terser dist/index-cjs.js -m -c

"use strict";var a=Math.floor(10*Math.random()),b=Math.floor(100*Math.random());function random(a){return a&&a%1==0?Math.floor(Math.random()*a):0}var test=Object.freeze({a:a,b:b,random:random});const a$1=1,b$1=2;console.log(test,1,2);var main=random;module.exports=main;
```

## terser API模式
我们可以通过API进行代码压缩，这也是Vue.js采用的方法，在项目中安装terser模块：
```bash
npm i -D terser
```
创建src/vue/terser目录，并创建index.js文件：
```bash
mkdir -p src/vue/terser
touch src/vue/terser/index.js
```
在src/vue/terser/index.js中写入如下代码，我们尝试通过fs模块读取dist/index-cjs.js文件内容，并通过terser进行压缩，这里关键的方法是`terser.minify(code, options)`：
```js
const fs = require('fs')
const terser = require('terser')

const code = fs.readFileSync('./dist/index-cjs.js').toString() // 同步读取代码文件
const minifyCode = terser.minify(code, { // 通过terser.minify进行最小化压缩
  output: {
    ascii_only: true // 仅支持ascii字符，非ascii字符将转成\u格式
  },
  compress: {
    pure_funcs: ['func'] // 如果func的返回值没有被使用，则进行替换
  }
})

console.log(minifyCode.code)
```
我们修改src/plugin/main.js的源码，用于压缩测试：
```js
import * as test from 'sam-test-data'

const a = 1
const b = 2
console.log(test, a, b)
function func() {
  return 'this is a function'
}
func() // 使用func()函数，但没有利用函数返回值，用于测试compress的pure_funcs参数
console.log('') // 加入非ascii字符，用于测试output的ascii_only参数
export default test.random
```
修改rollup.config.js配置文件，这里值得注意的是我们加入了treeshake:false的配置，因为默认情况下冗余代码会被rollup.js剔除，这样我们就无法测试terser压缩的效果了：
```js
import resolve from 'rollup-plugin-node-resolve'
import commonjs from 'rollup-plugin-commonjs'
import babel from 'rollup-plugin-babel'

export default {
  input: './src/plugin/main.js',
  output: [{
    file: './dist/index-cjs.js',
    format: 'cjs'
  }],
  plugins: [
    resolve(),
    commonjs(),
    babel()
  ],
  treeshake: false // 关闭tree-shaking特性，将不再自动删除冗余代码
}
```
应用rollup.js进行打包，并对打包后的代码进行压缩：
```bash
$ rollup -c

./src/plugin/main.js  ./dist/index-cjs.js...
created ./dist/index-cjs.js in 436ms

$ node src/vue/terser/index.js 

"use strict";var a=Math.floor(10*Math.random()),b=Math.floor(100*Math.random());function random(a){return a&&a%1==0?Math.floor(Math.random()*a):0}var test=Object.freeze({a:a,b:b,random:random}),a$1=1,b$1=2;function func(){return"this is a function"}console.log(test,a$1,b$1),console.log("\ud83d\ude01\ud83d\ude01");var main=random;module.exports=main;
```
查看压缩后的文件，发现我们的配置生效了：
- 单独调用的func()被删除，这个配置我们可以用于压缩时删除日志打印，配置方法如下：
```js
compress: {
	pure_funcs: ['func', 'console.log']
}
```
- 非ascii字符被替换：“”被替换为“\ud83d\ude01\ud83d\ude01”

# 总结
本教程主要讲解了以下知识点：
- fs模块：Node.js内置模块，用于本地文件系统处理；
- path模块：Node.js内置模块，用于本地路径解析；
- buble模块：用于ES6+语法编译；
- flow模块：用于Javascript源码静态检查；
- zlib模块：Node.js内置模块，用于使用gzip算法进行文件压缩；
- terser模块：用于Javascript代码压缩和美化。
