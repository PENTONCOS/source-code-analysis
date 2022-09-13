## 链表
链表是一种数据结构，链表中的每个节点至少包含两个部分：`数据域`和`指针域`。其中数据域用来存储数据，而指针域用来存储指向下一个数据的地址，如下图所示：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7cf902512d444ca99dd878c9f5bc5dcb~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

- 链表中的每个节点至少包含两个部分：`数据域`和`指针域`
- 链表中的每个节点，通过`指针域`的值，形成一个线性结构
- 查找节点`O(n)`，插入节点`O(1)`，删除节点`O(1)`
- 不适合 快速的定位数据，通过动态的插入或删除数据 的场景

模拟实现一个简单的链表：

1. 首先定义一个`Node`类
```js
 class Node{
     constructor(val) {
         // 数据域
         this.val = val
         // 指针域
         this.next = null
     }
 }
```

2. 接下来实现添加和打印功能：
```js
 class LinkNodeList{
     constructor() {
         this.head = null
         this.length = 0
     }
     // 添加节点
     append(val) {
         // 使用数据创建一个节点node
         let node = new Node(val)
         let p = this.head
         if(this.head) {
             // 找到链表的最后一个节点，把这个节点的.next属性赋值为node
             while(p.next) {
                 p = p.next
             }
             p.next = node
         } else {
             // 如果没有head节点，则代表链表为空，直接将node设置为头节点
             this.head = node
         }
         this.length++
     }
     // 打印链表
     print() {
         if(this.head) {
             let p = this.head
             let ret = ''
 ​
             do{
                 ret += `${p.val} --> `
                 p = p.next
             }while(p.next)
             ret += p.val
         } else {
             console.log('empty')
         }
     }
 }
 ​
 let linkList = new LinkNodeList()
 ​
 linkList.append(1)
 linkList.append(2)
 linkList.append(3)
 linkList.append(4)
 
 linkList.print() // 1 --> 2 --> 3 --> 4
 
 console.log(linkList.length) // 4
```
## 队列

队列是一种`先进先出`的数据结构。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f111f7b94a846c2a71934defa53f1f8~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

也就是只允许从队尾插入元素，队头弹出元素，如图：

出队操作

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ead95c261014e87aa5acbbc09655d74~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

入队操作

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72a0577def464031a97a5f6be3687978~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

其实对应到js中数组的操作是这样的：

只涉及数组中的两种操作`push`（后增），`shift`（前删）
```js
 // 伪代码
 class Queue {
    constructor() {
   this.queue = []
    },
    push(val) {
   // 使用push进行入队操作
     this.queue.push(val)
    },
    shift() {
   // 使用shift进行出队操作
         this.queue.shift()
    }
 }
```

## yocto-queue

