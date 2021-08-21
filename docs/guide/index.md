---
title: 介绍
toc: menu
order: 1
nav:
  path: /guide
  title: 项目指南
  order: 1
---

## 项目简介

仿 Jira 企业级项目管理系统。

从简单的单项目进度，到复杂的项目集、项目组合管理。

支持多级任务关联,任务看板/任务组列表模式随心切换, 安全、稳定、高效。

[线上演示](https://k1sumi.github.io/)

## 快速上手

### 环境准备
首先得有 [node](https://nodejs.org/zh-cn/)，并确保 node 版本是 10.13 或以上。
```shell
node -v
v10.13.0
```
### 项目初始化
+ 项目中所需要的依赖都会在`package.json`文件中展示。
+ 安装依赖需要使用npm命令，可去[npm官方文档](https://www.npmjs.cn/getting-started/installing-node/)使用及学习。
```shell
npm install
```
### 开始开发
执行 `npm start` 即可开始调试组件：
### 构建及部署
执行 `npm run build`即可对站点做构建，构建产物默认会输出到 build 目录下，我们可以将 build 目录部署在 now.sh、GitHub Pages 等静态站点托管平台或者某台服务器上。
