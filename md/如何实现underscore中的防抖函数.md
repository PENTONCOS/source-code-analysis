## 什么是防抖

在事件触发后n秒后执行回调函数，如果n秒内再次触发，则重新计算时间

## 防抖的使用场景

**鼠标事件：** click，mousemove，mouseover

**键盘事件：** keydown，keyup

**window事件:** resize，scroll

以上事件频繁触发就可能引起多次请求或页面卡顿，为了规避这种情况，就需要控制回调函数的执行次数。常见的有防抖和节流，本次介绍underscore的防抖函数。


## 使用案例

```html
<div id="container"></div>
```

```js
//<------ js代码 ------>
var count = 1;
var container = document.getElementById('container');

function getUserAction() {
  container.innerHTML = count++;
};

container.onmousemove = debounce(getUserAction, 1000);
```

## 代码实现
### 第一版

```js
function debounce(func, wait) {
  var timeout;
  return function () {
    clearTimeout(timeout)
    timeout = setTimeout(func, wait);
  }
}
```

### this
如果我们在 `getUserAction` 函数中 `console.log(this)`，在不使用 `debounce` 函数的时候，`this` 的值为：

```html
<div id="container"></div>
```

但是如果使用第一版的 `debounce` 函数，`this` 就会指向 `Window` 对象！
所以需要将 `this` 指向正确的对象。

修改下代码：
```js
function debounce(func, wait) {
  var timeout;

  return function () {
    var context = this;

    clearTimeout(timeout)
    timeout = setTimeout(function(){
      func.apply(context)
    }, wait);
  }
}
```

### 参数

JavaScript 在事件处理函数中会提供参数：事件对象 `event`，修改下 `getUserAction` 函数：
```js
function getUserAction(e) {
  console.log(e);
  container.innerHTML = count++;
};
```
如果我们不使用 `debouce` 函数，这里会打印 `MouseEvent` 对象。

但是在之前实现的 `debounce` 函数中，却只会打印 `undefined`!

所以再修改一下代码：
```js
function debounce(func, wait) {
  var timeout;

  return function () {
    var context = this;
    var args = arguments;

    clearTimeout(timeout)
    timeout = setTimeout(function () {
      func.apply(context, args)
    }, wait);
  }
}
```
### 返回值

再注意一个小点，`getUserAction` 函数可能是有返回值的，所以也要返回函数的执行结果
```js
function debounce(func, wait) {
  var timeout, result;

  return function () {
    var context = this;
    var args = arguments;

    clearTimeout(timeout)
    timeout = setTimeout(function () {
      result = func.apply(context, args)
    }, wait);

    return result;
  }
}
```

**注意：** 如果不设置函数立即执行的话，是不能得到返回值的，我测试了下 underscore ，确实是存在这个问题的。

### 立即执行

一个常见的需求：用户不希望非要等到事件停止触发后才执行，希望立刻执行函数，然后等到停止触发n秒后，才可以重新触发执行。

所以需要加个 `immediate` 参数判断是否是立刻执行。

```js
function debounce(func, wait, immediate) {

  var timeout, result;

  return function () {
    var context = this;
    var args = arguments;

    if (timeout) clearTimeout(timeout);
    if (immediate) {
      // 如果已经执行过，不再执行
      var callNow = !timeout;
      timeout = setTimeout(function () {
        timeout = null;
      }, wait)
      if (callNow) result = func.apply(context, args)
    }
    else {
      timeout = setTimeout(function () {
        result = func.apply(context, args)
      }, wait);
    }

    return result;
  }
}
```

### 取消

当时间间隔设置太长，立即执行后不想等，有个按钮可以取消`debounce`函数重新执行。

```js
// 第六版
function debounce(func, wait, immediate) {

  var timeout, result;

  var debounced = function () {
    var context = this;
    var args = arguments;

    if (timeout) clearTimeout(timeout);
    if (immediate) {
      // 如果已经执行过，不再执行
      var callNow = !timeout;
      timeout = setTimeout(function () {
        timeout = null;
      }, wait)
      if (callNow) result = func.apply(context, args)
    }
    else {
      timeout = setTimeout(function () {
        result = func.apply(context, args)
      }, wait);
    }
    return result;
  };

  debounced.cancel = function () {
    clearTimeout(timeout);
    timeout = null;
  };

  return debounced;
}
```

至此我们已经完整实现了一个 `underscore` 中的 `debounce` 函数。

## underscore的防抖源码

[源码地址](https://github.com/jashkenas/underscore/blob/master/modules/debounce.js)

```js
/**
 * 
 * @param {*} func 回调函数
 * @param {*} wait 等待时间
 * @param {*} immediate 是否立即执行回调函数
 * @returns 
 */
function debounce(func, wait, immediate) {
  var timeout, previous, args, result, context;

  var later = function() {
    //计算是否超时
    var passed = now() - previous;
    if (wait > passed) {
      //距离上一次调用已经过了passed时间
      timeout = setTimeout(later, wait - passed);
    } else {
      timeout = null;
      if (!immediate) result = func.apply(context, args);
      // This check is needed because `func` can recursively invoke `debounced`.
      if (!timeout) args = context = null;
    }
  };
//处理回调函数的参数
  var debounced = restArguments(function(_args) {
    context = this;
    args = _args;
    previous = now();
    if (!timeout) {
      //第一次的时候，timeout为null。如果immediate=true，立即执行回调函数
      timeout = setTimeout(later, wait);
      
      if (immediate) result = func.apply(context, args);
    }
    return result;
  });

//取消防抖的定时器
  debounced.cancel = function() {
    clearTimeout(timeout);
    timeout = args = context = null;
  };

  return debounced;
}


function restArguments(func, startIndex) {
  startIndex = startIndex == null ? func.length - 1 : +startIndex;
  return function() {
    var length = Math.max(arguments.length - startIndex, 0),
        rest = Array(length),
        index = 0;
    for (; index < length; index++) {
      rest[index] = arguments[index + startIndex];
    }
    switch (startIndex) {
      case 0: return func.call(this, rest);
      case 1: return func.call(this, arguments[0], rest);
      case 2: return func.call(this, arguments[0], arguments[1], rest);
    }
    var args = Array(startIndex + 1);
    for (index = 0; index < startIndex; index++) {
      args[index] = arguments[index];
    }
    args[startIndex] = rest;
    return func.apply(this, args);
  };
}
```