![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b1b8ed2bf234ea9885f7d50f2c0743f~tplv-k3u1fbpfcp-zoom-crop-mark:3024:3024:3024:1702.awebp?)

**如何实现一个 arrify 函数，把一个值转换为一个数组？**
思考这个问题的时候，我有一点疑惑🤔，转换的规则怎么确定呢？

> JavaScript 中的基本类型有 String、Boolean、Number、Symbol、BigInt、Undefined、Null、Object
Object 中又能扩展出 Date、Function、RegExp、Array、Map、Set、自定义对象.....

JavaScript 中有很多数据类型，不同类型的值根据什么规则转换为数组，应该是可以分析归纳出来的。

我打算先来设计自己的实现思路，补充一些遗漏的基础知识，再来看看 arrify 包如何实现。

## 设计 myArrify 函数
原生方法中和转换数组相关的是 `Array.from()`，它可以根据类数组对象或可迭代对象创建出一个数组。

### 概念回顾：什么是类数组对象？
类数组对象有三个特征：

- 有 `length`属性
- 能通过索引访问
- 不是数组，无法使用 `Array.prototype`上的方法

常见的类数组对象有内置的 `arguments`、`NodeList`，还有自定义对象例如 `{ length: 1 }`
`Array.from()`根据 length 和索引元素创建出新数组
```js
Array.from({length: 1}); // [undefined]
Array.from({length: 1, 0: 'a'}) // ['a']
Array.from({length: 2, 1: 'a'}) // [undefined, 'a']
```

### 概念回顾：什么是可迭代对象？
说起可迭代对象，很容易想到 `Map`、`Set`、`String`、`Array`，他们都是 JavaScript 内置的可迭代对象，另外类数组对象 `arguments`和 `NodeList`也是可迭代的。
可迭代对象和普通对象的差异在于实现了 `@@iterator`方法，这个方法返回一个**迭代器对象（iterator）** 。在迭代环境中，会先调用 `@@iterator` 方法得到迭代器对象，再使用迭代器对象获取迭代值。
迭代器对象的特征是有一个 `next`方法，这个方法返回 `{done: true, value?: any} | {done: false, value: any}`类型的对象，done 描述迭代是否结束，value 返回本次迭代的值。获取所有迭代值本质上是多次调用 `next`方法，直到返回的 `done`为 `true` 为止。
虽然 `@@iterator` 是一个内部方法，但可以通过 `[Symbol.iterator]`属性访问到，用 `typeof object.[Symbol.iterator] === 'function'`就能判断是否是可迭代对象了。
```js
// 我们可以实现一个简单的可迭代对象：返回迭代值 0,1,2,3
const simpleIterable = {
  [Symbol.iterator]: function () {   // @@iterator方法：返回迭代器对象
    let count = 0;
    return {												 // 迭代器对象：带有 next 方法
      next: function () {            // next方法：返回迭代信息对象
        if (count === 3) {
          return {                   // 迭代信息对象：由 done 和 value 构成
            done: true
          }
        } else {
          return {
            done: false,
            value: count++
          }
        }
      }
    }
  }
}

// 获取迭代器对象：@@iterator() -> iterator
const iterator = simpleIterable[Symbol.iterator]();
// 调用 next() 获取一次迭代
console.log(iterator.next()) // {done: false, value: 0};

// 迭代环境中，自动进行操作，多次调用 next 直到 done 为 true
for (const c of simpleIterable) {
  console.log(c);
}
```

```js
// 进阶：来实现一个可迭代的类数组对象
// 由于 object.@@iterator()，在 @@iterator 里可以通过 this 访问到对象上的属性
// 但 iterator.next() 无法访问到，需要加一个闭包
const arrayLikeIterable = {
  0: "a",
  1: "b",
  2: 'c',
  length: 3,
  [Symbol.iterator]: function () {
    let index = 0;
    const origin = this;
    return {
      next (value) {
        if (index >= origin.length) {
          return {
            done: true
          }
        } else {
          return {
            done: false,
            value: origin[index++]
          }
        }
      }
    }
  }
}
```

### 基于 Array.from 实现 myArrify

数组转换的分界在于能否按顺序提取出多个元素，可迭代对象可以获取多个迭代值，类数组对象也可以根据 `length` 和索引元素提取多个值；而其他的类型的对象却无法做到，只能作为整体看待。
最终设计出转换规则：

- Null 和 Undefined 返回空数组
- String 属于可迭代对象，但希望它保持整体，`'abc' -> ['abc']`
- 类数组对象，根据 length 和索引元素创建数组
- 可迭代对象，浅拷贝迭代值创建数组
- 其他类型直接被数组包裹就好

```js
function myArrify (value) {
  if (value === null || value === undefined) {
    return [];
  }
  if (typeof value === "string") {
    return [value]
  }
  if (typeof value[Symbol.iterable] === "function") {
    return Array.from(value);
  }
  if (value.length !== undefined) {
    return Array.from(value);
  }
  return [value];
}
```
## arrify 是怎么实现的

