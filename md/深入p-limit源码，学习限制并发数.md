## 前言

日常业务中，我们经常会遇到多个请求同时并发的场景，比如页面的初始化请求。我们使用`Promise.all`等API就可以轻松实现。

但如果要求我们去对这些并发请求控制呢？ 比如10个请求，单次并发只允许6个，当成功一个就接着请求下一个，直至请求完成。针对这样的场景，sindresorhus大神早已写了一个库`p-limit`。

那具体如何实现呢？我们一起来看一下。

## 基本使用

p-limit是一个限制并发的库。

源码地址：[https://github.com/sindresorhus/p-limit](https://github.com/sindresorhus/p-limit)

```js
import pLimit from 'p-limit';
// 1.生成限流函数（限流为1）
const limit = pLimit(1);
// 2.将需要的异步操作通过箭头函数包裹一层，让限流函数接管执行
const input = [
	limit(() => fetchSomething('foo')),
	limit(() => fetchSomething('bar')),
	limit(() => doSomething())
];

// 3.限流函数返回的也是promise，我们同样可以通过Promise.all接受结果；
// 而且同一时间只会执行一个请求（异步操作）
const result = await Promise.all(input);
console.log(result);
```

使用起来还是很简洁的，只是多了**生成限流函数**和**让限流函数接管执行**两个步骤，就可以做到并发控制。

## 源码解析

### pLimit

首先来看一下初始化的`pLimit`函数：
```js
export default function pLimit(concurrency) {
   // 验证并发数的合法性
   if (!((Number.isInteger(concurrency) || concurrency === Number.POSITIVE_INFINITY) && concurrency > 0)) {
     throw new TypeError('Expected `concurrency` to be a number from 1 and up');
   }
   // 初始化队列
   const queue = new Queue();
   // 初始化计数器
   let activeCount = 0;
   // 返回generator？
   return generator;
}
```
可以看到`pLimit`这个函数主要是判断传入并发值的合法性，初始化`activeCount`（计数器），初始化`queue`队列，`queue`是另外的一个库[yocto-queue](https://github.com/sindresorhus/yocto-queue)，实现了一个队列的结构，如果想要深入了解`yocto-queue`，请移步：

[深入yocto-queue源码，60余行代码实现一个链表队列](https://github.com/PENTONCOS/source-code-analysis/blob/main/md/%E6%B7%B1%E5%85%A5yocto-queue%E6%BA%90%E7%A0%81%EF%BC%8C60%E4%BD%99%E8%A1%8C%E4%BB%A3%E7%A0%81%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E9%93%BE%E8%A1%A8%E9%98%9F%E5%88%97.md)

在`p-limit`中主要用到了`yocto-queue`的两个方法：`enqueue`(入队)，`dequeue`(出队)。 多个 `generator` 函数会共用一个队列。

### generator

`generator`为初始化`pLimit`后的入口函数，接收异步函数和剩余参数，调用`Promise`将`resolve`传递给`enqueue`函数。 每个 `generator` 函数执行会将一个异步函数压入队列。

```js
 const generator = (fn, ...args) => new Promise(resolve => {
   // 调用enqueue函数，传入fn, resolve, args
   // resolve为promise返回的执行成功的函数
   enqueue(fn, resolve, args);
 });
```

接下来顺藤摸瓜再来看一下`enqueue`函数做了哪些事情。

### enqueue

`enqueue`主要是用于处理异步函数的入队操作。

```js
 const enqueue = (fn, resolve, args) => {
  // 入列：将函数fn的执行让run函数接管，并通过bind函数包裹一层，放入到queue队列中
  // run函数下一小节介绍，只需要知道它接管了fn函数执行
  // bind的作用就是让函数run先不要执行，只是入列，在出列后再执行
   queue.enqueue(run.bind(undefined, fn, resolve, args));
   // 自执行的async函数
   (async () => {
    // 延时：这个延时是通过Promise来实现微任务延时，让下面的出列操作单次循环的微任务之后，
    // 作用就是让你可以一次性多次执行限流函数入列，会等待你全部同步入列后才会执行下面的出列操作
     await Promise.resolve();

    // 出列：判断正在执行的数量没有超出限制就出列run函数并执行
     if (activeCount < concurrency && queue.size > 0) {
       queue.dequeue()();
     }
   })();
 };
```

### run

在被出队后的异步函数真正执行的是`run`函数。它在内部控制了计数器的个数，执行`resolve`函数以及执行下一个异步任务。

```js
const run = async (fn, resolve, args) => {
  // 标记正在执行的数量加1
  activeCount++;

  // 执行fn
  // 利用async包裹执行，让函数报错了只会报Uncaught (in promise) Error，但不会影响其他代码执行
  const result = (async () => fn(...args))();

  // 将限流函数返回的promise状态改为已完成且结果是fn的执行结果
  // 这里就是使用时，limit包裹一层依然能得到相同的promise结果的关键
  resolve(result);

  // 捕捉上面所说的可能发生会报的Uncaught (in promise) Error，让执行彻底不会抛出异常
  // await让result执行完后，再执行next函数
  try {
    await result;
  } catch {}

  // 下一小节介绍
  next();
};
```

### next

`next`函数主要处理此次异步函数执行完毕后的后续操作，包括对计数器`减一`，取出后续的异步任务。

```js
 const next = () => {
   // 当前执行任务减一
   activeCount--;
   // 如果队列中还有任务，那么继续取出执行
   if (queue.size > 0) {
     queue.dequeue()();
   }
 };
```

通过函数 `enqueue`、`run` 和 `next`，`plimit` 就产生了一个限制大小但不断消耗的异步函数队列，从而起到限流的作用。

### 设置外部可调用属性和方法

`p-limit`中还定义了一些辅助函数，比如：`activeCount`(获取当前执行任务的个数)、`pendingCount`(当前等待任务个数，队列中的个数)、`clearQueue`(清空队列)
```js
 Object.defineProperties(generator, {
   activeCount: {
     get: () => activeCount,
   },
   pendingCount: {
     get: () => queue.size,
   },
   clearQueue: {
     value: () => {
       queue.clear();
     },
   },
 });
```

## 总结

`p-limit`异步操作的并发控制原理就是通过队列控制多个异步操作并发，入列的时机是在限流函数的执行，出列的时机则有两种可能，优先限流函数的入列后就执行，但当达到限流条件时就会在每次执行异步操作后也会释放一次出列的机会。

回到代码层面，`p-limit`虽然实现代码只有68行，但内部有一些技巧很值得我们学习：

- `promise`的`resolve`传入内部和闭包的结合来接管外部函数的执行
- 闭包设置内部私有内部属性，`Object.defineProperties`设置外部可访问属性
- 前置一个微任务实现异步操作的并发控制


## 参考文章

- [Node.js 并发能力总结](https://mp.weixin.qq.com/s/6LsPMIHdIOw3KO6F2sgRXg)
- [关于请求并发控制的思考](https://juejin.cn/post/7045274658798567454#comment)
