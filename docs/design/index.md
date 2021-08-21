---
title: 模块设计
toc: menu
order: 1
nav:
  path: /design
  title: 项目设计
  order: 1
---

## 1.项目结构

```sh | pure
.
|-- __json_server_mock__            # json-server Mock 目录
|   |-- db.json                     	# 项目用 Mock 数据 
|   `-- middleware.js               	# json-server 中间件
|-- __tests__            			# 项目测试文件目录
|   |-- db.json                     	# 测试用 Mock 数据 
|   `-- ...   							# 其他测试文件
|-- public							# 公共资源目录，绕过 webpack 编译
|   |-- 404.html                    	# 404 页面
|   |-- index.html                  	# 项目唯一 html 文件，应用首页入口
|   |-- mockServiceWorker.js        	# mockServiceWorker 核心文件
|   `-- ...								# 其他静态资源文件
|-- src								# 项目的源代码目录
|   |-- App.css							# App 样式文件
|   |-- App.tsx                     	# App 根组件入口
|   |-- auth-provider.ts            	# 权限管理相关方法
|   |-- authenticated-app.tsx			# 认证通过页面
|   |-- index.css				 		# 应用全局样式文件
|   |-- index.tsx						# 应用入口
|   |-- assets							# 项目依赖资源目录，经过 webpack 编译 
|   |   |-- logo.svg						# 项目静态资源文件
|	|	`-- ...								# 其他依赖资源文件
|   |-- components						# 项目组件目录
|   |   |-- lib.tsx							# 基础组件
|   |   `-- ...								# 公共组件
|   |-- context							# 项目的 context 管理
|   |   |-- auth-context.tsx				# 权限管理 context
|   |   `-- index.tsx						# context 出口
|   |-- screens							# 项目业务页面目录
|   |   |-- epic							# 任务组页面目录
|   |   |-- kanban							# 看板页面目录
|   |   |-- project							# project 页面目录
|   |   `-- projectList						# project-list 页面目录
|   |-- types							# 项目类型声明目录
|   |   |-- index.ts						# 基本类型
|   |   `-- user.ts							# 其他业务用到的类型
|   |-- unauthenticated-app				# 未认证页面目录
|   |   |-- index.tsx						# 未认证页面入口
|   |   |-- login.tsx						# 登录页面 
|   |   `-- register.tsx					# 注册页面
|   |-- utils							# 项目公共工具集目录
|   |   |-- index.ts						# 基本工具库
|   |   `-- ...								# 其他业务用到的工具
|   `-- ...								# 其他配置文件
|-- tsconfig.json					# typescript 配置
|-- package.json					# 项目依赖配置
`-- yarn.lock						# 项目依赖版本信息
```

## 2.入口

### 2.1.应用入口

 `src/index.tsx`:

最顶部入口，经 webpack 编译处理依赖后注入 `index.html`

```tsx | pure
import "./wdyr";
import React from "react";
import ReactDOM from "react-dom";
import "./index.css";
import App from "./App";
import reportWebVitals from "./reportWebVitals";
import { DevTools, loadServer } from "jira-dev-tool";
import "antd/dist/antd.less";
import { AppProviders } from "context";

loadServer(() => //读取后
  ReactDOM.render(
    <React.StrictMode>
      <AppProviders>
        <DevTools />
        <App />
      </AppProviders>
    </React.StrictMode>,
    document.getElementById("root")
  )
);
```



## 页面

## 组件

