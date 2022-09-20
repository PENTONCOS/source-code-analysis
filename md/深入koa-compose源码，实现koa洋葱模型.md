## 什么是洋葱模型

![](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/58556699/1663231726114-d8a8e02d-b5ee-47ae-81ed-b371d659588d.png)

[koa文档](https://github.com/koajs/koa/blob/master/docs/guide.md#writing-middleware)上有个经典执行顺序`gif`图：

![](https://github.com/lxchuan12/koa-compose-analysis/raw/main/images/middleware.gif)

## 环境准备

源码地址：[https://github.com/koajs/compose/...](https://github.com/koajs/compose/blob/master/index.js)

执行命令

```cmd
git clone https://github.com/koajs/compose.git
cd componse
yarn
yarn test // 执行测试用例
```

## TDD（根据测试用例来学习代码）
[什么是 TDD](https://zhuanlan.zhihu.com/p/91755900)

测试驱动开发

根据代码中已经写好的测试用例（对应目录 [compose/test/test.js](https://github.com/koajs/compose/blob/master/test/test.js)），可以通过运行测试用例的方式来学习代码。

分享一个测试用例小技巧：我们可以在测试用例处加上`only`修饰。
```js
// 例如
it.only('should work', async () => {})
```
这样我们就可以只执行当前的测试用例，不关心其他的，不会干扰调试。

### 正常流程
打开 `compose/test/test.js` 文件，看第一个测试用例。
```js
// compose/test/test.js
'use strict'

/* eslint-env jest */

const compose = require('..')
const assert = require('assert')

function wait (ms) {
  return new Promise((resolve) => setTimeout(resolve, ms || 1))
}
// 分组
describe('Koa Compose', function () {
  it.only('should work', async () => {
    const arr = []
    const stack = []

    stack.push(async (context, next) => {
      arr.push(1)
      await wait(1)
      await next()
      await wait(1)
      arr.push(6)
    })

    stack.push(async (context, next) => {
      arr.push(2)
      await wait(1)
      await next()
      await wait(1)
      arr.push(5)
    })

    stack.push(async (context, next) => {
      arr.push(3)
      await wait(1)
      await next()
      await wait(1)
      arr.push(4)
    })

    await compose(stack)({})
    // 最后输出数组是 [1,2,3,4,5,6]
    expect(arr).toEqual(expect.arrayContaining([1, 2, 3, 4, 5, 6]))
  })
}
```

### compose 函数
简单来说，`compose` 函数主要做了两件事情。

1. 接收一个参数，校验参数是数组，且校验数组中的每一项是函数。
2. 返回一个函数，这个函数接收两个参数，分别是`context`和`next`，这个函数最后返回`Promise`。
```js
/**
 * Compose `middleware` returning
 * a fully valid middleware comprised
 * of all those which are passed.
 *
 * @param {Array} middleware
 * @return {Function}
 * @api public
 */
function compose (middleware) {
  // 校验传入的参数是数组，校验数组中每一项是函数
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch(i){
      // 省略，下文讲述
    }
  }
}
```
接着我们来看 dispatch 函数。

### dispatch 函数
```js
function dispatch (i) {
  // 一个函数中多次调用报错
  // await next()
  // await next()
  if (i <= index) return Promise.reject(new Error('next() called multiple times'))
  index = i
  // 取出数组里的 fn1, fn2, fn3...
  let fn = middleware[i]
  // 最后 相等，next 为 undefined
  if (i === middleware.length) fn = next
  // 直接返回 Promise.resolve()
  if (!fn) return Promise.resolve()
  try {
    return Promise.resolve(fn(context, dispatch.bind(null, i + 1)))
  } catch (err) {
    return Promise.reject(err)
  }
}
```

`fn(context, dispatch.bind(null, i + 1)`，`i + 1` 是为了 `let fn = middleware[i]` 取`middleware`中的**下一个函数**。 也就是 `next` 是下一个中间件里的函数。也就能解释上文中的 `gif`图函数执行顺序。 测试用例中数组的最终顺序是`[1,2,3,4,5,6]`。

### 简化 compose 便于理解
自己动手调试之后，你会发现 `compose` 执行后就是类似这样的结构（省略 `try catch `判断）。
```js
// 这样就可能更好理解了。
// simpleKoaCompose
const [fn1, fn2, fn3] = stack;
const fnMiddleware = function(context){
    return Promise.resolve(
      fn1(context, function next(){
        return Promise.resolve(
          fn2(context, function next(){
              return Promise.resolve(
                  fn3(context, function next(){
                    return Promise.resolve();
                  })
              )
          })
        )
    })
  );
};
```
> 也就是说`koa-compose`返回的是一个`Promise`，从中间件（传入的数组）中取出第一个函数，传入`context`和第一个`next`函数来执行。
> 
> 第一个`next`函数里也是返回的是一个`Promise`，从中间件（传入的数组）中取出第二个函数，传入`context`和第二个`next`函数来执行。
>
> 第二个`next`函数里也是返回的是一个`Promise`，从中间件（传入的数组）中取出第三个函数，传入`context`和第三个`next`函数来执行。
>
> 第三个...
>
> 以此类推。最后一个中间件中有调用`next`函数，则返回`Promise.resolve`。如果没有，则不执行`next`函数。 这样就把所有中间件串联起来了。这也就是我们常说的洋葱模型。

![](https://github.com/lxchuan12/koa-compose-analysis/raw/main/images/middleware.png)


### 错误捕获
```js
it('should catch downstream errors', async () => {
  const arr = []
  const stack = []

  stack.push(async (ctx, next) => {
    arr.push(1)
    try {
      arr.push(6)
      await next()
      arr.push(7)
    } catch (err) {
      arr.push(2)
    }
    arr.push(3)
  })

  stack.push(async (ctx, next) => {
    arr.push(4)
    throw new Error()
  })

  await compose(stack)({})
  // 输出顺序 是 [ 1, 6, 4, 2, 3 ]
  expect(arr).toEqual([1, 6, 4, 2, 3])
})
```
相信理解了第一个测试用例和 `compose` 函数，也是比较好理解这个测试用例了。这一部分其实就是对应的代码在这里。
```js
try {
    return Promise.resolve(fn(context, dispatch.bind(null, i + 1)))
} catch (err) {
  return Promise.reject(err)
}
```
### next 函数不能调用多次
```js
it('should throw if next() is called multiple times', () => {
  return compose([
    async (ctx, next) => {
      await next()
      await next()
    }
  ])({}).then(() => {
    throw new Error('boom')
  }, (err) => {
    assert(/multiple times/.test(err.message))
  })
})
```

这一块对应的则是：

```js
index = -1
dispatch(0)
function dispatch (i) {
  if (i <= index) return Promise.reject(new Error('next() called multiple times'))
  index = i
}
```
调用两次后 `i` 和 `index` 都为 `1`，所以会报错。

`compose/test/test.js`文件中总共 300余行，还有很多测试用例可以按照文中方法自行调试。

## 总结

1. 熟悉 `koa-compose` 中间件源码
2. 学会使用测试用例调试源码
3. 学会 `jest` 部分用法