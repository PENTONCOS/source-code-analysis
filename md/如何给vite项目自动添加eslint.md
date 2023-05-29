## 前言

为了提高代码质量，前端时间给组内同学做了代码规范说明，提供了一个vue3+vite的[项目模版](https://github.com/PENTONCOS/vue3-vite4-pinia-elementPlus-template)。

![](https://img-blog.csdnimg.cn/b649ff161a534482beb6b7bc01555c5f.png)

之前在接入 eslint 等工具的时候，每次都需要去旧项目里拷贝相关配置文件，同时去安装对应的插件。（种种原因，我们部门暂时没有维护统一的脚手架）。

我在想有没有什么高效的方法可以自动给新项目添加eslint，在社区找了一圈后，找到了[vite-pretty-lint](https://github.com/tzsk/vite-pretty-lint/)可以为vite项目一键添加eslint。

## 使用

进入项目，执行以下命令中的一个

```csharp
// NPM
npm init vite-pretty-lint

// YARN
yarn create vite-pretty-lint

// PNPM
pnpm create vite-pretty-lint
```

执行过程（项目类型我选择了vue，包管理工具我选择了npm)

![](https://img-blog.csdnimg.cn/24fbed05d92b43708b7d1873c859ca9a.png)


执行结果

![](https://img-blog.csdnimg.cn/98298bba3b2947c3af4143234308c1af.png)

## 源码

项目文件结构（以下结构仅包含了主要的实现代码 lib 目录）

```bash
├── ast.js
├── main.js
├── shared.js
├── templates
│   ├── react-ts.js
│   ├── react.js
│   ├── vue-ts.js
│   └── vue.js
└── utils.js
```



### 源码分析
#### 1. 整体流程
`lib/main.js` 包含了主要的实现代码，下面是主要实现的代码
```js
async function run() {
  console.log(
    chalk.bold(
      gradient.morning('\n🚀 Welcome to Eslint & Prettier Setup for Vite!\n')
    )
  );
  let projectType, packageManager;

  try {
    /**
     * 通过命令行交互的形式，获取用户的应用类型 projectType，包管理工具 packageManager
     * 应用类型主要提供了以下可选值： react、react-ts、vue、vue-ts
     * 包管理工具主要提供了以下可选值：yarn、npm、pnpm
     */
    const answers = await askForProjectType();
    projectType = answers.projectType;
    packageManager = answers.packageManager;
  } catch (error) {
    console.log(chalk.blue('\n👋 Goodbye!'));
    return;
  }
  /**
   * 通过选择的应用类型 projectType，同步读取对应的配置模版，
   * 获取需要依赖的包，以及对应的 eslint 配置信息
    */
    const { packages, eslintOverrides } = await import(

    `./templates/${projectType}.js`
  );

  // 获取所有依赖的包
  const packageList = [...commonPackages, ...packages];
  // 将从模版中获取的 eslint 配置信息覆盖到默认配置上
  const eslintConfigOverrides = [...eslintConfig.overrides, ...eslintOverrides];
  // 整理所需的 eslint 配置信息
  const eslint = { ...eslintConfig, overrides: eslintConfigOverrides };

  const commandMap = {
    npm: `npm install --save-dev ${packageList.join(' ')}`,
    yarn: `yarn add --dev ${packageList.join(' ')}`,
    pnpm: `pnpm install --save-dev ${packageList.join(' ')}`,
  };
  const viteJs = path.join(projectDirectory, 'vite.config.js');
  const viteTs = path.join(projectDirectory, 'vite.config.ts');
  const viteMap = {
    vue: viteJs,
    react: viteJs,
    'vue-ts': viteTs,
    'react-ts': viteTs,
  };

  // 获取 vite 配置文件的绝对路径
  const viteFile = viteMap[projectType];
  // 读取 vite 配置文件，并引入 eslint 配置信息
  const viteConfig = viteEslint(fs.readFileSync(viteFile, 'utf8'));
  // 根据选择的包管理工具，获取包安装指令
  const installCommand = commandMap[packageManager];

  if (!installCommand) {
    console.log(chalk.red('\n✖ Sorry, we only support npm、yarn and pnpm!'));
    return;
  }

  // 创建一个 spinner，用于显示进度
  const spinner = createSpinner('Installing packages...').start();
  // exec 函数：生成一个 shell，然后在该 shell 中执行“命令”，缓冲任何 生成的输出
  // 处理传递给 exec 函数的 command 字符串 直接由 shell 和特殊字符（因 shell 而异） 需要相应处理：
  // 执行安装依赖的 shell 命令
  exec(`${commandMap[packageManager]}`, { cwd: projectDirectory }, (error) => {
    if (error) {
      // 如果执行失败，则终止 spinner，并显示错误信息
      spinner.error({
        text: chalk.bold.red('Failed to install packages!'),
        mark: '✖',
      });
      console.error(error);
      return;
    }

    // 写入 eslint 配置文件
    fs.writeFileSync(eslintFile, JSON.stringify(eslint, null, 2));
    // 写入 prettier 配置文件
    fs.writeFileSync(prettierFile, JSON.stringify(prettierConfig, null, 2));
    // 写入 eslint 忽略文件
    fs.writeFileSync(eslintIgnoreFile, eslintIgnore.join('\n'));
    // 写入 vite 配置文件
    fs.writeFileSync(viteFile, viteConfig);
    
    // 执行成功，终止 spinner，并显示成功信息
    spinner.success({ text: chalk.bold.green('All done! 🎉'), mark: '✔' });
    console.log(
      chalk.bold.cyan('\n🔥 Reload your editor to activate the settings!')
    );
  });
}
```

### 2. shell 交互问答--askForProjectType 实现
`在lib/utils.js`中，下面是实现代码，可以看到，是通过 `enquirer` 库实现
```js
import enquirer from 'enquirer';
import fs from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';
// fileURLToPath：确保正确解码百分比编码的字符 以及确保跨平台有效的绝对路径字符串。
const __dirname = path.dirname(fileURLToPath(import.meta.url));

export function getOptions() {
  const OPTIONS = [];
  // 读取模版文件夹下的所有模版文件
  fs.readdirSync(path.join(__dirname, 'templates')).forEach((template) => {
    // 获取模版文件的名称，并去掉后缀
    const { name } = path.parse(path.join(__dirname, 'templates', template));
    // 将模版文件的名称添加到选项列表中
    OPTIONS.push(name);
  });
  return OPTIONS;
}

export function askForProjectType() {
  return enquirer.prompt([
    {
      // 选择 shell 交互类型 
      type: 'select',
      // 获取通过此参数获取对应选择结果
      name: 'projectType',
      // 本次操作的标题描述
      message: 'What type of project do you have?',
      // 可选择的选项
      choices: getOptions(),
    },
    {
      type: 'select',
      name: 'packageManager',
      message: 'What package manager do you use?',
      choices: ['npm', 'yarn', 'pnpm'],
    },
  ]);
}
```

### 3. viteEslint，通过babel修改原有的 vite 配置文件

- 读取 `vite.config.js` 文件的代码
- 通过 **babel** 转化成抽象语法书
- 在 **AST** 中查找需要插入的位置，并插入相关内容
- 再通过 **babel** 把 **AST** 转成普通 **JS** 代码

```js
export function viteEslint(code) {
  // 将传入的代码转换为 AST
  const ast = babel.parseSync(code, {
    // 指示代码应该被解析的模式，可以是 'script'、'module' 或 'unambiguous'。
    sourceType: 'module',
    // 是否在生成的 AST 中输出注释
    comments: false,
  });
  // 取出主题程序部分的 AST
  const { program } = ast;

  // 取出引入依赖（import）的 AST
  const importList = program.body
    .filter((body) => {
      return body.type === 'ImportDeclaration';
    })
    .map((body) => {
      // 删除注释的 AST
      delete body.trailingComments;
      return body;
    });

  // 查询是否引入了 vite-plugin-eslint，若已经引入了，就直接返回传人的代码 code
  if (importList.find((body) => body.source.value === 'vite-plugin-eslint')) {
    return code;
  }

  // 取出非 import 部分的代码的 AST
  const nonImportList = program.body.filter((body) => {
    return body.type !== 'ImportDeclaration';
  });
  // 取出 「export default」声明的代码 AST
  const exportStatement = program.body.find(
    (body) => body.type === 'ExportDefaultDeclaration'
  );

  // 判断当前声明的类型是否为 函数调用表达式
  if (exportStatement.declaration.type === 'CallExpression') {
    // 取出函数调用表达式的入参
    const [argument] = exportStatement.declaration.arguments;
    // 判断入参的类型是否为对象表达式
    if (argument.type === 'ObjectExpression') {
      // 取出对象表达式的 plugins 属性
      const plugin = argument.properties.find(
        ({ key }) => key.name === 'plugins'
      );

      if (plugin) {
        // 把 vite-plugin-eslint 插件加入到 plugins 属性中
        plugin.value.elements.push(eslintPluginCall);
      }
    }
  }

  importList.push(eslintImport);
  importList.push(blankLine);
  program.body = importList.concat(nonImportList);

  ast.program = program;

  // 将 AST 转换为代码
  return babel.transformFromAstSync(ast, code, { sourceType: 'module' }).code;
}
```

### 4. 彩色渐变的 log 输出

- [chalk](https://github.com/chalk/chalk)，给你的终端输出内容加上样式
- [gradient](https://github.com/bokub/gradient-string)，在终端输出漂亮的颜色渐变

### 5. 加载进度提示

[nanospinner](https://github.com/usmanyunusov/nanospinner)，最简单和最小的终端旋转器

## 展望

后续有时间打算定制一个属于自己团队的`lint-pretty`插件，大概思路如下：

1. Fock `vite-pretty-lint` 项目
2. 修改或替换 `./lib/templates` 下的模版文件
3. 因为 `vite-pretty-lint` 是基于 `vite` 项目的，如果是`非 vite` 项目，就需要调整 `./lib/main.js` 文件中对 `vite.config.js` 文件进行修改的部分代码。