arrify 是一个周下载量两千万的包，它只专注于一件事：把一个值转换为一个数组。
### 跑起 arrify 项目
由于不想在本地下载太多项目，我想要在远程一键跑起代码。
于是进入 [arrify 的 Github 仓库](https://github.com/sindresorhus/arrify)，输入 `Cmd + K`，再输入 `>` 出现了一个神奇的命令 **「Open in github.dev editor」**

Codespace 官网上介绍自己是 “超级快的云端开发环境”，它是支持 Visual Studio 代码的高性能虚拟机，可以在几秒内启动。
编写、运行、调试，本地 IDE 能做的都能在浏览器中完成，还能自动应用同步到 Github 账户的 VSCode 配置。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4537ec7b9610476ebab71b36a13506a0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

接下来摸索一下项目内容，看看它有什么，怎么跑起来。
浏览一眼项目目录

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f011459f912a4c7dbd9f8144a5bd3c9a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

`index.js` 想必是入口文件，`index.d.ts` 想必是类型定义，这个 `index.test-d.ts` 和 `test.js` 有什么关系吗，哪个是测试用的？不着急，打开 `package.json` 看看
```json
{
  "type": "module",		// 声明包遵循的模块化规范
  "exports": "./index.js",	// 替代 main 字段，能做出区别化设定，为不同环境和不同模块化规范指定导出文件
  "engines": {		        // 指定包所需的运行环境
    "node": ">=12"
  },
  "scripts": {
    "test": "xo && ava && tsd"
  },
  "files": [			// 发布的包内包含哪些文件
    "index.js",
    "index.d.ts"
  ]
  "devDependencies": {
    "ava": "^3.15.0",
    "tsd": "^0.14.0",
    "xo": "^0.39.1"
  }
}
```
这里有三个开发依赖

- xo：一个开箱即用的 Linter，内部是 ESLint，但 Lint 规则都预置好了，不接受 eslintrc 配置
- tsd：为类型定义编写测试，创建一个 .test-d.ts后缀的文件就行，非常方便
- ava：Node.js 环境下的测试运行器，执行根目录下 test.js 测试文件

还有一个测试启动脚本 `test: "xo && ava && tsd"`

- 先让 **xo** 对 js 和 ts 文件做 Lint
- 再交给 **ava** 跑 test.js 测试 index.js
- 最后是 **tsd** 跑 index.test-d.ts 测试 index.d.ts

依托这三个依赖，Lint 到测试全流程无需任何配置
接下来给 arrify 加一些测试用例跑一跑，看看它对各种类型数据的处理结果是否和我的版本一样。似乎有一点差异，arrify 并不会把类数组对象转换为数组。

### arrify 的转换规则
```js
export default function arrify(value) {
  if (value === null || value === undefined) {
    return [];
  }
  
  if (Array.isArray(value)) {
    return value;
  }
  
  if (typeof value === 'string') {
    return [value];
  }
  
  if (typeof value[Symbol.iterator] === 'function') {
    return [...value];
  }
  
  return [value];
}
```

arrify 的转换规则是：

- Null 和 Undefined 返回空数组
- 数组不必再转换，返回本身
- String 属于可迭代对象，但希望它保持整体作为数组元素存在
- 可迭代对象，浅拷贝迭代值创建新数组
- 其他类型直接被数组包裹就好

在 [github.com/sindresorhu…](https://github.com/sindresorhus/arrify/issues/2) 里作者解释了不转换类数组对象的原因：如果把类数组对象看作是数组而非对象，假设用户希望类数组对象作为数组中的一个元素该怎么办。API 应该保持单一明确，避免增加使用门槛，想要把类数组对象当作数组，应该由开发者在外部自行转换它，直接使用 `Array.from` 做转换会更合适。
## 我学到了什么
**基础知识回顾**

- 类数组对象是带有 `length`属性的非数组对象
- 可迭代对象本身或原型链上实现了 `[Symbol.iterable]`方法，可以根据 `typeof obj[Symbol.iterable] === "function"`判断是否是可迭代对象
迭代方法返回迭代器对象，一个带有 `next` 方法的对象，`next` 方法返回 `done` 和 `value` 构成的迭代信息

**好用的工具**

Codespaces 支持在线开发、运行、调试 Github 项目，在 Github 中输入 `Cmd + K` 快速唤出指令面板，输入 `Cmd + Shift + K`唤出命令模式面板

**一些依赖包**

- **xo:** 开箱即用的 Linter，缩减掉了 ESLint 的配置工序
- **tsd:** 支持为 TypeScript 类型定义文件编写测试
- **ava:** 简单便捷的测试运行器

**一点经验**

API 定义简化，混沌不清的部分交给外部处理，Less is better.