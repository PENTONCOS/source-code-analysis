## omit是什么，有什么作用

根据官方的介绍这是用于创建一个已删除了某些字段后的浅拷贝的对象的函数。 也就是说这个函数是用于删除对象中的一个或多个字段，并返回一个新的对象。但这个新对象是原对象除去已剔除字段后浅拷贝过的对象。

- [omit.js](https://github.com/benjycui/omit.js)可以剔除对象中的属性
- 源码位置：[https://github.com/benjycui/omit.js/blob/master/src/index.js](https://github.com/benjycui/omit.js/blob/master/src/index.js)

## omit.js的源码分析

omit.js用于删除对象中某个或某几个属性，返回值为删除属性后的对象(且不会改变数据简单的源对象）

```js
function omit(obj, fields) {
  // eslint-disable-next-line prefer-object-spread
  // Object.assign--浅拷贝，第一层数据深拷贝，第二层嵌套数据为浅拷贝
  const shallowCopy = Object.assign({}, obj);
  for (let i = 0; i < fields.length; i += 1) {
    const key = fields[i];
    delete shallowCopy[key];
  }
  return shallowCopy;
}
```

> 提示：这个函数还是有些可以改进的地方。1、参数的校验。2、多类型支持，比如fields这个参数，可以支持数组也可以支持字符串。如果是字符串的话可以将对应字段删除

改进后的代码：
```js
function customTypeOf(obj, type) {
  return Object.prototype.toString.call(obj) === `[object ${type}]`;
}
function omitPro(obj, fields) {
  if(!customTypeOf(obj, 'Object')) { 
    return obj;
  }
  const shallowCopy = Object.assign({}, obj);
  if (customTypeOf(fields, 'String')) {
    delete shallowCopy[fields];
  }
  if (customTypeOf(fields, 'Array')) {
    for (let i = 0; i < fields.length; i += 1) {
      const key = fields[i];
      delete shallowCopy[key];
    }
  }
  return shallowCopy;
}
```

## 单元测试

作为一个工具函数，一定要使用单元测试来保证其质量，我们看到该库利用`assert`模块来进行执行单元测试。`assert`模块是`Node`的内置模块，主要用于断言。这里我们来学习下在[`omitjs`](https://github.com/benjycui/omit.js/blob/master/tests/index.test.js)中单元测试的使用方法。

```js
import assert from 'assert';
import omit from '../src';

describe('omit', () => {
  it('should create a shallow copy', () => {
    const benjy = { name: 'Benjy' };
    const copy = omit(benjy, []);
    assert.deepEqual(copy, benjy);
    assert.notEqual(copy, benjy);
  });

  it('should drop fields which are passed in', () => {
    const benjy = { name: 'Benjy', age: 18 };
    assert.deepEqual(omit(benjy, ['age']), { name: 'Benjy' });
    assert.deepEqual(omit(benjy, ['name', 'age']), {});
  });
});
```

1. 目录结构，单测都是写在`tests`文件下的。
2. `omit`的测试工具利用的是`father`提供的测试能力，`father`中的用来测试的工具是`umi-test`这个包，`umi-test`中最终使用的是`jest`单元测试工具。`jest`会默认会执行`tests`文件下的所有测试。
3. 这里我们看到`describe`，`it`这些编写测试用例的一些关键函数。
- `describe(string, fucntion)：`定义了我们所谓的测试套件，它是各个测试规范的集合
- `it(string, function)`函数定义了一个单独的测试规范，其中包含一个或多个测试期望
4. 断言api学习：描述应用程序中预期的行为片段。
- `assert.deepEqual():` 用来比较两个对象。只要他们的属性一一对应，且值多相等，就认为两个对象相等。
- `assert.notEqual():`只有在实际值等于预期值时，会抛出错误。

代码中第一个it测试用例，预期是返回一个新的浅拷贝的对象。第二个it测试用例，预期是应该删除传入的字段。

## father
在omit的脚本中，我们看到很多命令都使用了father这个工具，发现这个工具很强大，扩展了解下。

其是专门提供组件库或者工具库打包的集成工具，只需要更改配置文件就能轻松搭建一款自带说明文档的组件库。

其集成了docz的文档功能和基于rollup和babel的组件打包功能。

这里只是简单介绍下，详细能力可以直接看起官方文档。[github.com/umijs/fathe…](https://github.com/umijs/father)

## np
在omit的脚本设置中，我们看到这样一条命令。
```bash
"prepublishOnly": "npm run compile && np --yolo --no-publish",
```

这条命令中，我们发现其使用了`np`这个命令，扩展了解下。这个工具是帮助我们快速发布`npm`包，简化手动发包流程。简单说就是将我们正常的发包流程比如更新版本号，打`git tag`，发布包到`github`，发布包到`npm`等流程自动化操作了。
这里可以直接参考其官方介绍：[www.npmjs.com/package/np](https://www.npmjs.com/package/np)
## 收获

- 学习到`omit`作为一个开源库，不只是简单的写个函数。从开发、lint校验、单测、调试运行、文档站、打包发布等，其标准化的能力完全具备。
- 对`单元测试`的实际使用有了更深的了解
- 了解了`father`这个组件库开发集成工具
- 复习了`Object.assign`这个方法，并进一步去复习了深浅拷贝的实现。