## 前言
在我们学习js过程中，用户点击了某个按钮，然后网页进行某个交互，那么这是如何做到的呢？在js中提供了`addEventListener`和`removeEventListener`两个API来监听和移除事件，用户点击某个节点后，该DOM元素产生了一个click事件并派发出去，`addEventListener`监听到事件后执行某个操作。那么我们该如何实现对应的事件注册、派发、移除等功能呢？接下来我们来看看 `mitt` 和 `tiny-emitter` 这两个库吧。

## mitt 分析
- 体积小，200b
- 简单，基于单个函数的事件发射器或发布订阅。
  
[mitt 详细使用](https://github.com/developit/mitt)


```js
import mitt from 'mitt'

const emitter = mitt()

// listen to an event
emitter.on('foo', e => console.log('foo', e) )

// listen to all events
emitter.on('*', (type, e) => console.log(type, e) )

// fire an event
emitter.emit('foo', { a: 'b' })

// clearing all events
emitter.all.clear()

// working with handler references:
function onFoo() {}
emitter.on('foo', onFoo)   // listen
emitter.off('foo', onFoo)  // unlisten
```
在 mitt 这个库中，整体分析起来比较简单，就是 导出了一个 `mitt([all])`函数，调用该函数返回一个 `emitter`对象，该对象包含`all、on(type, handler)`、`off(type, [handler])`和`emit(type, [evt])`这几个属性。

主要实现就是：

1. `all = all || new Map()` mitt 支持传入 all 参数用来存储事件类型和事件处理函数的映射Map，如果不传，就 `new Map()`赋值给 all
2. `on(type, handler)`定义函数 on来注册事件，以type为属性，`[handler]`为属性值，存储在 all 中，属性值为数组的原因是可能存在监听一个事件，多个处理程序
3. `off(type, [handler])`来取消某个事件的某个处理函数，根据 type 找到对应的事件处理数组，对比 handler 是否相等，相等则删除该处理函数，不传则删除该事件的全部处理函数
4. `emit(type, [evt])`来派发事件，根据 type 找到对应的事件处理数组并依次执行，传入参数 evt(对象最好，传多个参数只会取到第一个)

具体的可参考[源码实现](https://github.com/developit/mitt/blob/main/src/index.ts)：
```js
// 导出函数 mitt ，调用该函数返回一个 对象，该对象包含 all、on、off、emit 方法
export default function mitt<Events extends Record<EventType, unknown>>(
	all?: EventHandlerMap<Events>
): Emitter<Events> {
	type GenericEventHandler =
		| Handler<Events[keyof Events]>
		| WildcardHandler<Events>;
	all = all || new Map();

	return {

		all,

		on<Key extends keyof Events>(type: Key, handler: GenericEventHandler) {
			// 获取 type 对应的的 事件处理函数数组
			const handlers: Array<GenericEventHandler> | undefined = all!.get(type);
			// 存在，说明设置过，handlers 已经是数组，直接 push 进去
			if (handlers) {
				handlers.push(handler);
			}
			else {
				// 不存在，说明这是第一次 注册该事件，那么 设置 type 的属性值为 [handler]
				all!.set(type, [handler] as EventHandlerList<Events[keyof Events]>); // !.是TS断言，意思是all中必有set这个东西
			}
		},

		off<Key extends keyof Events>(type: Key, handler?: GenericEventHandler) {
			// 找到 type 事件的 事件处理函数数组
			const handlers: Array<GenericEventHandler> | undefined = all!.get(type);
			if (handlers) {
				if (handler) {
          // 1. >>> 运算符执行无符号右移位运算。
          // 2. 它把无符号的 32 位整数所有数位整体右移。
          // 3. 对于无符号数或正数右移运算，无符号右移与有符号右移运算的结果是相同的
          // 4. 对于负数来说，无符号右移将使用 0 来填充所有的空位

          // 删除指定的 handler处理函数, 找到了 idx >>> 0 就是idx对应的索引，
					// indexOf 如果找不到对应的下标，是会返回一个 -1 的，没找到 -1 变为 4294967295，原数组不会改变
					handlers.splice(handlers.indexOf(handler) >>> 0, 1);
				}
				else {
					// 说明删除全部
					all!.set(type, []);
				}
			}
		},

		emit<Key extends keyof Events>(type: Key, evt?: Events[Key]) {
			// 找到 type 事件的 事件处理函数数组
			let handlers = all!.get(type);
			if (handlers) {
				// 依次执行每个函数，并传入相应的参数
				(handlers as EventHandlerList<Events[keyof Events]>)
					.slice() // 生成新数组，使得后面的map不会改变原数组
					.map((handler) => {
						handler(evt!);
					});
			}

			// 获取监听全部事件的处理函数，type事件为全部事件~
			handlers = all!.get('*');
			if (handlers) {
				// 依次执行每个函数，并传入相应的参数
				(handlers as WildCardEventHandlerList<Events>)
					.slice()
					.map((handler) => {
						handler(type, evt!);
					});
			}
		}
	};
}
```

缺点：

1. mitt 支持传入 all 参数，如果 all = {} 而不是 new Map() 那么会报错，不能正常使用，在ts中当然会提醒你，但是如果在js中使用这个库就没有提示，运行时会报错（当然可能是我用法错了，我直接在html文件中引用）
2. `emit(type, [evt])`中只能接受一个参数，要是传多个参数需要将多个参数合并成对象传入，然后在事件处理函数中解构

## tiny-emitter 分析

- 体积小，<1k
- 简单，基于构造函数和原型的事件发射器。

[tiny-emitter 详细使用](https://github.com/scottcorgan/tiny-emitter)

在 tiny-emitter 这个库中，定义了函数`E`，修改`E.prototype`，在原型对象中引入了`on(name, callback, ctx)`、`once(name, callback, ctx)`、`emit(name)`、`off(name, callback)`四个方法，整体功能和 mitt 大同小异，直接看[源代码](https://github.com/scottcorgan/tiny-emitter/blob/master/index.js)：
```js
function E () {
  // 使其为空让其很容易继承
}

// 在原型上修改
E.prototype = {
  // 在原型对象上建立方法
  on: function (name, callback, ctx) {
    // 获取事件处理函数和事件类型的映射对象，第一次不存在则赋值为空对象
    var e = this.e || (this.e = {});

    // 经典简洁的有值取值，无值初始化
    (e[name] || (e[name] = [])).push({
      fn: callback,
      ctx: ctx
    });

    return this; // return this 支持链式调用
  },

  once: function (name, callback, ctx) { // 注册单次触发事件
    var self = this; // 防止里面的函数this丢失
    function listener () {
      self.off(name, listener); // 先对订阅函数进行删除
      callback.apply(ctx, arguments); // 执行处理函数
    };
    // 如果想在事件触发前移除注册的事件怎么办呢？
    // 这里用listener._ = callback将原来的handler挂在listener的_属性上。
    // off方法中保留符合条件handler的逻辑如下：
    // 通过once注册的handler会因为满足evts[i].fn._ === callback 而被移除。
    listener._ = callback
    return this.on(name, listener, ctx);
  },

  emit: function (name) {
    // 获取传入的参数，可以多个
    var data = [].slice.call(arguments, 1);
    // 获取 name 类型的事件处理函数数组
    var evtArr = ((this.e || (this.e = {}))[name] || []).slice();
    var i = 0;
    var len = evtArr.length;

    // 遍历执行事件处理函数
    for (i; i < len; i++) {
      // evtArr[i] ==>  { fn: callback, ctx: ctx }, data 是传入的参数
      evtArr[i].fn.apply(evtArr[i].ctx, data);
    }

    return this;
  },

  off: function (name, callback) {
    var e = this.e || (this.e = {});
    var evts = e[name]; // 事件处理函数数组
    var liveEvents = []; // 注销某事件的一个处理函数后的剩余函数数组

    if (evts && callback) {
      for (var i = 0, len = evts.length; i < len; i++) {
        // 过滤掉与传入的回调函数相同的函数
        // evts[i].fn._ 对应 once 的时候，此时 evts[i].fn 是 listener 函数， evts[i].fn._  === callback
        if (evts[i].fn !== callback && evts[i].fn._ !== callback)
          // 该方法用数组liveEvents暂存未被移除的handler。
          liveEvents.push(evts[i]);
      }
    }

    // 注销事件后，还存在其他事件就进行赋值，否则删除
    (liveEvents.length)
      ? e[name] = liveEvents
      : delete e[name];

    return this;
  }
};

module.exports = E;
module.exports.TinyEmitter = E;
```

## mitt 和 tiny-emitter 对比分析

### 共同点
都支持`on(type, handler)`、`off(type, [handler])`和`emit(type, [evt])`三个方法来注册、注销、派发事件
### 不同点
- emit
  - 有 all 属性，可以拿到对应的事件类型和事件处理函数的映射对象，是一个Map不是{}
  - 支持监听'*'事件，可以调用emitter.all.clear()清除所有事件
  - 返回的是一个对象，对象存在上面的属性；基于函数，调用函数生成的每个emitter中各有一套map和on、off、emit方法。
- tiny-emitter
  - 支持链式调用, 通过e属性可以拿到所有事件（需要看代码才知道）
  - 多一个 `once` 方法 并且 支持设置 this(指定上下文 ctx)
  - 返回的一个函数实例，通过修改该函数原型对象来实现的；基于构造函数和原型，不同的emitter实例共享同一套on、once、off、emit方法。