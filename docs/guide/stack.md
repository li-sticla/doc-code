---
title: 技术选型及相关说明
toc: menu
order: 2
---
## 1.CI/CD、静态站点托管💻
使用 CI / CD 工具可自动完成构建，测试和部署新代码的过程。即使只更改了其中一行甚至是一个字符，团队成员都可以立即获得有关其代码生产准备情况的反馈。如此一来，每位团队成员都可以将他们的代码推送到生产体系当中，而构建，测试和部署的过程则自动完成，以便他们放心大胆地继续处理应用程序的下一部分。

### 1.1.CloudBase Webify:

本项目文档利用腾讯打造的云上开发部署平台 [CloudBase Webify](https://cloud.tencent.com/product/webify)进行静态站点托管，并基于 Web 应用托管服务的内置 CI/CD 能力，只需要将变更推送至 Git 仓库，便可触发应用的构建和重新部署。
除此之外，Web 应用托管还为每个应用分配一个专属的支持 HTTPS的默认域名，并提供 CDN 接入能力。

### 1.2.Github-Pages:

由于CloudBase Webify 仍处于开发阶段，**边缘路由**功能还未正式上线，无法配置应用的路由逻辑，影响到项目的正常功能，本项目将成品托管于 github-pages。

#### 👀 [项目地址](https://k1sumi.github.io/)

#### 1.2.1.安装 gh-pages：

```sh
npm install -D gh-pages
```

#### 1.2.2.发布：

在 package.json 中添加发布脚本：

```json
// ...
"scripts": {
	// ...
	"predeploy": "npm run build",
	"deploy": "gh-pages -d build"
}
```

`predeploy` 将会在 `deploy` 运行前自动运行。

运行命令进行发布：

```sh
npm run deploy
```

#### 1.2.3.确认代码分支:

在项目的 settings 的 GitHun Pages 设置部分中，确保源代码 Source 使用的是 gh-pages 分支。

#### 1.2.4.路由配置：

在项目 public 目录新建 `404.html`:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Single Page Apps for GitHub Pages</title>
  <script type="text/javascript">
    // Single Page Apps for GitHub Pages
    // MIT License
    // https://github.com/rafgraph/spa-github-pages
    // This script takes the current url and converts the path and query
    // string into just a query string, and then redirects the browser
    // to the new url with only a query string and hash fragment,
    // e.g. https://www.foo.tld/one/two?a=b&c=d#qwe, becomes
    // https://www.foo.tld/?/one/two&a=b~and~c=d#qwe
    // Note: this 404.html file must be at least 512 bytes for it to work
    // with Internet Explorer (it is currently > 512 bytes)

    // If you're creating a Project Pages site and NOT using a custom domain,
    // then set pathSegmentsToKeep to 1 (enterprise users may need to set it to > 1).
    // This way the code will only replace the route part of the path, and not
    // the real directory in which the app resides, for example:
    // https://username.github.io/repo-name/one/two?a=b&c=d#qwe becomes
    // https://username.github.io/repo-name/?/one/two&a=b~and~c=d#qwe
    // Otherwise, leave pathSegmentsToKeep as 0.
    var pathSegmentsToKeep = 0;

    var l = window.location;
    l.replace(
      l.protocol + '//' + l.hostname + (l.port ? ':' + l.port : '') +
      l.pathname.split('/').slice(0, 1 + pathSegmentsToKeep).join('/') + '/?/' +
      l.pathname.slice(1).split('/').slice(pathSegmentsToKeep).join('/').replace(/&/g, '~and~') +
      (l.search ? '&' + l.search.slice(1).replace(/&/g, '~and~') : '') +
      l.hash
    );

  </script>
</head>
<body>
</body>
</html>

```

在 `public/index.html`中加入以下 script：

```html
<!-- Start Single Page Apps for GitHub Pages -->
<script type="text/javascript">
  // Single Page Apps for GitHub Pages
  // MIT License
  // https://github.com/rafgraph/spa-github-pages
  // This script checks to see if a redirect is present in the query string,
  // converts it back into the correct url and adds it to the
  // browser's history using window.history.replaceState(...),
  // which won't cause the browser to attempt to load the new url.
  // When the single page app is loaded further down in this file,
  // the correct url will be waiting in the browser's history for
  // the single page app to route accordingly.
  (function (l) {
    if (l.search[1] === "/") {
      var decoded = l.search
        .slice(1)
        .split("&")
        .map(function (s) {
          return s.replace(/~and~/g, "&");
        })
        .join("?");
      window.history.replaceState(
        null,
        null,
        l.pathname.slice(0, -1) + decoded + l.hash
      );
    }
  })(window.location);
</script>
<!-- End Single Page Apps for GitHub Pages -->
```

## 2.工程规范📄

为了提高整体开发效率，首先要将一些代码规范考虑在内，需要保持git仓库的代码格式统一。

综合考虑后使用组合工具：
1. 规范提交代码（必需）：`eslint + stylelint + prettier + husky + lint-staged`
2. 规范commit日志（必需）：`commitlint + commitizen + cz-conventional-changelog`
### 2.1.规范提交代码:
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

### 2.2.规范 git 提交日志:

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

## 3.打包构建🧳

本项目使用目前最流行的打包工具 webpack 进行应用打包，经 webpack 打包压缩后的产物即可直接部署到服务器上。

### 3.1.Webpack:

>本质上，[webpack](https://webpack.js.org/) 是一个用于现代 JavaScript 应用程序的 *静态模块打包工具*。当 webpack 处理应用程序时，它会在内部构建一个 [依赖图(dependency graph)](https://webpack.docschina.org/concepts/dependency-graph/)，此依赖图对应映射到项目所需的每个模块，并生成一个或多个 *bundle*。
>
>它可以打包你的 JavaScript 应用程序（支持 ESM 和 CommonJS），可以扩展为支持许多不同的静态资源，例如：images, fonts 和 stylesheets。

#### 3.1.1.安装：

```sh
npm install --save-dev webpack
```

#### 3.1.2.覆盖默认配置：

使用 CRA 即 create-react-app 创建的应用已经有一套完善的webpack配置。如果需要修改 webpack 默认配置，可以选择`react-app-rewired`方案。

安装 customize-cra用来获得一组 CRA 2.0兼容的 rewirers，
babel-plugin-import是一个用于按需加载组件代码和样式的 babel 插件：

```sh
npm i react-app-rewired customize-cra babel-plugin-import
```

在项目根目录创建一个` config-overrides.js `用于修改默认配置:

```js
// 修改 webpack 默认配置，目的：按需引入 antdesign
// ......customize-cra 包含很多api
 const { override, fixBabelImports } = require('customize-cra');

 const rewiredMap = () => config => {
   // config 为所有的 webpack 配置
   
   config.devtool = config.mode === 'development' ? 'cheap-module-source-map' : false  // 生产环境关闭 sourcemap
   return config
 }
 module.exports = override(
   fixBabelImports('import', {
     libraryName: 'antd',
     libraryDirectory: 'es',
     style: 'css',
   }),
   rewiredMap()
 );
```

#### 3.1.3.打包：

`package.json`添加一个执行打包的 script:

```json
   "scripts":{
    ......
    "build": "webpack",
}
```

在命令行中执行打包脚本：
```sh
npm run build
```

#### 3.1.4.使用 webpack-dev-server 预览 :

>`webpack-dev-server` 提供了一个基本的 web server，并且具有 live reloading(实时重新加载)。

`package.json`添加一个可以直接运行 dev server 的 script：

```
   "scripts":{
    ......
    "start": "webpack serve --open",
}
```

现在，在命令行中运行 `npm start`，我们会看到浏览器自动加载页面。如果你更改任何源文件并保存它们，web server 将在编译代码后自动重新加载。

## 4.Mock测试🧪

### 4.1.Mock：

Mock主要是通过正常请求在后端接口进度落后的情况下，还能获取到和后端约定数据结构一样的模拟数据的一门技术，以避免后端接口进度滞后影响我们正常的开发。

本项目采用**Mock Server**方案，依赖本地 node 服务器，服务器提供接口产生相应的 mock 数据。

考虑后使用组合工具：`node/express/json-server + mockjs/fakejs`

#### 4.1.1.安装 [json-server](https://github.com/typicode/json-server)：

```sh
npm install -g json-server
```

#### 4.1.2.启动 json-server：

> `json-server`是可以直接把一个`json`文件托管成一个具备全`RESTful`风格的`API`, 并支持跨域、`jsonp`、路由订制、数据快照保存等功能的 web 服务器。

通过以下命令，把`db.json`文件托管成一个 web 服务。

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

#### 4.1.5.自动化操作：

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

> `Service workers `本质上充当 Web 应用程序、浏览器与网络（可用时）之间的代理服务器。这个 API 旨在创建有效的离线体验，它会拦截网络请求并根据网络是否可用采取来适当的动作、更新来自服务器的的资源。它还提供入口以推送通知和访问后台同步 API。借助于`Service Worker`，可以轻松实现对网络请求的控制，对于不同的网络请求，采取不同的策略。

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

## 5.调试Debug🐛

### 



## 6.语言及规范🌐

### 6.1.CSS-in-JS:

本项目采用 CSS-in-JS 的方式高效编写 CSS 样式。

> 指组织 CSS 代码的一种方式，而不是一个具体的库实现。实现了CSS-in-JS的库有很多，代表库有 [styled-component](https://styled-components.com/docs) 和 [emotion](https://emotion.sh/docs/introduction)。
>
> CSS-in-JS 将应用的CSS样式写在 JavaScript 文件里面，而不是独立为一些`.css`，`.scss`或者`less `之类的文件。
>
> CSS-in-JS 可以用模块化的方式组织 CSS， 依托于 JS 的模块化方案。可以在 CSS 中使用一些属于 JS 的诸如模块声明，变量定义，函数调用和条件判断等语言特性来提供灵活的可扩展的样式定义。

#### 6.1.1.安装 emotion 库：

```sh
npm install --save @emotion/react @emotion/styled
```

#### 6.1.2.结合 styled 使用：

引入 styled，以 `styled.HTMLElement`` `或`styled(Component)`` `格式编写样式：

```js
import styled from "@emotion/styled";

export const Row = styled.div<{
  gap?: number | boolean;
  between?: boolean;
  marginBottom?: number;
}>`
  display: flex;
  align-items: center;
  justify-content: ${(props) => (props.between ? "space-between" : undefined)};
  margin-bottom: ${(props) => props.marginBottom + "rem"};
  > * {
    margin-top: 0 !important;
    margin-bottom: 0 !important;
    margin-right: ${(props) =>
      typeof props.gap === "number"
        ? props.gap + "rem"
        : props.gap
        ? "2rem"
        : undefined};
  }
`;
export const ButtonNoPadding = styled(Button)`
  padding: 0;
`;
```

### 6.2.TypeScript：



## 7.React相关⚛️

本项目不同于传统 web 应用，是基于 React 的单页面应用 (SPA)，其中涉及一些 React 相关的技术栈。

### 7.1.React：

> [React](http://facebook.github.io/react/) 是一个为数据提供渲染为 HTML 视图的开源 JavaScript 库。React 不是一个框架 —— 它的应用甚至不局限于 Web 开发，它可以与其他库一起使用以渲染到特定环境。例如，[React Native](https://reactnative.dev/) 可用于构建移动应用程序；[React 360](https://facebook.github.io/react-360/) 可用于构建虚拟现实应用程序……

> React 的主要目标是最大程度地减少开发人员构建 UI 时发生的错误。它通过使用组件——描述部分用户界面的、自包含的逻辑代码段——来实现此目的。这些组件可以组合在一起以创建完整的 UI，React 将许多渲染工作进行抽象化，使开发者可以专注于 UI 设计。

#### 7.1.1.安装：

全局安装脚手架：

```sh
npm i -g create-react-app
```

#### 7.1.2.使用：

使用 create-react-app 脚手架快速创建应用：

```sh
cd $your_dir
create-react-app react-demo
```



### 7.2.React-Router:

本项目在 React 单页面应用的基础上使用 React-Router 路由库，通过建立组件与路由之间的映射关系实现类似多个页面的效果。

> [React Router](https://github.com/ReactTraining/react-router) 是一个基于 [React](http://facebook.github.io/react/) 之上的强大路由库，它可以让你向应用中快速地添加视图和数据流，同时保持页面与 URL 间的同步，是完整的 React 路由解决方案。它拥有简单的 API 与强大的功能例如代码缓冲加载、动态路由匹配、以及建立正确的位置过渡处理。

#### 7.2.1.安装：

根据应⽤运行的环境选择安装 react-router-dom (在浏览器中使⽤)，本项目版本为 v6.0.0-beta.0

```sh
npm install --save react-router@6 react-router-dom@6
```

顺便安装history(history是为React Router提供核心功能的包):

```
npm install history
```

#### 7.2.2.使用:

**路由配置:**

路由配置是一组指令，用来告诉 router 如何匹配 URL以及匹配后如何执行代码。

示例：

```tsx | pure
import { Navigate, Route, Routes } from "react-router";
import { BrowserRouter as Router } from "react-router-dom";
<Router>
  <Routes>
    <Route
      path={"/projects"}
      element={
        <ProjectListScreen
          projectButton={
            <ButtonNoPadding
              onClick={() => setProjectModalOpen(true)}
              type={"link"}
            >
              创建项目
            </ButtonNoPadding>
          }
        />
      }
    />
    <Route path={"/projects/:projectId/*"} element={<ProjectScreen />} />
    <Navigate to={"/projects"} />
  </Routes>
</Router>;
```

### 7.3.React-Redux：

本项目开发中期使用 React-Redux 管理部分组件的状态。(后期重构为使用 Context 接管)

> **[Redux](https://redux.js.org/) 是一个使用叫做 `action` 的事件来管理和更新应用状态的模式和工具库**。它以集中式Store（centralized store）的方式对整个应用中使用的状态进行集中管理，其规则确保状态只能以可预测的方式更新。

> [React-Redux](https://react-redux.js.org/) 是 Redux 的 React 版本，它使得 Redux 能更好地与 React 的特性结合。

#### 7.3.1.安装：

同时安装 [React-Redux](https://react-redux.js.org/) 及 [Redux Toolkit](https://redux-toolkit.js.org/) 工具库让 Redux 的使用更高效：

```sh
npm install react-redux @reduxjs/toolkit -d
```

#### 7.3.2.使用：

定义全局 store：

```ts
export const store = configureStore({
  reducer: rootReducer,
});
```

定义 reducer 切片：

```ts
export const authSlice = createSlice({
  name: "auth",
  initialState,
  reducers: {
    setUser(state, action) {// 以 Action creator 辅助函数方式返回 Action 对象
      state.user = action.payload;
    },
  },
});
```

将 reducer 切片组合为根 reducer：

```ts
export const rootReducer = {
  //reducer 切片
  projectList: projectListSlice.reducer,
  auth: authSlice.reducer,
};
```

用 useSelector 从根状态树中获取指定切片的状态：

```ts
export const selectUser = (state: RootState) => state.auth.user;

const user = useSelector(selectUser);
```

从 reducer 中获取 Action 对象：

```ts
const { setUser } = authSlice.actions;
```

对 dispatch 传入 Action 以改变状态：

```ts
  const dispatch = useDispatch();
// dispatch中如果传入的action类型为函数，则会由 redux-thunk 接管
// dispatch(login(form)) 执行异步操作
  export const login = (form: AuthForm) => (dispatch: AppDispatch) =>
  auth.login(form).then((user) => dispatch(setUser(user)));
```

### 7.4.React-Query：

本项目使用 React-Query 处理服务端缓存。

> [React Query](https://react-query.tanstack.com/) 通常被描述为 React 缺少的数据获取(data-fetching)库，但是从更广泛的角度来看，它使 React 程序中的**获取，缓存，同步和更新服务器状态**变得轻而易举。

>React Query 无疑是管理服务器状态的最佳库之一。它非常好用，**开箱即用，无需配置**，并且可以随着应用的增长而根据自己的喜好**进行定制**。

#### 7.4.1.安装：

```sh
npm i react-query
```

#### 7.4.2.使用：

该示例非常简要地说明了 React Query 的 3 个核心概念：

- [查询 Queries](https://cangsdarm.github.io/react-query-web-i18n/guides&concepts/queries)
- [修改 Mutations](https://cangsdarm.github.io/react-query-web-i18n/guides&concepts/mutations)
- [查询错误处理 Query Invalidation](https://cangsdarm.github.io/react-query-web-i18n/guides&concepts/query-invalidation)

```tsx | pure
import {
  useQuery,
  useMutation,
  useQueryClient,
  QueryClient,
  QueryClientProvider,
} from 'react-query'
import { getTodos, postTodo } from '../my-api'

// 创建一个 client
const queryClient = new QueryClient()

function App() {
  return (
    // 提供 client 至 App
    <QueryClientProvider client={queryClient}>
      <Todos />
    </QueryClientProvider>
  )
}

function Todos() {
  // 访问 client
  const queryClient = useQueryClient()

  // 查询最新数据
  const query = useQuery('todos', getTodos)
  
  //获取缓存数据
  const queryCache = queryClient.getQueryData('todos')

  // 修改
  const mutation = useMutation(postTodo, {
    onSuccess: () => {
      // 错误处理和刷新
      queryClient.invalidateQueries('todos')// 调用 invalidateQueries 使缓存失效
    },
  })

  return (
    <div>
      <ul>
        {query.data.map((todo) => (
          <li key={todo.id}>{todo.title}</li>
        ))}
      </ul>

      <button
        onClick={() => {
          mutation.mutate({
            id: Date.now(),
            title: 'Do Laundry',
          })
        }}
      >
        Add Todo
      </button>
    </div>
  )
}

render(<App />, document.getElementById('root'))
```

## 8.UI框架及组件库🐜

### 8.1.Ant Design of React:

本项目用到的基础展示组件来源于Ant Design 的组件库。

> Ant Design 源自蚂蚁金服体验技术部的后台产品开发，是一套提炼和应用于企业级后台产品的交互语言和视觉体系。

> [antd](https://ant.design/docs/react/introduce-cn) 是基于 Ant Design 设计体系的 React UI 组件库，主要用于研发企业级中后台产品，是目前前端界主流组件库之一。

#### 8.1.1.安装:

```sh
npm install antd --save-dev
```

#### 8.1.2.高级配置：

使用 create-react-app 创建的应用，默认无法进行主题配置。

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

```js
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

## 9.性能优化⬆️

本项目使用 React 开发性能监测插件。

### 9.1.Why Did You Render：

> [Why Did You Render ](https://github.com/welldone-software/why-did-you-render)是一个能够帮助侦测 React 组件重新渲染的库。

#### 9.1.1.安装：

```sh
npm install @welldone-software/why-did-you-render --save
```

#### 9.1.2.使用：

在 src 根目录新建`wdyr.js`文件：

```js
/// <reference types="@welldone-software/why-did-you-render" />
import React from 'react';

if (process.env.NODE_ENV === 'development') {
  const whyDidYouRender = require('@welldone-software/why-did-you-render');
  whyDidYouRender(React, {
    trackAllPureComponents: true, //跟踪所有组件
  });
}
```

在`index.tsx`文件的第一行引入 ：

```js
import './wdyr'; // <--- first import
```

## 10.搜索引擎优化🔍

