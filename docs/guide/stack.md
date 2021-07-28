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
> [eslint](https://eslint.org/): 对 js 做规则约束。强制校验
>
> [stylelint](https://stylelint.io/): 对 css 做规则约束
>
> [prettier](https://prettier.io/): 代码格式化。强制格式化
>
> [husky](https://typicode.github.io/husky/#/) +  **[lint-staged](https://github.com/okonet/lint-staged)**：本地git钩子工具

#### 2.1.1 eslint安装：

```shell
npm install --save-dev eslint
```



#### `.eslintrc.js`配置：

#### 

### 2.2.git 提交日志规范

在多人协作的项目中，如果Git的提交说明精准，在后期协作以及Bug处理时会变得有据可查，项目的开发可以根据规范的提交说明快速生成开发日志，从而方便开发者或用户追踪项目的开发信息和功能特性。

规范的Git提交说明:
+ 提供更多的历史信息，方便快速浏览
+ 可以过滤某些commit，便于筛选代码review
+ 可以追踪commit生成更新日志
+ 可以关联issues
commitlint

## 打包构建

## Mock测试

## 调试Debug

## 语言及规范

## React相关

## UI框架及组件

## 性能优化

## 搜索引擎优化（seo）