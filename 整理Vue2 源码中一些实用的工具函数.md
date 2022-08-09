## 为了降低文章难度，直接学习打包后的源码

源代码的代码[vue/vue/src/shared](https://github.com/vuejs/vue/blob/dev/src/shared/util.js)，使用了[Flow](https://flow.org/)类型，可能不太好理解。

为了降低阅读难度，直接学习源码仓库中的[打包后的 dist/vue.js 14行到379行](https://github.com/vuejs/vue/blob/dev/dist/vue.js#L14-L379)。

## 工具函数

### 1. emptyObject
```js
/*!
 * Vue.js v2.6.14
 * (c) 2014-2021 Evan You
 * Released under the MIT License.
 */
/*  */
var emptyObject = Object.freeze({});
```
冻结对象。第一层无法修改。对象中也有判断是否冻结的方法。
```js
Object.isFrozen(emptyObject); // true
```
### 2. isUndef 是否是未定义

```js
// These helpers produce better VM code in JS engines due to their
// explicitness and function inlining.
function isUndef (v) {
  return v === undefined || v === null
}
```

### 3. isDef 是否是已经定义
JavaScript中假值有六个。
```js
false
null
undefined
0
'' (空字符串)
NaN
```

为了判断准确，Vue2 源码中封装了isDef、 isTrue、isFalse函数来准确判断。
见名知意。
```js
function isDef (v) {
  return v !== undefined && v !== null
}
```
### 4. isTrue 是否是 true
见名知意。
```js
function isTrue (v) {
  return v === true
}
```

### 5. isFalse 是否是 false
见名知意。
```js
function isFalse (v) {
  return v === false
}
```
### 6. isPrimitive 判断值是否是原始值
判断是否是字符串、或者数字、或者 symbol、或者布尔值。
```js
/**
 * Check if value is primitive.
 */
function isPrimitive (value) {
  return (
    typeof value === 'string' ||
    typeof value === 'number' ||
    // $flow-disable-line
    typeof value === 'symbol' ||
    typeof value === 'boolean'
  )
}
```
### 7. isObject 判断是对象
因为 `typeof null` 是 'object'。数组等用这个函数判断也是 true
```js
/**
 * Quick object check - this is primarily used to tell
 * Objects from primitive values when we know the value
 * is a JSON-compliant type.
 */
function isObject (obj) {
  return obj !== null && typeof obj === 'object'
}

// 例子:
isObject([]) // true
// 有时不需要严格区分数组和对象
```
### 8. toRawType 转换成原始类型
`Object.prototype.toString()` 方法返回一个表示该对象的字符串。

```js
/**
 * Get the raw type string of a value, e.g., [object Object].
 */
var _toString = Object.prototype.toString;

function toRawType (value) {
  return _toString.call(value).slice(8, -1)
}

// 例子：
toRawType('') // 'String'
toRawType() // 'Undefined'
```
### 9. isPlainObject 是否是纯对象
```js
/**
 * Strict object type check. Only returns true
 * for plain JavaScript objects.
 */
function isPlainObject (obj) {
  return _toString.call(obj) === '[object Object]'
}

// 上文 isObject([]) 也是 true
// 这个就是判断对象是纯对象的方法。
// 例子：
isPlainObject([]) // false
isPlainObject({}) // true
```
### 10. isRegExp 是否是正则表达式
```js
function isRegExp (v) {
  return _toString.call(v) === '[object RegExp]'
}

// 例子：
// 判断是不是正则表达式
isRegExp(/jiapandong/) // true
```
### 11. isValidArrayIndex 是否是可用的数组索引值
数组可用的索引值是 0 ('0')、1 ('1') 、2 ('2') ...

```js
/**
 * Check if val is a valid array index.
 */
function isValidArrayIndex (val) {
  var n = parseFloat(String(val));
  return n >= 0 && Math.floor(n) === n && isFinite(val)
}
```
该全局`isFinite() `函数用来判断被传入的参数值是否为一个有限数值（`finite number`）。在必要情况下，参数会首先转为一个数值。
```js
isFinite(Infinity);  // false
isFinite(NaN);       // false
isFinite(-Infinity); // false

isFinite(0);         // true
isFinite(2e64);      // true, 在更强壮的Number.isFinite(null)中将会得到false

isFinite('0');       // true, 在更强壮的Number.isFinite('0')中将会得到false
```

### 12. isPromise 判断是否是 promise
```js
function isPromise (val) {
  return (
    isDef(val) &&
    typeof val.then === 'function' &&
    typeof val.catch === 'function'
  )
}

// 例子：
// 判断是不是Promise对象 
const p1 = new Promise(function(resolve, reject){
    resolve('jiapandong');
});
isPromise(p1); // true
```
这里用 `isDef` 判断其实相对 `isObject` 来判断 来说有点不严谨。但是够用。
### 13. toString 转字符串
转换成字符串。是数组或者对象并且对象的 `toString` 方法是 `Object.prototype.toString`，用 `JSON.stringify` 转换。
```js
/**
 * Convert a value to a string that is actually rendered.
 */
function toString (val) {
  return val == null
    ? ''
    : Array.isArray(val) || (isPlainObject(val) && val.toString === _toString)
      ? JSON.stringify(val, null, 2)
      : String(val)
}
```
### 14. toNumber 转数字
转换成数字。如果转换失败依旧返回原始字符串。
```js
/**
 * Convert an input value to a number for persistence.
 * If the conversion fails, return original string.
 */
function toNumber (val) {
  var n = parseFloat(val);
  return isNaN(n) ? val : n
}

toNumber('a') // 'a'
toNumber('1') // 1
toNumber('1a') // 1
toNumber('a1') // 'a1'
```
### 15. makeMap 生成一个 map （对象）
传入一个以逗号分隔的字符串，生成一个 map(键值对)，并且返回一个函数检测 key 值在不在这个 map 中。第二个参数是小写选项。
```js
/**
 * Make a map and return a function for checking if a key
 * is in that map.
 */
function makeMap (
  str,
  expectsLowerCase
) {
  var map = Object.create(null);
  var list = str.split(',');
  for (var i = 0; i < list.length; i++) {
    map[list[i]] = true;
  }
  return expectsLowerCase
    ? function (val) { return map[val.toLowerCase()]; }
    : function (val) { return map[val]; }
}

// Object.create(null) 没有原型链的空对象
```
### 16. isBuiltInTag 是否是内置的 tag
```js
/**
 * Check if a tag is a built-in tag.
 */
var isBuiltInTag = makeMap('slot,component', true);

// 返回的函数，第二个参数不区分大小写
isBuiltInTag('slot') // true
isBuiltInTag('component') // true
isBuiltInTag('Slot') // true
isBuiltInTag('Component') // true
```
### 17. isReservedAttribute 是否是保留的属性
```js
/**
 * Check if an attribute is a reserved attribute.
 */
var isReservedAttribute = makeMap('key,ref,slot,slot-scope,is');

isReservedAttribute('key') // true
isReservedAttribute('ref') // true
isReservedAttribute('slot') // true
isReservedAttribute('slot-scope') // true
isReservedAttribute('is') // true
isReservedAttribute('IS') // undefined
```
### 18. remove 移除数组中的中一项
```js
/**
 * Remove an item from an array.
 */
function remove (arr, item) {
  if (arr.length) {
    var index = arr.indexOf(item);
    if (index > -1) {
      return arr.splice(index, 1)
    }
  }
}
```
`splice` 其实是一个很耗性能的方法。删除数组中的一项，其他元素都要移动位置。

**引申：**`axios InterceptorManager` [拦截器源码](https://github.com/axios/axios/blob/master/lib/core/InterceptorManager.js) 中，拦截器用数组存储的。但实际移除拦截器时，只是把拦截器置为 `null` 。而不是用`splice`移除。最后执行时为 `null` 的不执行，同样效果。`axios` 拦截器这个场景下，不得不说为性能做到了很好的考虑。**因为拦截器是用户自定义的，理论上可以有无数个，所以做性能考虑是必要的。**
看如下 `axios` 拦截器代码示例：
```js
// 代码有删减
// 声明
this.handlers = [];

// 移除
if (this.handlers[id]) {
    this.handlers[id] = null;
}

// 执行
if (h !== null) {
    fn(h);
}
```
### 19. hasOwn 检测是否是自己的属性
```js
/**
 * Check whether an object has the property.
 */
var hasOwnProperty = Object.prototype.hasOwnProperty;
function hasOwn (obj, key) {
  return hasOwnProperty.call(obj, key)
}

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
### 20. cached 缓存
利用闭包特性，缓存数据
```js
/**
 * Create a cached version of a pure function.
 */
function cached (fn) {
  var cache = Object.create(null);
  return (function cachedFn (str) {
    var hit = cache[str];
    return hit || (cache[str] = fn(str))
  })
}

```

### 21. camelize 连字符转小驼峰
连字符 - 转驼峰  on-click => onClick
```js
/**
 * Camelize a hyphen-delimited string.
 */
var camelizeRE = /-(\w)/g;
var camelize = cached(function (str) {
  return str.replace(camelizeRE, function (_, c) { return c ? c.toUpperCase() : ''; })
});
```
### 22. capitalize 首字母转大写
首字母转大写
```js
/**
 * Capitalize a string.
 */
var capitalize = cached(function (str) {
  return str.charAt(0).toUpperCase() + str.slice(1)
});
```
### 23. hyphenate 小驼峰转连字符
onClick => on-click
```js
/**
 * Hyphenate a camelCase string.
 */
var hyphenateRE = /\B([A-Z])/g;
var hyphenate = cached(function (str) {
  return str.replace(hyphenateRE, '-$1').toLowerCase()
});
```
### 24. polyfillBind bind 的垫片
```js
/**
 * Simple bind polyfill for environments that do not support it,
 * e.g., PhantomJS 1.x. Technically, we don't need this anymore
 * since native bind is now performant enough in most browsers.
 * But removing it would mean breaking code that was able to run in
 * PhantomJS 1.x, so this must be kept for backward compatibility.
 */

/* istanbul ignore next */
function polyfillBind (fn, ctx) {
  function boundFn (a) {
    var l = arguments.length;
    return l
      ? l > 1
        ? fn.apply(ctx, arguments)
        : fn.call(ctx, a)
      : fn.call(ctx)
  }

  boundFn._length = fn.length;
  return boundFn
}

function nativeBind (fn, ctx) {
  return fn.bind(ctx)
}

var bind = Function.prototype.bind
  ? nativeBind
  : polyfillBind;
```
简单来说就是兼容了老版本浏览器不支持原生的 `bind` 函数。同时兼容写法，对参数的多少做出了判断，使用`call`和`apply`实现，据说参数多适合用 `apply`，少用 `call` 性能更好。

### 25. toArray 把类数组转成真正的数组
把类数组转换成数组，支持从哪个位置开始，默认从 0 开始。
```js
/**
 * Convert an Array-like object to a real Array.
 */
function toArray (list, start) {
  start = start || 0;
  var i = list.length - start;
  var ret = new Array(i);
  while (i--) {
    ret[i] = list[i + start];
  }
  return ret
}

// 例子：
function fn(){
  var arr1 = toArray(arguments);
  console.log(arr1); // [1, 2, 3, 4, 5]
  var arr2 = toArray(arguments, 2);
  console.log(arr2); // [3, 4, 5]
}
fn(1,2,3,4,5);
```
### 26. extend 合并
```js
/**
 * Mix properties into target object.
 */
function extend (to, _from) {
  for (var key in _from) {
    to[key] = _from[key];
  }
  return to
}

// 例子：
const data = { name: '恨少' };
const data2 = extend(data, { mp: '恨少', name: '是恨少啊' });
console.log(data); // { name: "是恨少啊", mp: "恨少" }
console.log(data2); // { name: "是恨少啊", mp: "恨少" }
console.log(data === data2); // true
```
### 27. toObject 转对象
```js
/**
 * Merge an Array of Objects into a single Object.
 */
function toObject (arr) {
  var res = {};
  for (var i = 0; i < arr.length; i++) {
    if (arr[i]) {
      extend(res, arr[i]);
    }
  }
  return res
}

// 数组转对象
toObject(['恨少', '恨少学习'])
// {0: '恨', 1: '少', 2: '学', 3: '习'}
```
### 28. noop 空函数
```js
/* eslint-disable no-unused-vars */
/**
 * Perform no operation.
 * Stubbing args to make Flow happy without leaving useless transpiled code
 * with ...rest (https://flow.org/blog/2017/05/07/Strict-Function-Call-Arity/).
 */
function noop (a, b, c) {}

// 初始化赋值
```
### 29. no 一直返回 false
```js
/**
 * Always return false.
 */
var no = function (a, b, c) { return false; };
/* eslint-enable no-unused-vars */
```
### 30. identity 返回参数本身
```js
/**
 * Return the same value.
 */
var identity = function (_) { return _; };
```
### 31. genStaticKeys 生成静态属性
```js
/**
 * Generate a string containing static keys from compiler modules.
 */
function genStaticKeys (modules) {
  return modules.reduce(function (keys, m) {
    return keys.concat(m.staticKeys || [])
  }, []).join(',')
}
```
### 32. looseEqual 宽松相等
由于数组、对象等是引用类型，所以两个内容看起来相等，严格相等都是不相等。
```js
var a = {};
var b = {};
a === b; // false
a == b; // false
```
所以该函数是对数组、日期、对象进行递归比对。如果内容完全相等则宽松相等。
```js
/**
 * Check if two values are loosely equal - that is,
 * if they are plain objects, do they have the same shape?
 */
function looseEqual (a, b) {
  if (a === b) { return true }
  var isObjectA = isObject(a);
  var isObjectB = isObject(b);
  if (isObjectA && isObjectB) {
    try {
      var isArrayA = Array.isArray(a);
      var isArrayB = Array.isArray(b);
      if (isArrayA && isArrayB) {
        return a.length === b.length && a.every(function (e, i) {
          return looseEqual(e, b[i])
        })
      } else if (a instanceof Date && b instanceof Date) {
        return a.getTime() === b.getTime()
      } else if (!isArrayA && !isArrayB) {
        var keysA = Object.keys(a);
        var keysB = Object.keys(b);
        return keysA.length === keysB.length && keysA.every(function (key) {
          return looseEqual(a[key], b[key])
        })
      } else {
        /* istanbul ignore next */
        return false
      }
    } catch (e) {
      /* istanbul ignore next */
      return false
    }
  } else if (!isObjectA && !isObjectB) {
    return String(a) === String(b)
  } else {
    return false
  }
}
```
### 33. looseIndexOf 宽松的 indexOf
该函数实现的是宽松相等。原生的 `indexOf` 是严格相等。
```js
/**
 * Return the first index at which a loosely equal value can be
 * found in the array (if value is a plain object, the array must
 * contain an object of the same shape), or -1 if it is not present.
 */
function looseIndexOf (arr, val) {
  for (var i = 0; i < arr.length; i++) {
    if (looseEqual(arr[i], val)) { return i }
  }
  return -1
}
```
### 34. once 确保函数只执行一次
利用闭包特性，存储状态
```js
/**
 * Ensure a function is called only once.
 */
function once (fn) {
  var called = false;
  return function () {
    if (!called) {
      called = true;
      fn.apply(this, arguments);
    }
  }
}


const fn1 = once(function(){
  console.log('哎嘿，无论你怎么调用，我只执行一次');
});

fn1(); // '哎嘿，无论你怎么调用，我只执行一次'
fn1(); // 不输出
fn1(); // 不输出
fn1(); // 不输出
```
### 35. 生命周期等
```js
var SSR_ATTR = 'data-server-rendered';

var ASSET_TYPES = [
  'component',
  'directive',
  'filter'
];

[Vue 生命周期](https://cn.vuejs.org/v2/api/#%E9%80%89%E9%A1%B9-%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E9%92%A9%E5%AD%90)

var LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured',
  'serverPrefetch'
];