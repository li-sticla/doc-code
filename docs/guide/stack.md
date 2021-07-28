---
title: 技术选型及相关配置
toc: menu
order: 2
---
## 1.CI/CD
使用 CI / CD 工具可自动完成构建，测试和部署新代码的过程。即使只更改了其中一行甚至是一个字符，团队成员都可以立即获得有关其代码生产准备情况的反馈。如此一来，每位团队成员都可以将他们的代码推送到生产体系当中，而构建，测试和部署的过程则自动完成，以便他们放心大胆地继续处理应用程序的下一部分。

本项目利用腾讯打造的云上开发部署平台 [CloudBase Webify](https://cloud.tencent.com/product/webify)进行快速开发，并基于 Web 应用托管服务的内置 CI/CD 能力，只需要将变更推送至 Git 仓库，便可触发应用的构建和重新部署。
除此之外，Web 应用托管还为每个应用分配一个专属的支持 HTTPS的默认域名，并提供 CDN 接入能力。
## 2.工程规范
为了提高整体开发效率，首先要将一些代码规范考虑在内，需要保持git仓库的代码格式统一。

考虑后使用组合工具：
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

安装[eslint-config-prettier](https://github.com/prettier/eslint-config-prettier#installation)以关闭eslint中与prettier冲突的规则：

```shell
npm install --save-dev eslint-config-prettier
```

#### 2.1.4.配置：

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
    "*.{js,css,md,ts,tsx}": "prettier --write" // 提交前强制格式化
  },
```

新建默认prettier配置文件让编辑器和其他工具意识到正在使用prettier：

```shell
echo {}> .prettierrc.json
```

在项目根目录新建[.prettierignore](https://prettier.io/docs/en/ignore.html)文件并写入以下内容来指定忽略哪些部分的文件:

```
# Ignore artifacts:
build
coverage
```

### 2.2.git 提交日志规范

在多人协作的项目中，如果Git的提交说明精准，在后期协作以及Bug处理时会变得有据可查，项目的开发可以根据规范的提交说明快速生成开发日志，从而方便开发者或用户追踪项目的开发信息和功能特性。

规范的Git提交说明:
+ 提供更多的历史信息，方便快速浏览

+ 可以过滤某些commit，便于筛选代码review

+ 可以追踪commit生成更新日志

+ 可以关联issues

Git提交说明结构:

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

#### 2.2.1.commitlint安装：

```shell
npm install --save-dev @commitlint/config-conventional @commitlint/cli
```

#### 2.2.2.配置：

在根目录执行以下命令作为commitlint的配置：

```shell
echo "module.exports = {extends: ['@commitlint/config-conventional']}" > commitlint.config.js
```

#### 2.2.3.激活 Husky 的 `commit-msg` hook：

```shell
npx husky install

npx husky add .husky/commit-msg
```

在新生成的 `.husky`文件夹下找到`commit-msg`文件，加入：

```shell
npx --no-install commitlint --edit $1
```

#### 2.2.4.commitizen及cz-conventional-changelog:

commitlint约定了提交的格式，但每次书写都需要记住那些约定，增加记忆负担。所以**使用cz工具让提交commit过程更加可视化**。

commitizen/cz-cli是一个可以实现规范的提交说明的工具，提供选择的提交信息类别，快速生成提交说明。

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



## 打包构建

## Mock测试

## 调试Debug

## 语言及规范

## React相关

## UI框架及组件

## 性能优化

## 搜索引擎优化（seo）