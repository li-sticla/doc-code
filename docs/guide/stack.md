---
title: 技术选型及相关说明
toc: menu
order: 2
---
## 1.CI/CD
使用 CI / CD 工具可自动完成构建，测试和部署新代码的过程。即使只更改了其中一行甚至是一个字符，团队成员都可以立即获得有关其代码生产准备情况的反馈。如此一来，每位团队成员都可以将他们的代码推送到生产体系当中，而构建，测试和部署的过程则自动完成，以便他们放心大胆地继续处理应用程序的下一部分。

本项目利用腾讯打造的云上开发部署平台 [CloudBase Webify](https://cloud.tencent.com/product/webify)进行快速开发，并基于 Web 应用托管服务的内置 CI/CD 能力，只需要将变更推送至 Git 仓库，便可触发应用的构建和重新部署。
除此之外，Web 应用托管还为每个应用分配一个专属的支持 HTTPS的默认域名，并提供 CDN 接入能力。
## 2.工程规范
为了提高整体开发效率，首先要将一些代码规范考虑在内，需要保持git仓库的代码格式统一。

综合考虑后使用组合工具：
1. 规范提交代码（必需）：`eslint + stylelint + prettier + husky + lint-staged`
2. 规范commit日志（必需）：`commitlint + commitizen + cz-conventional-changelog`
### 2.1.规范提交代码
针对写出符合团队代码规范的js、css代码。
> [eslint](https://eslint.org/): 对 js 做规则约束。（强制校验）
>
> [stylelint](https://stylelint.io/): 对 css 做规则约束。
>
> [prettier](https://prettier.io/): 代码格式化。（强制格式化）
>
> [husky](https://typicode.github.io/husky/#/) + [lint-staged](https://github.com/okonet/lint-staged)：本地git钩子工具。

#### 2.1.1. eslint、prettier 安装：

```shell
npm install --save-dev eslint

npm install --save-dev --save-exact prettier
```

#### 2.1.2.提交前强制校验：

将约束命令放置在提交代码前检查，这就要使用[husky](https://typicode.github.io/husky/#/) + [lint-staged](https://github.com/okonet/lint-staged)工具，工具能在提交代码precommit时调用钩子命令。

```shell
npm install husky lint-staged -D
```

#### 2.1.3.解决eslint、prettier冲突：

安装[eslint-config-prettier](https://github.com/prettier/eslint-config-prettier#installation)以关闭 eslint 中与 prettier 冲突的规则：

```shell
npm install --save-dev eslint-config-prettier
```

#### 2.1.4.配置：

激活 Husky 并新建一个 hook：

```sh
npx husky install

npx husky add .husky/pre-commit
```

在新生成的 `.husky`文件夹下找到`pre-commit`文件，加入以下内容，以启动 lint-staged：

```sh
npx lint-staged
```

配置`package.json`文件：

```json
  "eslintConfig": {
    "extends": [
      "react-app",
      "react-app/jest",
      "prettier" // 添加prettier规则以覆盖react原有规则
    ]
  },
  "lint-staged": {
    "*.{js,jsx,css,md,ts,tsx}": "prettier --write" // 对于指定后缀的文件提交前强制格式化
  },
```

新建默认 prettier 配置文件让编辑器和其他工具意识到正在使用prettier：

```shell
echo {}> .prettierrc.json
```

在项目根目录新建[.prettierignore](https://prettier.io/docs/en/ignore.html)文件并写入以下内容来指定忽略哪些部分的文件:

```
# Ignore artifacts:
build
coverage
```

### 2.2.规范 git 提交日志

在多人协作的项目中，如果Git的提交说明精准，在后期协作以及Bug处理时会变得有据可查，项目的开发可以根据规范的提交说明快速生成开发日志，从而方便开发者或用户追踪项目的开发信息和功能特性。

**规范的Git提交说明:**

+ 提供更多的历史信息，方便快速浏览

+ 可以过滤某些commit，便于筛选代码review

+ 可以追踪commit生成更新日志

+ 可以关联issues

**Git提交说明结构:**

Git提交说明可分为三个部分：Header、Body和Footer。

```html
<Header>

<Body>

<Footer>
```

Header部分包括三个字段type（必需）、scope（可选）和subject（必需）。

```shell
<type>(<scope>): <subject>
```

type 用于说明 commit 的提交性质。(Angular规范)

- feat 新增一个功能
- fix 修复一个Bug
- docs 文档变更
- style 代码格式（不影响功能，例如空格、分号等格式修正）
- refactor 代码重构
- perf 改善性能
- test 测试
- build 变更项目构建或外部依赖（例如scopes: webpack、gulp、npm等）
- ci 更改持续集成软件的配置文件和package中的scripts命令，例如scopes: Travis, Circle等
- chore 变更构建流程或辅助工具
- revert 代码回退

scope 用于说明影响的范围，比如数据层、控制层、视图层等等。

subject 为主题，简短描述的一行。

Body 为对 subject 的补充，可以多行。

Footer 主要是一些关联 issue 的操作。

#### 2.2.1.commitlint 安装：

```shell
npm install --save-dev @commitlint/config-conventional @commitlint/cli
```

#### 2.2.2.配置：

在根目录执行以下命令作为 commitlint 的配置：

```shell
echo "module.exports = {extends: ['@commitlint/config-conventional']}" > commitlint.config.js
```

#### 2.2.3.新建 Husky 的 `commit-msg` hook：

```shell
npx husky add .husky/commit-msg
```

在新生成的 `.husky`文件夹下找到`commit-msg`文件，加入：

```shell
npx --no-install commitlint --edit $1
```

#### 2.2.4.commitizen 及 cz-conventional-changelog 实现可视化:

commitlint 约定了提交的格式，但每次书写都需要记住那些约定，增加记忆负担。所以**使用 cz 工具让提交 commit 过程更加可视化**。

commitizen/cz-cli 是一个可以实现规范的提交说明的工具，提供选择的提交信息类别，快速生成提交说明。

安装：

```sh
npm i commitizen cz-conventional-changelog -D
```

配置：

使用 cz-conventional-changelog 适配器（介于AngularJS规范），约定提交代码时的选项。

`.package.json`文件：

```sh
 "scripts": {
    "commit": "git add -A && git-cz"
},

 "config": {
    "commitizen": {
      "path": "cz-conventional-changelog"
    }
```

以后 git commit 都使用 npm run commit（即git-cz）代替。

## 3.打包构建

## 4.Mock测试

### 4.1.Mock：

Mock主要是通过正常请求在后端接口进度落后的情况下，还能获取到和后端约定数据结构一样的模拟数据的一门技术，以避免后端接口进度滞后影响我们正常的开发。

本项目采用**Mock Server**方案，依赖本地 node 服务器，服务器提供接口产生相应的mock数据。

考虑后使用组合工具：`node/express/json-server + mockjs/fakejs`

#### 4.1.1.安装 [json-server](https://github.com/typicode/json-server)：

```sh
npm install -g json-server
```

#### 4.1.2.启动 json-server：

`json-server`是可以直接把一个`json`文件托管成一个具备全`RESTful`风格的`API`, 并支持跨域、`jsonp`、路由订制、数据快照保存等功能的 web 服务器。通过以下命令，把`db.json`文件托管成一个 web 服务。

```sh
json-server --watch db.json
```

#### 4.1.3.Mockjs 动态生成模拟数据：

例如启动 json-server 的命令：`json-server --watch app.js` 是把一个 js 文件返回的数据托管成 web 服务。

app.js 配合 [mockjs ](http://mockjs.com/)库可以很方便的进行生成模拟数据。

安装：

```sh
npm install mockjs
```

使用：

```js
// 用mockjs模拟生成数据
var Mock = require('mockjs');

module.exports = () => {
  // 使用 Mock
  var data = Mock.mock({
    'course|227': [
      {
        // 属性 id 是一个自增数，起始值为 1，每次增 1
        'id|+1': 1000,
        course_name: '@ctitle(5,10)',
        autor: '@cname',
        college: '@ctitle(6)',
        'category_Id|1-6': 1
      }
    ],
    'course_category|6': [
      {
        "id|+1": 1,
        "pid": -1,
        cName: '@ctitle(4)'
      }
    ]
  });
  // 返回的data会作为json-server的数据
  return data;
};
```

#### 4.1.4.自定义配置文件：

通过命令行配置路由、数据文件、监控等会让命令变的很长，而且容易敲错，可以把命令写到 npm 的 scripts 中，但是依然配置不方便。

为了解决这个问题，json-server 允许我们把所有的配置放到一个配置文件中，这个配置文件默认为`json-server.json`;

```json
{
  "port": 5000,              //自定义端口
  "watch": true,             //自动监听变化
  "static": "./public",
  "read-only": false,        //是否只能使用GET请求
  "no-cors": false,          //是否支持跨域
  "no-gzip": false,          //是否支持压缩
  "routes": "route.json"     //路由配置地址
}
```

路由配置文件`route.json`(简单示例，具体参见[官方文档](https://github.com/typicode/json-server#routes))：

```json
{
  "/api/*": "/$1"         //   /api/posts => /posts
}
```

#### 4.1.5.简化操作：

为方便项目使用，安装到本地：

```sh
npm i json-server -D
```

并在根目录新建文件夹 `__json_server_mock__` 用于放置托管文件。

在`package.json`中加入scripts：

```json
"scripts": {
"mock": "json-server --c json-server.json __json_server_mock__/db.json"
}
```

至此，使用 npm run mock 即可快速启动 json-server。

#### 4.1.6.请求代理：

项目中符合 RESTful API 规范的请求可以由 json-server 进行 mock，除此之外的自定义 API 则需要利用中间件自行编写逻辑进行模拟，无疑增加了开发的复杂度。

考虑后使用 [MSW(Mock Service Worker)](https://github.com/mswjs/msw) 以 [Service Worker](https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API) 为原理实现后端 API 模拟，对网络请求进行代理，经过后端逻辑处理后，以 `localStorage` 为数据库进行增删改查操作。

`Service workers `本质上充当 Web 应用程序、浏览器与网络（可用时）之间的代理服务器。这个 API 旨在创建有效的离线体验，它会拦截网络请求并根据网络是否可用采取来适当的动作、更新来自服务器的的资源。它还提供入口以推送通知和访问后台同步 API。借助于`Service Worker`，可以轻松实现对网络请求的控制，对于不同的网络请求，采取不同的策略。

登录判断请求示例：

```js
// src/mocks/handlers.js
import { rest } from 'msw'

export const handlers = [
  rest.post('/login', (req, res, ctx) => {
    // Persist user's authentication in the session
    sessionStorage.setItem('is-authenticated', 'true')

    return res(
      // Respond with a 200 status code
      ctx.status(200)
    )
  }),

  rest.get('/user', (req, res, ctx) => {
    // Check if the user is authenticated in this session
    const isAuthenticated = sessionStorage.getItem('is-authenticated')

    if (!isAuthenticated) {
      // If not authenticated, respond with a 403 error
      return res(
        ctx.status(403),
        ctx.json({
          errorMessage: 'Not authorized',
        })
      )
    }

    // If authenticated, return a mocked user details
    return res(
      ctx.status(200),
      ctx.json({
        username: 'admin',
      })
    )
  }),
]
```

具体使用方法见[官方文档](https://mswjs.io/docs/)。

## 5.调试Debug

## 6.语言及规范

### 6.1.CSS-in-JS:

指组织 CSS 代码的一种方式，而不是一个具体的库实现。实现了CSS-in-JS的库有很多，代表库有 styled-component 和 emotion。

CSS-in-JS 将应用的CSS样式写在 JavaScript 文件里面，而不是独立为一些`.css`，`.scss`或者`less `之类的文件。

CSS-in-JS 可以用模块化的方式组织 CSS， 依托于 JS 的模块化方案。可以在 CSS 中使用一些属于 JS 的诸如模块声明，变量定义，函数调用和条件判断等语言特性来提供灵活的可扩展的样式定义。

安装`emotion`库：

```sh
npm install --save @emotion/react
```









## 7.React相关





## 8.UI框架及组件库

### 8.1.Ant Design of React:

Ant Design 源自蚂蚁金服体验技术部的后台产品开发，是一套提炼和应用于企业级后台产品的交互语言和视觉体系。

[antd](https://ant.design/docs/react/introduce-cn) 是基于 Ant Design 设计体系的 React UI 组件库，主要用于研发企业级中后台产品，是目前前端界主流组件库之一。

#### 8.1.1.安装:

```sh
npm install antd --save-dev
```

#### 8.1.2.高级配置：

使用 create-react-app 创建的应用，无法进行主题配置。

此时我们需要对 create-react-app 的默认配置进行自定义。使用 [CRACO](https://github.com/gsoft-inc/craco)（**C**reate **R**eact **A**pp **C**onfiguration **O**verride）一个对 create-react-app 进行自定义配置的社区解决方案。

通过在应用程序的根目录添加一个` craco.config.js` 文件，可以自定义` eslint、babel、postcss` 配置等。

安装 craco 并修改 `package.json` 里的 `scripts` 属性。

```sh
yarn add @craco/craco
```

```json
/* package.json */
"scripts": {
-   "start": "react-scripts start",
-   "build": "react-scripts build",
-   "test": "react-scripts test",
+   "start": "craco start",
+   "build": "craco build",
+   "test": "craco test",
}
```

按照 [配置主题](https://ant.design/docs/react/customize-theme-cn) 的要求，自定义主题需要用到类似 [less-loader](https://github.com/webpack-contrib/less-loader/) 提供的 less 变量覆盖功能。我们可以引入 [craco-less](https://github.com/DocSpring/craco-less) 来帮助加载 less 样式和修改变量。安装 `craco-less`:

```sh
 yarn add craco-less
```

在项目根目录的`index.tsx `中修改样式引用为 less 文件：

```tsx
import 'antd/dist/antd.less'
```

然后在项目根目录创建一个 `craco.config.js` 用于修改默认配置。

```js
const CracoLessPlugin = require('craco-less');

module.exports = {
  plugins: [
    {
      plugin: CracoLessPlugin,
      options: {
        lessLoaderOptions: {
          lessOptions: {
            modifyVars: { '@primary-color': '#1DA57A' },
            javascriptEnabled: true,
          },
        },
      },
    },
  ],
};
```

这里利用了 [less-loader](https://github.com/webpack/less-loader#less-options) 的 `modifyVars` 来进行主题配置，变量和其他配置方式可以参考 [配置主题](https://ant.design/docs/react/customize-theme-cn) 文档。修改后重启 `yarn start`，如果看到一个绿色的按钮就说明配置成功了。

## 9.性能优化

## 10.搜索引擎优化（seo）