[yocto-queue](https://github.com/sindresorhus/yocto-queue)其实是实现了一个微小的队列结构，它的目的实际上是为了让用户在操作数据量较大的数组时提高程序执行的效率。


### 数组

数组在内存中是一种按顺序进行存储的数据结构

例如我们有一个数组ary:
```js
 let ary = ['吃饭', '睡觉', '吃黄瓜']
```
它在可用内存中是这样存储的：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3d051f8ea1b4188b1b042527ec92cf4~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

可以看到数组中的每一项都是按照在内存中的顺序进行存储的，这样的数据结构可以很便捷的访问数组中的每一项，可以直接通过下标进行访问：
```js
 // 取数组中的第二项
 ary[1]  // 睡觉
 ​
 // 因为是顺序存储，所以下标的位置就是数组中值的所在的位置
```
但是这种数据结构也由此产生了一个非常麻烦的问题，如果我们删除数据的第一项（数组中的`shift`操作）或者在任意一个位置插入元素,那么其他后面所有的位置全部都要向前移动一位！
因为数组是顺序存储的，如果不进行移动，那么我们在使用`ary[0]`访问第一个元素的时候，打印的会是`undefined`
```js
 // 伪代码
 // 假设数组不向前移动
 ary.shift() // [undefined, '睡觉', '吃黄瓜']
 ary[0] // undefined
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b4af434744948c4b74eccc88bd45663~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

上图为数组移位操作

还有一个问题是，如上面的存储空间的示意图所示：如果我们的数组后面的空间被占用了，但是我们还想要再次增加一位，这时就需要将数组全部转移到内存的其他地方。

所以对数组的有些操作对性能的损耗将是非常大的。

### 链表

例如我们有一个链表`queue`:
```js
 let queue = node('吃饭') -> node('睡觉') -> node('吃黄瓜')
```

由于链表的数据结构中存在`指针域`的存在，用于存储下一个元素的位置的坐标，所以数组中所产生的两个问题就不会存在了，链表中的任意一个值可以存储在内存中的任意一个地方，如下图：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee8b84919bc0465990c782be5fc17ee9~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

所以我们进行链表的删除或者插入操作时，只需要将上一个节点的`next`指针指向新插入元素，将新插入的元素的`next`指针指向下一个元素即可，**没有被`next`指针所连接的元素会自动被回收**，而我们上文中提到删除第一个元素就更简单了，只需要我们将`head`头节点变更为第二个元素即可。

但是链表这种结构也并不是完美无缺的，因为链表是随机存储的，每个值的位置都不是固定的，所以我们就不可能通过下标的方式进行访问链表的元素了，只能通过一个一个遍历的方式进行查找：
```js
 // 伪代码
 ​
 while(this.head) {
  this.head = this.head.next
 }
```

`yocto-queue`实现了队列的几种基本操作：`enqueue`(入队)、`dequeue`(出队)、`clear`(清除)、`size`(获取长度)、`[Symbol.iterator]`(处理for...of遍历)

接下来我们就从yocto-queue的测试文件入手，来一个一个的实现每个功能的逻辑：

### node节点类

由于链表中每一个元素都是一个节点，所以，我们先创建一个节点类：
```js
 class Node {
   // 定义节点值->value (用于保存节点真正的值)
   value;
   // 定义指针域->next (用于保存指向下一个节点的指针地址)
   next;
   constructor(value) {
     // 初始化节点值
     this.value = value
   }
 }
```

这个地方有一个需要注意的点，可以看到我们的初始化的值是定义在`constructor`外面的，也就是说，定义变量初始值有两种写法：

**例如：**

```js
 // 第一种写法
 class Person {
   constructor() {
     this._count = 0;
   }
 }
```
```js
 class Person {
   _count = 0;
   constructor() {
   }
 }
```
上面代码中，第二种写法的实例属性`_count`，放在了`constructor`上面，与`constructor`处于同一个层级。这时，不需要在实例属性前面加上`this`。

注意，第二种写法定义的属性是实例对象自身的属性，而不是定义在实例对象的原型上面。

而`yocto-queue`中定义`Node`类的初始化值的写法正是采用了第二种写法。

### Queue 类
```js
 class Queue {
     // 长度
     #size;
     // 头节点
     #head;
     // 尾节点
     #tail;
     constructor() {
       // 初始化类时，手动初始化一次各个属性值
       this.clear()
     }
 }
```

`Queue`类中定义了三个属性，`this.#size`用于保存队列的长度，`this.#head`用于保存头节点，返回整个队列只需要返回头节点即可，因为各个节点都是通过`next`属性串联起来的，换言之，获得了头节点，就可以获得整个通过链表生成的整个队列所有的值。

那么为什么还要实现 `this.#tail` 用于保存尾节点呢？

其实在`yocto-queue`源码中已经做了回答：

> How it works: this.#head is an instance of Node which keeps track of its current value and nests another instance of Node that keeps the value that comes after it. When a value is provided to .enqueue(), the code needs to iterate through this.#head, going deeper and deeper to find the last value. However, iterating through every single item is slow. This problem is solved by saving a reference to the last value as this.#tail so that it can reference it to add a new value.

> 它是如何工作的： this.#head是一个Node的实例，它记录了它当前的值，并嵌套了另一个Node的实例，它记录了它后面的值。 当一个值被提供给.enqueue()时，代码需要遍历this.#head，不断深入以找到最后的值。然而，遍历每一个单项是很慢的。这个问题的解决方法是将最后一个值的引用保存为this.#tail，这样它就可以引用它来添加新的值。

所以这就可以解释`this.#tail`的作用：用于保存尾节点便于更快的添加新节点，当我们添加新节点时，只需要将尾节点（也就是`this.#tail`）的`next`属性定义为新节点即可。
### enqueue(入队)

入队操作

我们先来分析一下`enqueue`方法的单测文件，看一下都实现了那些功能：
```js
test('.enqueue()', t => {
    // 创建一个queue结构
    const queue = new Queue();
    // 入队一个彩虹小马
    queue.enqueue('🦄');
    // dequeue为出队操作（下文会讲解），出队后是否等于彩虹小马
    t.is(queue.dequeue(), '🦄');
  
    // 连续入队彩虹和黑桃心
    queue.enqueue('🌈');
    queue.enqueue('❤️');
    // 此时链表中为🌈 ❤️
    t.is(queue.dequeue(), '🌈');  
    // 是否为头部出队
    t.is(queue.dequeue(), '❤️');
});
```
接下来就可以实现`enqueue`函数功能了，可以看到核心就是两点：

- 根据值创建节点
- 从尾部入队

```js
enqueue(value) {
    // 创建节点
    const node = new Node(value);
    // 首先判断头节点是否存在
    if (this.#head) {
        // 存在的话，直接将尾节点的next属性设置为新节点
	this.#tail.next = node;
        // 将尾节点移动最后一个位置
	this.#tail = node;
    } else {   
        // 如果头节点不存在（整个队列为空，那么直接将头节点设置为新节点）
	this.#head = node;
        // 此时队列中只存在一个节点，头节点和尾节点都指向新元素
	this.#tail = node;
    }
    // 更新长度值
    this.#size++;
}
```

### dequeue(出队)

还是先来看一下单测文件：
```js
test('.dequeue()', t => {
    const queue = new Queue();
     // 如果队列中并没有值，那么返回undefined
    t.is(queue.dequeue(), undefined);
    t.is(queue.dequeue(), undefined);
    queue.enqueue('🦄');
    // 从头部开始弹出
    t.is(queue.dequeue(), '🦄');
    t.is(queue.dequeue(), undefined);
});
```
其实出队操作也只有两点：

- 如果队列为空，那么直接返回`undefined`
- 从头部开始弹出队列

```js
dequeue() {
    // 保存头节点，防止丢失
    const current = this.#head;
    // 如果头节点没有值，那么直接返回
    if (!current) {
        return;
    }
    // 将头节点移动到第二个元素
    this.#head = this.#head.next;
    // 更新长度值
    this.#size--;
    // 返回已删除头节点
    return current.value;
}
```

### clear(清除)

清除队列的操作

清除队列的操作很简单，只需要将`head`和`tail`的指针都变为`undefined`，清空长度值即可

```js
clear() {
    his.#head = undefined;
    this.#tail = undefined;
    this.#size = 0;
}
```
### size(返回长度)
返回队列长度的操作也很简单，只需要将#size属性返回即可
```js
get size() {
    return this.#size
}
```
### [Symbol.iterator]

`Symbol.iterator`属性为可遍历数据的遍历器函数，只要是在内部部署了`[Symbol.iterator]`函数，那么就可以被`for...of`所遍历。`for...of`中的实现内部就会自动去调用`[Symbol.iterator]`函数的`next`方法，所以`for...of`在执行的时候不断地执行`next`进行调用，遇到`Generator`函数中的`yield`关键字会暂停，并返回值，然后再调用`next`方法从中断的位置继续执行...直到遍历完成。
```js
// 生成器函数
* [Symbol.iterator]() {
    // 获取头节点
     let current = this.#head;

     while (current) {
         // 暂停
         yield current.value;
         current = current.next;
     }
}
```

## 总结
从`yocto-queue`源码中，可以学习队列实现方式以及对应API的单元测试的完整过程；了解到链表与数组的一些区别以及`Symbol.iterator`的使用场景。