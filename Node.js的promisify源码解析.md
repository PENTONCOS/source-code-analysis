## 前言

我们经常会在本地git仓库切换tags，或者git仓库切换tags。那么我们是否想过如果获取tags呢。本文就从 `remote-git-tags` 这个22行代码的源码库入手。

## 源码解析
### 用处

从远程仓库获取所有标签。

原理：通过执行 `git ls-remote --tags repoUrl` （仓库路径）获取 `tags`


### 源码
```js
// index.js
import {promisify} from 'node:util';
import childProcess from 'node:child_process';

const execFile = promisify(childProcess.execFile);

export default async function remoteGitTags(repoUrl) {
	const {stdout} = await execFile('git', ['ls-remote', '--tags', repoUrl]);
	const tags = new Map();

	for (const line of stdout.trim().split('\n')) {
		const [hash, tagReference] = line.split('\t');

		// Strip off the indicator of dereferenced tags so we can override the
		// previous entry which points at the tag hash and not the commit hash
		// `refs/tags/v9.6.0^{}` → `v9.6.0`
		const tagName = tagReference.replace(/^refs\/tags\//, '').replace(/\^{}$/, '');

		tags.set(tagName, hash);
	}

	return tags;
}
```
[主文件](https://github.com/sindresorhus/remote-git-tags/blob/main/index.js)一共22行代码，非常容易理解。


### node:util


[Node文档](https://nodejs.org/dist/latest-v16.x/docs/api/modules.html#core-modules)

> Core modules can be identified using the node: prefix, in which case it bypasses the require cache. For instance, require('node:http') will always return the built in HTTP module, even if there is require.cache entry by that name.

也就是说：核心模块也可以使用`node:`前缀来标识，在这种情况下，它会绕过`require.cache`缓存。例如，`require('node:http')`将始终返回内建的`http`模块，即使有`require.cache`中存在以该名称的缓存条目。

其中有一段：

```js
const execFile = promisify(childProcess.execFile);
```
`child_process.execFile()`与[`child_process.exec()`](https://nodejs.org/docs/latest-v17.x/api/child_process.html#child_processexeccommand-options-callback)类似，区别是它默认不生成新的shell

`promisify`函数是把 `callback` 形式转成 `promise` 形式。`node`中的异步编程依托于回调实现。回调函数的第一个参数是错误信息。也就是错误优先。类似于：

```js
import {readFile} from 'fs'
readFile('input.txt', function (err, data) {
    if (err) return console.error(err);
    console.log(data.toString());
});

console.log("程序执行结束!");
```

接着说本文的重点： `promisify`。

## promisify

### 源码

```js
const kCustomPromisifiedSymbol = SymbolFor('nodejs.util.promisify.custom');
const kCustomPromisifyArgsSymbol = Symbol('customPromisifyArgs');

let validateFunction;

function promisify(original) {
  // Lazy-load to avoid a circular dependency.
  if (validateFunction === undefined)
    ({ validateFunction } = require('internal/validators'));

  validateFunction(original, 'original');

  if (original[kCustomPromisifiedSymbol]) {
    const fn = original[kCustomPromisifiedSymbol];

    validateFunction(fn, 'util.promisify.custom');

    return ObjectDefineProperty(fn, kCustomPromisifiedSymbol, {
      __proto__: null,
      value: fn, enumerable: false, writable: false, configurable: true
    });
  }

  const argumentNames = original[kCustomPromisifyArgsSymbol];
  // 核心实现 start
  function fn(...args) {
    return new Promise((resolve, reject) => {
      // ArrayPrototypePush = Array.prototype.push
      ArrayPrototypePush(args, (err, ...values) => {
        if (err) {
          return reject(err);
        }
        if (argumentNames !== undefined && values.length > 1) {
          const obj = {};
          for (let i = 0; i < argumentNames.length; i++)
            obj[argumentNames[i]] = values[i];
          resolve(obj);
        } else {
          resolve(values[0]);
        }
      });
      // ReflectApply = Reflect.apply
      ReflectApply(original, this, args);
    });
  }
  // 核心实现 end
  ObjectSetPrototypeOf(fn, ObjectGetPrototypeOf(original));

  ObjectDefineProperty(fn, kCustomPromisifiedSymbol, {
    __proto__: null,
    value: fn, enumerable: false, writable: false, configurable: true
  });
  return ObjectDefineProperties(
    fn,
    ObjectGetOwnPropertyDescriptors(original)
  );
}
```

### 简化实现

```js
function promisify(original) {
  return function(...args) {
    return new Promise((resolve, reject) => {
      args.push((err, value) => {
        if(err) {
          reject()
        } else {
          resolve(value)
        }
      })
      // 相当于original.apply(this, args)
      Reflect.apply(original, this, args)
    })
  }
}
```

**实现细节**

- 将原函数作为参数调用`promisify`返回一个包装后函数`fn`
- 调用`fn`并传入原函数需要的参数，在原参数后面追加一个回调函数
- 回调函数的第一个参数是错误信息`err`，第二个参数是`Promise`的返回值，根据是否有错误信息改变`Promise`的状态
- 执行原函数并传入处理后的参数`args`

使用的时候可以**在原函数的最后一个参数传入callback供异步调用**

```js
const loadImg = function(src, callback) {
  const img = document.createElement('img')
  img.src = src
  img.style = 'width: 200px;height: 280px';
  img.onload = callback(null, src) // 正常加载时，err传null
  img.onerror = callback(new Error('图片加载失败'))
  document.body.append(img);
}
const loadImgPromise = promisify(loadImg)
const load = async () => {
  try {
    const res = await loadImgPromise('https://avatars.githubusercontent.com/u/40193497?v=4')
    console.log(res)
  } catch(err) {
    console.log(err)
  }
}
load()
```







