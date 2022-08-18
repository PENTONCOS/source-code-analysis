## 准备

本文就是讲`shared`模块，对应的文件路径是：[vue/packages/shared/src/index.ts](https://github.com/vuejs/core/blob/main/packages/shared/src/index.ts)


## 工具函数

### 1. EMPTY_OBJ 空对象

```js
const EMPTY_OBJ = (process.env.NODE_ENV !== 'production')
    ? Object.freeze({})
    : {};

// 例子：
// Object.freeze 是 冻结对象
// 冻结的对象最外层无法修改。
const EMPTY_OBJ_1 = Object.freeze({});
EMPTY_OBJ_1.name = 'jiapandong';
console.log(EMPTY_OBJ_1.name); // undefined

const EMPTY_OBJ_2 = Object.freeze({ props: { mp: '恨少' } });
EMPTY_OBJ_2.props.name = 'jiapandong';
EMPTY_OBJ_2.props2 = 'props2';
console.log(EMPTY_OBJ_2.props.name); // 'jiapandong'
console.log(EMPTY_OBJ_2.props2); // undefined
console.log(EMPTY_OBJ_2);
/**
 * 
 * { 
 *  props: {
     mp: "恨少",
     name: "jiapandong"
    }
 * }
 * */
```
`process.env.NODE_ENV` 是 `node` 项目中的一个环境变量，一般定义为：`development` 和`production`。根据环境写代码。比如开发环境，有报错等信息，生产环境则不需要这些报错警告。

### 2. EMPTY_ARR 空数组
```js
const EMPTY_ARR = (process.env.NODE_ENV !== 'production') ? Object.freeze([]) : [];

// 例子：
EMPTY_ARR.push(1) // 报错，也就是为啥生产环境还是用 []
EMPTY_ARR.length = 3;
console.log(EMPTY_ARR.length); // 0
```

### 3. NOOP 空函数
```js
const NOOP = () => { };

// 很多库的源码中都有这样的定义函数，比如 jQuery、underscore、lodash 等
// 使用场景：1. 方便判断， 2. 方便压缩
// 1. 比如：
const instance = {
    render: NOOP
};

// 条件
const dev = true;
if(dev){
    instance.render = function(){
        console.log('render');
    }
}

// 可以用作判断。
if(instance.render === NOOP){
 console.log('i');
}
// 2. 再比如：
// 方便压缩代码
// 如果是 function(){} ，不方便压缩代码
```
### 4. 永远返回 false 的函数
```js
/**
 * Always return false.
 */
const NO = () => false;

// 除了压缩代码的好处外。（写NO比写false少了几个字符）
// 一直返回 false
```

### 5. isOn 判断字符串是不是 on 开头，并且 on 后首字母不是小写字母
```js
const onRE = /^on[^a-z]/;
const isOn = (key) => onRE.test(key);

// 例子：
isOn('onChange'); // true
isOn('onchange'); // false
isOn('on3change'); // true
```
`onRE` 是正则。`^`符号在开头，则表示是什么开头。而在其他地方是指非。

与之相反的是：`$`符合在结尾，则表示是以什么结尾。

`[^a-z]`是指不是a到z的小写字母。

### 6. isModelListener 监听器
判断字符串是不是以`onUpdate:`开头
```js
const isModelListener = (key) => key.startsWith('onUpdate:');

// 例子：
isModelListener('onUpdate:change'); // true
isModelListener('1onUpdate:change'); // false
// startsWith 是 ES6 提供的方法
```

### 7. extend 继承 合并
说合并可能更准确些。
```js
const extend = Object.assign;

// 例子：
const data = { name: 'jiapandong' };
const data2 = extend(data, { mp: '恨少', name: '是恨少啊' });
console.log(data); // { name: "是恨少啊", mp: "恨少" }
console.log(data2); // { name: "是恨少啊", mp: "恨少" }
console.log(data === data2); // true
```

### 8. remove 移除数组的一项
```js
const remove = (arr, el) => {
  const i = arr.indexOf(el);
  if (i > -1) {
      arr.splice(i, 1);
  }
};

// 例子：
const arr = [1, 2, 3];
remove(arr, 3);
console.log(arr); // [1, 2]
```
`splice` 其实是一个很耗性能的方法。删除数组中的一项，其他元素都要移动位置。

在[之前Vue2工具函数分析内也有提到](https://github.com/PENTONCOS/source-code-analysis/blob/main/%E6%95%B4%E7%90%86Vue2%20%E6%BA%90%E7%A0%81%E4%B8%AD%E4%B8%80%E4%BA%9B%E5%AE%9E%E7%94%A8%E7%9A%84%E5%B7%A5%E5%85%B7%E5%87%BD%E6%95%B0.md#18-remove-%E7%A7%BB%E9%99%A4%E6%95%B0%E7%BB%84%E4%B8%AD%E7%9A%84%E4%B8%AD%E4%B8%80%E9%A1%B9)

### 9. hasOwn 是不是自己本身所拥有的属性
```js
const hasOwnProperty = Object.prototype.hasOwnProperty;
const hasOwn = (val, key) => hasOwnProperty.call(val, key);

// 例子：

// 特别提醒：__proto__ 是浏览器实现的原型写法，后面还会用到
// 现在已经有提供好几个原型相关的API
// Object.getPrototypeOf
// Object.setPrototypeOf
// Object.isPrototypeOf

// .call 则是函数里 this 显示指定以为第一个参数，并执行函数。

hasOwn({__proto__: { a: 1 }}, 'a') // false
hasOwn({ a: undefined }, 'a') // true
hasOwn({}, 'a') // false
hasOwn({}, 'hasOwnProperty') // false
hasOwn({}, 'toString') // false
// 是自己的本身拥有的属性，不是通过原型链向上查找的。
```

### 10. isArray 判断数组
```js
const isArray = Array.isArray;

isArray([]); // true
const fakeArr = { __proto__: Array.prototype, length: 0 };
isArray(fakeArr); // false
fakeArr instanceof Array; // true
// 所以 instanceof 这种情况 不准确
```

### 11. isMap 判断是不是 Map 对象
```js
const isMap = (val) => toTypeString(val) === '[object Map]';

// 例子：
const map = new Map();
const o = { p: 'Hello World' };

map.set(o, 'content');
map.get(o); // 'content'
isMap(map); // true
```

> ES6 提供了 Map 数据结构。它类似于对象，也是键值对的集合，但是“键”的范围不限于字符串，各种类型的值（包括对象）都可以当作键。也就是说，Object 结构提供了“字符串—值”的对应，Map 结构提供了“值—值”的对应，是一种更完善的 Hash 结构实现。如果你需要“键值对”的数据结构，Map 比 Object 更合适。

### 12. isSet 判断是不是 Set 对象
```js
const isSet = (val) => toTypeString(val) === '[object Set]';

// 例子：
const set = new Set();
isSet(set); // true
```

`ES6` 提供了新的数据结构 `Set`。它类似于数组，但是成员的值都是唯一的，没有重复的值。

`Set`本身是一个构造函数，用来生成 `Set` 数据结构。

### 13. isDate 判断是不是 Date 对象
```js
const isDate = (val) => val instanceof Date;

// 例子：
isDate(new Date()); // true

// `instanceof` 操作符左边是右边的实例。但不是很准，但一般够用了。原理是根据原型链向上查找的。

isDate({__proto__: new Date()}); // true
// 实际上是应该是 Object 才对。
// 所以用 instanceof 判断数组也不准确。
// 再比如
({__proto__: [] }) instanceof Array; // true
// 实际上是对象。
// 所以用 数组本身提供的方法 Array.isArray 是比较准确的。
```

### 14. isFunction 判断是不是函数
```js
const isFunction = (val) => typeof val === 'function';
// 判断函数有多种方法，但这个是比较常用也相对兼容性好的。
```

### 15. isString 判断是不是字符串
```js
const isString = (val) => typeof val === 'string';

// 例子：
isString('') // true
```

### 16. isSymbol 判断是不是 Symbol
```js
const isSymbol = (val) => typeof val === 'symbol';

// 例子：
let s = Symbol();

typeof s; // "symbol"
// Symbol 是函数，不需要用 new 调用。
```

`ES6` 引入了一种新的原始数据类型`Symbol`，表示独一无二的值。

### 17. isObject 判断是不是对象
```js
const isObject = (val) => val !== null && typeof val === 'object';

// 例子：
isObject(null); // false
isObject({name: 'jiapandong'}); // true
// 判断不为 null 的原因是 typeof null 其实 是 object
```

### 18. isPromise 判断是不是 Promise
```js
const isPromise = (val) => {
    return isObject(val) && isFunction(val.then) && isFunction(val.catch);
};

// 判断是不是Promise对象
const p1 = new Promise(function(resolve, reject){
  resolve('jiapandong');
});
isPromise(p1); // true

// promise 对于初学者来说可能比较难理解。但是重点内容，JS异步编程，要着重掌握。
// 现在 web 开发 Promise 和 async await 等非常常用。
```

### 19. objectToString 对象转字符串
```js
const objectToString = Object.prototype.toString;

// 对象转字符串
```

### 20. toTypeString  对象转字符串
```js
const toTypeString = (value) => objectToString.call(value);

// call 是一个函数，第一个参数是 执行函数里面 this 指向。
// 通过这个能获得 类似  "[object String]" 其中 String 是根据类型变化的
```

### 21. toRawType  对象转字符串 截取后几位
```js
const toRawType = (value) => {
    // extract "RawType" from strings like "[object RawType]"
    return toTypeString(value).slice(8, -1);
};

// 截取到
toRawType('');  'String'
```

### 22. isPlainObject 判断是不是纯粹的对象
```js
const objectToString = Object.prototype.toString;
const toTypeString = (value) => objectToString.call(value);
// 
const isPlainObject = (val) => toTypeString(val) === '[object Object]';

// 前文中 有 isObject 判断是不是对象了。
// isPlainObject 这个函数在很多源码里都有，比如 jQuery 源码和 lodash 源码等，具体实现不一样
// 上文的 isObject([]) 也是 true ，因为 type [] 为 'object'
// 而 isPlainObject([]) 则是false
const Ctor = function(){
    this.name = '我是构造函数';
}
isPlainObject({}); // true
isPlainObject(new Ctor()); // true
```

### 23. isIntegerKey 判断是不是数字型的字符串key值
```js
const isIntegerKey = (key) => isString(key) &&
    key !== 'NaN' &&
    key[0] !== '-' &&
    '' + parseInt(key, 10) === key;

// 例子:
isIntegerKey('a'); // false
isIntegerKey('0'); // true
isIntegerKey('011'); // false
isIntegerKey('11'); // true
// 其中 parseInt 第二个参数是进制。
// 字符串能用数组取值的形式取值。
//  还有一个 charAt 函数，但不常用 
'abc'.charAt(0) // 'a'
// charAt 与数组形式不同的是 取不到值会返回空字符串''，数组形式取值取不到则是 undefined
```

### 24. makeMap && isReservedProp
传入一个以逗号分隔的字符串，生成一个 `map(键值对)`，并且返回一个函数检测 `key` 值在不在这个 `map` 中。第二个参数是小写选项。
```js
/**
 * Make a map and return a function for checking if a key
 * is in that map.
 * IMPORTANT: all calls of this function must be prefixed with
 * \/\*#\_\_PURE\_\_\*\/
 * So that rollup can tree-shake them if necessary.
 */
function makeMap(str, expectsLowerCase) {
    const map = Object.create(null);
    const list = str.split(',');
    for (let i = 0; i < list.length; i++) {
        map[list[i]] = true;
    }
    return expectsLowerCase ? val => !!map[val.toLowerCase()] : val => !!map[val];
}
const isReservedProp = /*#__PURE__*/ makeMap(
// the leading comma is intentional so empty string "" is also included
',key,ref,' +
    'onVnodeBeforeMount,onVnodeMounted,' +
    'onVnodeBeforeUpdate,onVnodeUpdated,' +
    'onVnodeBeforeUnmount,onVnodeUnmounted');

// 保留的属性
isReservedProp('key'); // true
isReservedProp('ref'); // true
isReservedProp('onVnodeBeforeMount'); // true
isReservedProp('onVnodeMounted'); // true
isReservedProp('onVnodeBeforeUpdate'); // true
isReservedProp('onVnodeUpdated'); // true
isReservedProp('onVnodeBeforeUnmount'); // true
isReservedProp('onVnodeUnmounted'); // true
```

> 在书《Vuejs设计与实现》中第二章解释了`/*#_PURE_*/`的用途，如果这个函数没有被调用，tree-shaking的时候可以放心切掉不用担心有副作用。

### 25. cacheStringFunction 缓存
```js
const cacheStringFunction = (fn) => {
  const cache = Object.create(null);
  return ((str) => {
      const hit = cache[str];
      return hit || (cache[str] = fn(str));
  });
};
```

这个函数也是和上面 `MakeMap` 函数类似。只不过接收参数的是函数。

《JavaScript 设计模式与开发实践》书中的第四章 JS单例模式也是类似的实现。
```js
var getSingle = function(fn){ // 获取单例
  var result;
  return function(){
      return result || (result = fn.apply(this, arguments));
  }
};
```

### 26. hasChanged 判断是不是有变化

```js
const hasChanged = (value, oldValue) => !Object.is(value, oldValue);
```

以下是原先的源码。
```js
// compare whether a value has changed, accounting for NaN.
const hasChanged = (value, oldValue) => value !== oldValue && (value === value || oldValue === oldValue);
// 例子：
// 认为 NaN 是不变的
hasChanged(NaN, NaN); // false
hasChanged(1, 1); // false
hasChanged(1, 2); // true
hasChanged(+0, -0); // false
// Obect.is 认为 +0 和 -0 不是同一个值
Object.is(+0, -0); // false           
// Object.is 认为  NaN 和 本身 相比 是同一个值
Object.is(NaN, NaN); // true
// 场景
// watch 监测值是不是变化了

// (value === value || oldValue === oldValue)
// 为什么会有这句 因为要判断 NaN 。认为 NaN 是不变的。因为 NaN === NaN 为 false
```

`ES5`可以通过以下代码部署Object.is。
```js
Object.defineProperty(Object, 'is', {
  value: function() {x, y} {
      if (x === y) {
          // 针对+0不等于-0的情况
          return x !== 0 || 1 / x === 1 / y;
      }
      // 针对 NaN的情况
      return x !== x && y !== y;
  },
  configurable: true,
  enumerable: false,
  writable: true
});
```

### 27. invokeArrayFns  执行数组里的函数
```js
const invokeArrayFns = (fns, arg) => {
  for (let i = 0; i < fns.length; i++) {
      fns[i](arg);
  }
};

// 例子：
const arr = [
  function(val){
      console.log(val + '的博客地址是：https://blog.csdn.net/Pentoncos');
  },
  function(val){
      console.log('谷歌搜索 Pentoncos 可以找到' + val);
  },
  function(val){
      console.log('微信搜索 jiapandong 可以找到关注' + val);
  },
]
invokeArrayFns(arr, '我');
```

为什么这样写，我们一般都是一个函数执行就行。

数组中存放函数，函数其实也算是数据。这种写法方便统一执行多个函数。


### 28. toNumber 转数字
```js
const toNumber = (val) => {
    const n = parseFloat(val);
    return isNaN(n) ? val : n;
};

toNumber('111'); // 111
toNumber('a111'); // 'a111'
parseFloat('a111'); // NaN
isNaN(NaN); // true
```

其实 `isNaN` 本意是判断是不是 `NaN` 值，但是不准确的。
比如：`isNaN('a')` 为 `true`。
所以 `ES6` 有了 `Number.isNaN` 这个判断方法，为了弥补这一个API。

```js
Number.isNaN('a')  // false
Number.isNaN(NaN); // true
```

### 29. getGlobalThis 全局对象
```js
let _globalThis;
const getGlobalThis = () => {
    return (_globalThis ||
        (_globalThis =
            typeof globalThis !== 'undefined'
                ? globalThis
                : typeof self !== 'undefined'
                    ? self
                    : typeof window !== 'undefined'
                        ? window
                        : typeof global !== 'undefined'
                            ? global
                            : {}));
};
```
获取全局 `this` 指向。

初次执行肯定是 `_globalThis` 是 `undefined`。所以会执行后面的赋值语句。

如果存在 `globalThis` 就用 `globalThis`。[MDN globalThis](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/globalThis)

如果存在`self`，就用`self`。在 `Web Worker` 中不能访问到 `window` 对象，但是我们却能通过 `self` 访问到 `Worker` 环境中的全局对象。

如果存在`window`，就用`window`。

如果存在`global`，就用`global`。Node环境下，使用`global`。

如果都不存在，使用空对象。可能是微信小程序环境下。

下次执行就直接返回 `_globalThis`，不需要第二次继续判断了。这种写法值得我们学习。

##  总结
文中主要通过学习 `shared` 模块下的几十个工具函数，比如有：`isPromise`、`makeMap`、`cacheStringFunction`、`invokeArrayFns`、`getGlobalThis`等等。

平常我们工作中也是经常能使用到这些工具函数。通过学习一些简单源码，拓展视野的同时，还能落实到自己工作开发中，收益相对比较高。

