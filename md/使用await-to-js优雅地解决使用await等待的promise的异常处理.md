## 背景
话不多说，看代码:
```js
function getData(data) {
  return new Promise((resolve, reject) => {
    if (data === 1) {
      setTimeout(() => {
        resolve("getdata success");
      }, 1000);
    } else {
      setTimeout(() => {
        reject("getdata error");
      }, 1000);
    }
  });
}

try {
  let res = await getData(3);
  console.log(res);
} catch (error) {
  console.log(res);
}
```
我们可以看到，针对`Promise`的错误捕获是使用`try + catch`的，该结构其实已经很清晰明了了，但是有没有发现，我们为了捕获错误多了很多行代码？
```js
//不捕获:
let res = await getData(3);
console.log(res);

//捕获:
try {
  let res = await getData(3);
  console.log(res);
} catch (error) {
  console.log(res);
}
```

那如果有很多个`Promise`需要捕获异常这样就需要写`n`个`try + catch `。这样写会显得很繁琐。那么就可以使用这个库来解决这个问题：`await-to-js`。

## 学习内容和准备工作

### 学习内容：
github: [await-to-js](https://github.com/scopsy/await-to-js)

官方博客：[how-to-write-async-await-without-try-catch-blocks-in-javascript](https://blog.grossman.io/how-to-write-async-await-without-try-catch-blocks-in-javascript/)

### 准备工作：
```
1.clone代码  git clone  https://github.com/scopsy/await-to-js
2.安装依赖 npm  i
3.打包 npm run build (源码是用ts写的，打包后有js版本)
```
## 学习过程
先看一下是什么，怎么用：
```js
import to from 'await-to-js';

async function asyncFunctionWithThrow() {
  const [err, user] = await to(UserModel.findById(1));
  if (!user) throw new Error('User not found');
}
```
可以看到使用`to`装饰之后的调用，还是用`await`发请求，但是直接就可以解构错误信息，而不用通过`.catch`的方式，是不是很方便呢？看一下源码的实现吧~
### 源码解读
ts代码的位置： [src\await-to-js.ts](https://github.com/scopsy/await-to-js/blob/master/src/await-to-js.ts)
```js
/**
 * @param { Promise } promise
 * @param { Object= } errorExt - Additional Information you can pass to the err object
 * @return { Promise }
 */
export function to<T, U = Error> (
  promise: Promise<T>,
  errorExt?: object
): Promise<[U, undefined] | [null, T]> {
  return promise
    .then<[null, T]>((data: T) => [null, data])
    .catch<[U, undefined]>((err: U) => {
      if (errorExt) {
        const parsedError = Object.assign({}, err, errorExt);
        return [parsedError, undefined];
      }

      return [err, undefined];
    });
}

export default to;
```

打包后js代码位置：
> dist\await-to-js.es5.js
```js
/**
 * @param { Promise } promise
 * @param { Object= } errorExt - Additional Information you can pass to the err object
 * @return { Promise }
 */
function to(promise, errorExt) {
    return promise
        .then(function (data) { return [null, data]; })
        .catch(function (err) {
        if (errorExt) {
            var parsedError = Object.assign({}, err, errorExt);
            return [parsedError, undefined];
        }
        return [err, undefined];
    });
}

export { to };
export default to;
//# sourceMappingURL=await-to-js.es5.js.map
```
参数为`promise`和可选错误信息对象；`then`中返回`[null，data]` ，`null`表示没有发错错误（异常），`data`为实际的数据；`catch`中判断是否提供了额外的错误对象，如果有则合并`promise`返回的异常和错误对象，然后返回`[parsedError, undefined]`表示发生了错误没有数据。如果没有提供额外的错误对象，则返回`promise`返回的异常和`undefined`，即`[err, undefined]`。最后导出一个对象以及默认导出，为了支持不同的导入方式。
### 测试
```js
function to(promise, errorExt) {
  return promise
    .then(function (data) { return [null, data]; })
    .catch(function (err) {
    if (errorExt) {
      var parsedError = Object.assign({}, err, errorExt);
      return [parsedError, undefined];
    }
    return [err, undefined];
  });
}
async function func(){
	const testInput = 41;
	const promise = Promise.resolve(testInput);
	const [err, data] = await to(promise);
	console.log(err, data)
	// null 41
}
func()

async function func2(){
	const promise = Promise.reject('Error');
	const [err, data] = await to(promise);
	console.log(err, data)
	// Error undefined
}
func2()

async function func3(){
	const promise = Promise.reject({ error: 'Error message' });
	const [err, data] = await to(promise, {extraKey: 1});
	console.log(err, data)
	// {error: 'Error message', extraKey: 1}  undefined
}
func3()


async function func4(){
	const promise = Promise.resolve({ name: '123' });
	const [err, data] = await to(promise);
	console.log(err, data)
	// null {name: '123'}
}
func4()
```
## 一个简版的实现
此简版的实现摘自官方的[介绍文档](https://github.com/scopsy/await-to-js#typescript-usage)：
```js
export default function to(promise) {
  return promise.then(data => {
    return [null, data];
  })
  .catch(err => [err]);
}
```
## 思考
看到这个库的用法我第一反应是想到了react钩子函数：
```js
const [count, setCount] = useState(0);
```
还想到了node.js中的回调函数的传参形式也是 `err,data`：
```js
var fs = require("fs");

fs.readFile('input.txt', function (err, data) {
    if (err) return console.error(err);
    console.log(data.toString());
});

console.log("程序执行结束!");
```