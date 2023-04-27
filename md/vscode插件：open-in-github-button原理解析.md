最近 [antfu](https://github.com/antfu) 在 Twitter 上发推开发了一个 [VSCode 插件](https://github.com/antfu/vscode-open-in-github-button)。


# 1. 环境准备
```bash
# 克隆项目
git clone git@github.com:antfu/vscode-open-in-github-button.git
# npm i -g pnpm
cd vscode-open-in-github-button
pnpm install
# vscode-open-in-github 项目
cd vscode-open-in-github-button
pnpm install
```
# 2. vscode-open-in-github-button 项目
## 2.1 package.json scripts 命令解析
主要依赖依赖如下：

- [eslint 配置 @antfu/eslint-config](https://github.com/antfu/eslint-config)
- [使用正确的包 @antfu/ni](https://github.com/antfu/ni)
- [提升版本发布相关 bumpp](https://github.com/antfu/bumpp)
- [执行node ts 相关 esno](https://github.com/esbuild-kit/esno)
- [vscode 插件发布相关 vsce](https://github.com/microsoft/vscode-vsce)

**scripts** 分析
```json
{
    "scripts": {
        // 用 tsup 打包 vscode 插件扩展
        "build": "tsup src/index.ts --external vscode",
        // 开发
        "dev": "nr build --watch",
        "lint": "eslint .",
        // 预发布
        "vscode:prepublish": "nr build",
        // 发布，不包含依赖
        "publish": "vsce publish --no-dependencies",
        // 打包，不包含依赖
        "pack": "vsce package --no-dependencies",
        // 测试
        "test": "vitest",
        // type 检测
        "typecheck": "tsc --noEmit",
        // 发布
        "release": "bumpp && nr publish"
    }
}
```
## 2.2 github actions
看项目前，我们先来看下 github action 配置。[了解 GitHub Actions](https://docs.github.com/zh/actions/learn-github-actions/understanding-github-actions)

一个开源项目，一般会有基础的 workflow。

- ci 每次 git push 命令时自动执行 lint 和 test 等，保证校验通过。
- release：每次检测到 git tag，就自动发一个包。

### 2.2.1 ci
```yml
# vscode-open-in-github-button/.github/workflows/ci.yml
name: CI

# main 分支 push 和 pr 会触发 
on:
  push:
    branches:
      - main

  pull_request:
    branches:
      - main
#  安装 pnpm ，使用 node 16.x，全局安装 ni 工具，执行 nci npm ci npm run lint
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install pnpm
        uses: pnpm/action-setup@v2

      - name: Set node
        uses: actions/setup-node@v3
        with:
          node-version: 16.x
          cache: pnpm

      - name: Setup
        run: npm i -g @antfu/ni

      - name: Install
        run: nci

      - name: Lint
        run: nr lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install pnpm
        uses: pnpm/action-setup@v2

      - name: Set node
        uses: actions/setup-node@v3
        with:
          node-version: 16.x
          cache: pnpm

      - name: Setup
        run: npm i -g @antfu/ni

      - name: Install
        run: nci

      - name: Typecheck
        run: nr typecheck

  test:
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        node: [16.x]
        os: [ubuntu-latest, windows-latest, macos-latest]
      fail-fast: false

    steps:
      - uses: actions/checkout@v3

      - name: Install pnpm
        uses: pnpm/action-setup@v2

      - name: Set node version to ${{ matrix.node }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
          cache: pnpm

      - name: Setup
        run: npm i -g @antfu/ni

      - name: Install
        run: nci

      - name: Build
        run: nr build

      - name: Test
        run: nr test
```
### 2.2.2 release 发布
git push tag 时触发，用 [changelogithub](https://github.com/antfu/changelogithub) 生成 changelog。
[其中自动令牌身份验证 secrets.GITHUB_TOKEN](ttps://docs.github.com/zh/actions/security-guides/automatic-token-authentication)

```yml
# vscode-open-in-github-button/.github/workflows/release.yml

name: Release

# 赋予 secrets.GITHUB_TOKEN 写内容的权限
permissions:
  contents: write

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v3
        with:
          node-version: 16.x

      - run: npx changelogithub
        env:
          GITHUB_TOKEN: ${{secrets.GITHUB_TOKEN}}
```

## 2.3 入口文件 index.ts
```js
import { StatusBarAlignment, window } from 'vscode'

export function activate() {
  const statusBar = window.createStatusBarItem(StatusBarAlignment.Left, 0)
  statusBar.command = 'openInGitHub.openProject'
  statusBar.text = '$(github)'
  statusBar.tooltip = 'Open in GitHub'
  statusBar.show()
}

export function deactivate() {

}
```
点击状态栏的 `open in github `按钮，其实执行的是 `'openInGitHub.openProject'` 命令。

## 2.4 openInGitHub.openProject

翻看文档和 项目`package.json` 文件，我们可以得知，这个命令并不是官方提供的，而是依赖第三方的扩展插件。
```json
{
  "extensionPack": [
    "fabiospampinato.vscode-open-in-github"
  ],
}
```
我们来看这个插件，[vscode-open-in-github](https://github.com/fabiospampinato/vscode-open-in-github.git)。

Opening the project

![](https://github.com/fabiospampinato/vscode-open-in-github/raw/master/resources/demo/project.gif)

Opening the file
![](https://github.com/fabiospampinato/vscode-open-in-github/raw/master/resources/demo/file.gif)



# 3. vscode-open-in-github package.json
   找到 **main** 入口文件。

```json
{
  "main": "out/extension.js",
}
```

# 4. 调试项目
克隆项目，然后安装依赖
```bash
git clone https://github.com/fabiospampinato/vscode-open-in-github.git
cd vscode-open-in-github

npm install
# 或者
# npm i -g yarn
yarn install
# 或者
# npm i -g pnpm
pnpm install
```
选中 **package.json** 中的，**"main": "out/extension.js"**，按 **ctrl + shift + p**。输入选择 **debug launcher auto** 即可调试。

# 5. 入口 src/extension.ts
我们可以从 **webpack.config.js** 配置看到入口文件 **src/extension.ts**，可以提前去打好断点等。
```js
// vscode-open-in-github/webpack.config.js
const config = {
  target: 'node',
  entry: './src/extension.ts',
}
```
```ts
// vscode-open-in-github/src/extension.ts
/* IMPORT */
import Utils from './utils';
/* ACTIVATE */
const activate = Utils.initCommands;
/* EXPORT */
export {activate};
```
## 5.1 Utils 工具函数
```ts
/* IMPORT */
import * as _ from 'lodash';
import * as absolute from 'absolute';
import * as findUp from 'find-up';
import * as path from 'path';
import * as pify from 'pify';
import * as simpleGit from 'simple-git';
import * as vscode from 'vscode';
import * as Commands from './commands';
import Config from './config';

/* UTILS */
const Utils = {
  // 初始化命令
  initCommands ( context: vscode.ExtensionContext ) {
    /**
     * 
     * contributes: {
     * "commands": [
      {
        "command": "openInGitHub.openProject",
        "title": "Open in GitHub: Project"
      },
     * }
     *     
    */
    const {commands} = vscode.extensions.getExtension ( 'fabiospampinato.vscode-open-in-github' ).packageJSON.contributes;
    commands.forEach ( ({ command, title }) => {
      const commandName = _.last ( command.split ( '.' ) ) as string,
            // openProject
            handler = Commands[commandName],
            // 注册 openProject 命令，函数是 Commands[openProject]
            disposable = vscode.commands.registerCommand ( command, () => handler () );
      context.subscriptions.push ( disposable );
    });
    return Commands;
  },
}
```
## 5.2 Commands 导出的命令函数
```ts
// vscode-open-in-github/src/commands.ts
/* IMPORT */
import URL from './url';
/* COMMANDS */
function openProject () {
  return URL.open ();
}
// 省略了若干其他命令
export { openProject };
```
导出的函数，我们可以看出是调用的 **URL.open** 函数。
我们可以接着看 **URL** 对象。

## 5.3 URL 对象
```ts
// vscode-open-in-github/src/url.ts
/* IMPORT */
import * as _ from 'lodash';
import * as vscode from 'vscode';
import Config from './config';
import Utils from './utils';

/* URL */
const URL = {
  // 省略 get copy 函数
  async get ( file = false, permalink = false, page? ) {},
  async copy ( file = false, permalink = false, page? ) {},
  // 调用的函数
  async open ( file = false, permalink = false, page? ) {
    const url = await URL.get ( file, permalink, page );
    vscode.env.openExternal ( vscode.Uri.parse ( url ) );
  }
}
```
之前如果我们在 **URL.get** 这里断点。
我们可以跟着断点调试，来看 **URL.get** 函数。

### 5.3.1 URL.get 函数
```ts
const URL = {
  async get ( file = false, permalink = false, page? ) {
    // 获取仓库路径
    const repopath = await Utils.repo.getPath ();
    if ( !repopath ) return vscode.window.showErrorMessage ( 'You have to open a git project before being able to open it in GitHub' );

    // 根据仓库路径获取 git 实例
    const git = Utils.repo.getGit ( repopath ),
    // 用 git 获取 url
          repourl = await Utils.repo.getUrl ( git );
          // 没有找到仓库地址，报错
    if ( !repourl ) return vscode.window.showErrorMessage ( 'Remote repository not found' );
    // 获取用户配置或者默认配置
    /***
     * import * as vscode from 'vscode';
        const Config = {
          get ( extension = 'openInGitHub' ) {
            return vscode.workspace.getConfiguration ().get ( extension ) as any;
          }
        };
     * export default Config;
     * */
    const config = Config.get ();
    let filePath = '',
        branch = '',
        lines = '',
        hash = '';

    // 省略 file 的逻辑
    branch = encodeURIComponent ( branch );
    filePath = encodeURIComponent ( filePath ).replace ( /%2F/g, '/' );
    // 拼接 url
    const url = _.compact ([ repourl, page, branch, hash, filePath, lines ]).join ( '/' );
    return url;
  },
}
```
我们来重点看下 **Utils.repo** 对象导出的函数。

## 5.4 Utils.repo 对象
```ts
/* IMPORT */
import * as _ from 'lodash';
import * as absolute from 'absolute';
import * as findUp from 'find-up';
import * as path from 'path';
import * as pify from 'pify';
import * as simpleGit from 'simple-git';
import * as vscode from 'vscode';
import * as Commands from './commands';
import Config from './config';

const Utils = {
  repo: {
    // 用 simpleGif 仓库获取 git
    getGit ( repopath ) {
      return pify ( _.bindAll ( simpleGit ( repopath ), ['branch', 'getRemotes'] ) );
    },
    // 获取 hash
    async getHash ( git ) {
      return ( await git.revparse ([ 'HEAD' ]) ).trim ();
    },
    // 通过目前路径，获取根路径
    async getPath () {
      const {activeTextEditor} = vscode.window,
            editorPath = activeTextEditor && activeTextEditor.document.uri.fsPath,
            rootPath = Utils.folder.getRootPath ( editorPath );
      if ( !rootPath ) return false;
      // 获取路径
      return await Utils.folder.getWrapperPathOf ( rootPath, editorPath || rootPath, '.git' );
    },
}
```
### 5.4.1 Utils.repo.getUrl 获取 Url
```js
const Utils = {
  repo: {
    async getUrl ( git ) {
      // 获取配置信息
      const config = Config.get(),
      // 远程信息
            remotes = await git.getRemotes ( true ),
            remotesGithub = remotes.filter ( remote => ( remote.refs.fetch || remote.refs.push ).includes ( config.github.domain ) ),
            remoteOrigin = remotesGithub.filter ( remote => remote.name === config.remote.name )[0],
            remote = remoteOrigin || remotesGithub[0];
      if ( !remote ) return;
      const ref = remote.refs.fetch || remote.refs.push,
            re = /\.[^.:/]+[:/]([^/]+)\/(.*?)(?:\.git|\/)?$/,
            match = re.exec ( ref );
      if ( !match ) return;
      return `https://${config.github.domain}/${match[1]}/${match[2]}`;
    }
  }
}
```
总结一下大致流程：

- **vscode** 使用 **vscode.commands.registerCommand ( command, () => handler () )** 注册 **openProject** 命令
- **ctrl + shift + p** 输入 **>open in github**，选择触发 **openInGithub.openProject** 命令
- 执行 **openProject** 函数，实际调用 **URL.open()** 函数
  ```js
  async open ( file = false, permalink = false, page? ) {
    const url = await URL.get ( file, permalink, page );
    vscode.env.openExternal ( vscode.Uri.parse ( url ) );
  }
  ```

- 实际调用的是 **URL.get** 函数
  - 先根据 vscode 的能力，获取到仓库的路径
  - 再根据仓库的路径，获取 git 实例（simple-git）
  - 根据 git 实例，获取到仓库的 url

- 最后打开仓库 url 链接