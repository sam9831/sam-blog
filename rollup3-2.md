> 原文地址：https://juejin.im/post/5bf823b96fb9a049e93c61a8

# 前言
本文是[《10分钟快速精通rollup.js——Vue.js源码打包过程深度分析》](https://juejin.im/post/5bf822c2f265da61307486c7)的前置学习教程，讲解的知识点以理解Vue.js打包源码为目标，不会做过多地展开。教程将保持rollup.js系列教程的一贯风格，大部分知识点都将提供可运行的代码案例和实际运行的结果，让大家通过教程就可以看到实现效果，省去亲自上机测试的时间。

> 学习本教程需要理解buble、flow和terser模块的用途，还不了解的小伙伴可以查看这篇教程：[《10分钟快速精通rollup.js——前置学习之基础知识篇》](https://juejin.im/post/5bf82319f265da614b11a621)

# 1. rollup-plugin-buble插件
> Convert ES2015 with buble.

buble插件的用途是在rollup.js打包的过程中进行代码编译，将ES6+代码编译成ES2015标准，首先在项目中引入buble插件：
```bash
npm i -D rollup-plugin-buble
```
修改./src/vue/buble/index.js的源码，写入如下内容：
```js
const a = 1 // ES6新特性：const
let b = 2 // ES6新特性：let
const c = () => a + b // ES6新特性：箭头函数
console.log(a, b, c())
```
修改rollup.plugin.config.js配置文件：
```js
import resolve from 'rollup-plugin-node-resolve'
import commonjs from 'rollup-plugin-commonjs'
import buble from 'rollup-plugin-buble'

export default {
  input: './src/vue/buble/index.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }],
  plugins: [
    resolve(),
    commonjs(),
    buble()
  ]
}
```
应用rollup.js进行编译：
```bash
$ rollup -c rollup.plugin.config.js 
./src/vue/buble/index.js  ./dist/index-plugin-cjs.js...
created ./dist/index-plugin-cjs.js in 35ms
```
查看生成文件的内容：
```bash
$ cat dist/index-plugin-cjs.js 
'use strict';

var a = 1;
var b = 2;
var c = function () { return a + b; };
console.log(a, b, c());
```
通过node执行打包后的代码：
```bash
$ node dist/index-plugin-cjs.js 
1 2 3
```

# 2. rollup-plugin-alias插件
> Provide an alias for modules in Rollup

alias插件提供了为模块起别名的功能，用过webpack的小伙伴应该对这个功能非常熟悉，首先在项目中引入alias插件：
```bash
npm i -D rollup-plugin-alias
```
创建src/vue/alias目录，并创建一个本地测试库lib.js和测试代码index.js：
```bash
mkdir -p src/vue/alias
touch src/vue/alias/index.js
touch src/vue/alias/lib.js
```
在lib.js中写入如下内容：
```js
export function square(n) {
  return n * n
}
```
在index.js中写入如下内容：
```js
import { square } from '@/vue/alias/lib' // 使用别名

console.log(square(2))
```
修改rollup.plugin.config.js的配置文件：
```js
import alias from 'rollup-plugin-alias'
import path from 'path'

const pathResolve = p => path.resolve(__dirname, p)

export default {
  input: './src/vue/alias/index.js',
  output: [{
    file: './dist/index-plugin-es.js',
    format: 'es'
  }],
  plugins: [
    alias({
      '@': pathResolve('src')
    })
  ]
}
```
这里我们提供了一个pathResolve方法，用于生成绝对路径，引入alias插件中我们传入一个对象作为参数，对象的key是模块中使用的别名，对象的value是别名对应的真实路径。使用rollup.js进行打包：
```bash
$ rollup -c rollup.plugin.config.js 

./src/vue/alias/index.js  ./dist/index-plugin-es.js...
created ./dist/index-plugin-es.js in 23ms
```
查看打包后的代码，可以看到正确识别出别名对应的路径：
```bash
$ cat dist/index-plugin-es.js 
function square(n) {
  return n * n
}

console.log(square(2));
```

# 3. rollup-plugin-flow-no-whitespace插件

flow插件用于在rollup.js打包过程中清除flow类型检查部分的代码，我们修改rollup.plugin.config.js的input属性：
```js
input: './src/vue/flow/index.js',
```
尝试直接打包flow的代码：
```bash
$ rollup -c rollup.plugin.config.js 

./src/vue/flow/index.js  ./dist/index-plugin-cjs.js...
[!] Error: Unexpected token
src/vue/flow/index.js (2:17)
1: /* @flow */
2: function square(n: number): number {
                    ^
3:   return n * n
4: }
Error: Unexpected token
```
可以看到rollup.js会抛出异常，为了解决这个问题我们引入flow插件：
```bash
npm i -D rollup-plugin-flow-no-whitespace
```
修改配置文件：
```js
import flow from 'rollup-plugin-flow-no-whitespace'

export default {
  input: './src/vue/flow/index.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }],
  plugins: [
    flow()
  ]
}
```
重新运行打包：
```bash
$ rollup -c rollup.plugin.config.js 
./src/vue/flow/index.js  ./dist/index-plugin-cjs.js...
created ./dist/index-plugin-cjs.js in 40ms
```
查看打包后的源码：
```bash
$ cat ./dist/index-plugin-cjs.js 
'use strict';

/*  */
function square(n) {
  return n * n
}

console.log(square(2));
```
可以看到flow的代码被成功清除，但是`/* @flow */`被修改为了`/*  */`，这个问题我们可以使用terser插件后进行修复，去除注释。

# 4. rollup-plugin-replace插件
> Replace content while bundling

replace插件的用途是在打包时动态替换代码中的内容，首先引入replace插件：
```bash
npm i -D rollup-plugin-replace
```
创建src/vue/replace文件夹和index.js测试文件：
```bash
mkdir -p src/vue/replace
touch src/vue/replace/index.js
```
在index.js文件中写入如下内容：
```js
const a = 1
let b = 2
if (__SAM__) { // 使用replace值__SAM__，注意这个值没有定义，如果直接运行会报错
  console.log(`__SAM__,${a},${b},${c}`) // 使用__SAM__，打包时会被替换
}
```
修改rollup.plugin.config.js配置文件：
```js
import buble from 'rollup-plugin-buble'
import replace from 'rollup-plugin-replace'

export default {
  input: './src/vue/replace/index.js',
  output: [{
    file: './dist/index-plugin-cjs.js',
    format: 'cjs'
  }],
  plugins: [
    replace({
      __SAM__: true
    }),
    buble()
  ]
}
```
我们向replace插件传入一个对象，key是_\_SAM\_\_，value是true。所以rollups.js打包时就会将\_\_SAM\_\_替换为true。值得注意的是代码使用了ES6了特性，所以需要引入buble插件进行编译，我们执行打包指令：
```bash
$ rollup -c rollup.plugin.config.js 
./src/vue/replace/index.js  ./dist/index-plugin-cjs.js...
created ./dist/index-plugin-cjs.js in 28ms
```
查看打包结果：
```bash
$ cat dist/index-plugin-cjs.js 
'use strict';

var a = 1;
var b = 2;
{
  console.log(("true," + a + "," + b + "," + c));
}
```
可以看到_\_SAM\_\_被正确替换为了true。如果大家使用的是WebStorm编辑器，使用flow语法会出现警告，这时我们需要打开设置，进入Languages & Frameworks > Javascript，将Javascript language version改为Flow。
![WebStorm flow配置](https://user-gold-cdn.xitu.io/2018/11/23/167414ba5dc5fb13?w=1480&h=736&f=png&s=232089)
除此之外，使用_\_SAM\_\_也会引发警告。
![类型错误](https://user-gold-cdn.xitu.io/2018/11/23/167414ba591d4ac9?w=1068&h=596&f=png&s=77062)
解决这个问题的办法是打开flow/test.js，添加\_\_SAM\_\_的自定义变量（declare var），这样警告就会消除：
```js
declare type Test = {
  a?: number; // a的类型为number，可以为空
  b?: string; // b的类型为string，可以为空
  c: (key: string) => boolean; // c的类型为function，只能包含一个参数，类型为string，返回值为boolean，注意c不能为空
}

declare var __SAM__: boolean; // 添加自定义变量
```
# 5. rollup-plugin-terser插件
> Rollup plugin to minify generated es bundle

terser插件帮助我们在rollup.js打包过程中实现代码压缩，首先引入terser插件：
```bash
npm i -D rollup-plugin-terser
```
修改src/vue/replace/index.js的源码，这里我们做一轮综合测试，将我们之前学习的插件全部应用进来：
```js
/* @flow */
import { square } from '@/vue/alias/lib' // 通过别名导入本地模块
import { random } from 'sam-test-data' // 导入es模块
import { a as cjsA } from 'sam-test-data-cjs' // 导入commonjs模块

const a: number = 1 // 通过flow进行类型检查
let b: number = 2 // 使用ES6新特性：let
const c: string = '' // 加入非ascii字符
if (__SAM__) { // 使用replace字符串
  console.log(`__SAM__,${a},${b},${c}`) // 使用ES6新特性``模板字符串
}
export default {
  a, b, c, d: __SAM__, square, random, cjsA
} // 导出ES模块
```
我们希望通过上述代码验证以下功能：
- replace插件：代码中的\_\_SAM\_\_被正确替换；
- flow插件：正确去掉flow的类型检查代码；
- buble插件：将ES6+代码编译为ES2015；
- alias插件：将模块中'@'别名替换为'src'目录；
- commonjs插件：支持CommonJS模块；
- resovle插件：合并外部模块代码；
- terser插件：代码最小化打包。

修改rollup.plugin.config.js配置文件：
```js
import resolve from 'rollup-plugin-node-resolve'
import commonjs from 'rollup-plugin-commonjs'
import buble from 'rollup-plugin-buble'
import replace from 'rollup-plugin-replace'
import flow from 'rollup-plugin-flow-no-whitespace'
import { terser } from 'rollup-plugin-terser'
import alias from 'rollup-plugin-alias'
import path from 'path'

const pathResolve = p => path.resolve(__dirname, p)

export default {
  input: './src/vue/replace/index.js',
  output: [{
    file: './dist/index-plugin-es.js',
    format: 'es'
  }],
  plugins: [
    replace({
      __SAM__: true
    }),
    flow(),
    buble(),
    alias({
      '@': pathResolve('src')
    }),
    commonjs(),
    resolve(),
    terser({
      output: {
        ascii_only: true // 仅输出ascii字符
      },
      compress: {
        pure_funcs: ['console.log'] // 去掉console.log函数
      }
    }),
  ]
}
```
terser插件的配置方法与API模式基本一致，打包代码：
```bash
$ rollup -c rollup.plugin.config.js 
./src/vue/replace/index.js  ./dist/index-plugin-es.js...
created ./dist/index-plugin-es.js in 308ms
```
查看打包后的代码文件：
```bash
$ cat dist/index-plugin-es.js 
function square(a){return a*a}function random(a){return a&&a%1==0?Math.floor(Math.random()*a):0}var a$1=Math.floor(10*Math.random()),b$1=Math.floor(100*Math.random());function random$1(a){return a&&a%1==0?Math.floor(Math.random()*a):0}var _samTestDataCjs_0_0_1_samTestDataCjs={a:a$1,b:b$1,random:random$1},_samTestDataCjs_0_0_1_samTestDataCjs_1=_samTestDataCjs_0_0_1_samTestDataCjs.a,a$2=1,b$2=2,c="\ud83d\ude00",index={a:a$2,b:b$2,c:c,d:!0,square:square,random:random,cjsA:_samTestDataCjs_0_0_1_samTestDataCjs_1};export default index;
```
可以看到各项特性全部生效，尝试通过babel-node运行代码：
```bash
$ babel-node 
> require('./dist/index-plugin-es')
{ default:
   { a: 1,
     b: 2,
     c: '',
     d: true,
     square: [Function: square],
     random: [Function: random],
     cjsA: 4 } }
```
代码成功运行，说明打包过程成功。

# 6. intro和outro配置
intro和outro属性与我们之前讲解的banner和footer属性类似，都是用来为代码添加注释。那么这四个属性之间有什么区别呢？首先了解一下rollup.js官网对这四个属性的解释：
- intro：在打包好的文件的块的内部(wrapper内部)的最顶部插入一段内容
- outro：在打包好的文件的块的内部(wrapper内部)的最底部插入一段内容
- banner：在打包好的文件的块的外部(wrapper外部)的最顶部插入一段内容
- footer：在打包好的文件的块的外部(wrapper外部)的最底部插入一段内容

简单地说就是intro和outro添加的注释在代码块的内部，banner和footer在外部，下面举例说明，修改src/plugin/main.js代码：
```js
const a = 1
console.log(a)
export default a
```
修改rollup.config.js的配置：
```js
export default {
  input: './src/plugin/main.js',
  output: [{
    file: './dist/index-cjs.js',
    format: 'cjs',
    banner: '// this is banner',
    footer: '// this is footer',
    intro: '// this is a intro comment',
    outro: '// this is a outro comment',
  }]
}
```
打包代码：
```bash
$ rollup -c

./src/plugin/main.js  ./dist/index-cjs.js...
created ./dist/index-cjs.js in 11ms
```
输出结果：
```bash
$ cat dist/index-cjs.js 
// this is banner
'use strict';

// this is a intro comment

const a = 1;
console.log(a);

module.exports = a;

// this is a outro comment
// this is footer
```
可以看到banner和footer的注释在最外层，而intro和outro的注释在内层，intro的注释在'use strict'下方，所以如果要给代码最外层包裹一些元素，如module.exports或export default之类的，需要使用intro和outro，在Vue.js源码打包过程中就使用了intro和outro为代码添加模块化特性。我们还可以将intro和outro配置写入plugins的方法来实现这个功能：
```js
export default {
  input: './src/plugin/main.js',
  output: [{
    file: './dist/index-cjs.js',
    format: 'cjs',
    banner: '// this is banner',
    footer: '// this is footer'
  }],
  plugins: [
    {
      intro: '// this is a intro comment',
      outro: '// this is a outro comment'
    }
  ]
}
```
该配置文件生成的效果与之前完全一致。

# 总结
本教程主要为大家讲解了以下知识点：
- rollup-plugin-buble插件：编译ES6+语法为ES2015，无需配置，比babel更轻量；
- rollup-plugin-alias插件：替换模块路径中的别名；
- rollup-plugin-flow-no-whitespace插件：去除flow静态类型检查代码；
- rollup-plugin-replace插件：替换代码中的变量为指定值；
- rollup-plugin-terser插件：代码压缩，取代uglify，支持ES模块。
- intro和outro配置：在代码块内添加代码注释。
