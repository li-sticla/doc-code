---
title: 项目规范
toc: menu
order: 3
---

## 1.项目结构规范
### 1.1.核心思想
+ 业务功能模块的相关代码都集中在一块，方便移动和删除
+ 实现关注点分离，方便开发、调试、维护、编写、查阅、理解代码

### 1.2.项目目录

## 2.资源命名规范

### 2.1.目录命名
+ 能直观的感受当前目录文件的作用
+ 以小驼峰方式命名
+ 特殊缩写名称可大写开头
```
│   ├── pages/                 
│   ├── components/
│   │   ├── UItemp/
```
### 2.2.页面命名
+ 能直观的感受当前文件的作用
+ 以小驼峰方式命名
```
│   │   │   ├── login.tsx       # 登录页面
│   │   │   ├── changePhone.tsx # 用户中心-修改电话号码页面
```
### 2.3.组件命名
+ 能直观的感受当前组件的用途
+ 组件命名始终是多个单词的，避免跟html元素或标签冲突
+ 要么大写开头，要么始终用横线链接

### 2.4.资源命名
+ 图片文件夹一般遵从页面或者模块命名,如：login/）
+ 图片不可随意命名，且严禁使用0，1，等数字直接命名图片。
+ 图片命名可遵循：用途+描述，多个单词用下划线隔开，如：login_icon.png,pwd_icon.png
+ 10k以下图片建议放置assets/img下（webpack打包时直接转为base64嵌入）
+ 大图且不常更换的图片放置public/img下
+ 可用css编写的样式严禁使用图片
+ 国际化图片，后缀使用简体-cn,英文-en,繁体-tw
```
│   ├── assets/               
│   │   ├── img/                          # 图片
│   │   │    ├── common/                  # 公共图片
│   │   │    │    ├── default_avatar.png  # 默认头像
│   │   │    ├── login/                   # 登录模块
│   │   │    │    ├── login1.png          # 登录模块图片
│   │   │    │    ├── login_icon-en.png      
│   │   │    │    ├── login_icon-cn.png     
│   │   │    │    ├── login_icon-tw.png      
│   │   │    ├── userInfo/                   # 用户中心模块的图片
```
## 3.项目路由规范

### 3.1.路由命名
+ 普通路由(非动态多级)命名，可以直接使用页面组件的命名。
### 3.2.多级路由
+ 动态多级路由，遵循：用途或作用或功能,嵌套不可超过三层。
```
/user/personal/infomaition  用户中心 -> 个人中心 -> 个人信息
/user/company/infomaition  用户中心 -> 企业中心 -> 企业信息
```
### 3.3.路由权限
+ 一般页面无需登录可查看，重要页面需要登录权限。
+ 登录权限字段在路由meta标签中确定。
## 4.CSS规范

### 4.1.css初始化及公用样式
```
/* ==========================================================================
   css 初始化
 ============================================================================ */
 
 /* margin padding 初始化为0 */
 body, h1, h2, h3, h4, h5, h6, hr, p, blockquote, dl, dt, dd, ul, ol, li, pre, form, fieldset, legend, button, input, textarea, th, td {
	margin: 0;
	padding: 0;
}

body, button, input, select, textarea {
	font: 12px/1.5tahoma, arial, \5b8b\4f53;
}

h1, h2, h3, h4, h5, h6 {	font-size: 100%;}
address, cite, dfn, em, var {	font-style: normal;}

code, kbd, pre, samp {font-family: couriernew, courier, monospace;}

small {font-size: 12px;}

ul, ol {list-style: none;}

a {text-decoration: none;}

a:hover {text-decoration: none;}

sup {vertical-align: text-top;}

sub {vertical-align: text-bottom;}

legend {color: #000;}

fieldset, img {border: 0;}

button, input, select, textarea {font-size: 100%;}

table {
	border-collapse: collapse;
	border-spacing: 0;
}

html, body {font-size: 10px !important;}

```
### 4.2.id/class 命名规则
+ 首先根据内容命名，如：nav,header
+ 内容中的子元素使用-链接，名称一律小写，如：card-item
+ 修饰类（易变的）使用--链接，如：card-item--warning
+ 若无内容，结合行为进行辅助，如：box-shawder
+ 不影响语义的情况下，可缩写，如：img-box, btn
+ 避免广告拦截词汇：ad, gg, banner, guagngao
### 4.3.属性
+ 顺序
1. 位置属性(position, top, right, z-index, display, float)
2. 大小(width, height, padding, margin)
3. 文字(font,line-height, letter-spacing, color)
4. 背景 (background, border)
5. 其他(animation, transition)
+ 简写
属性简写需要非常清楚属性值的正确顺序，而且在大多数情况下并不需要设置属性简写中包含的所有值，所以建议尽量分开声明会更加清晰；
```
/* not good */
.element {
    transition: opacity 1s linear 2s;
}
 
/* good */
.element {
    transition-delay: 2s;
    transition-timing-function: linear;
    transition-duration: 1s;
    transition-property: opacity;
}
```
## 5.编码规范

### 5.1.缩进
+ html, css, js 缩进一致，使用4个空格

### 5.2.空格
+ 二元运算符前后
+ 三元运算符前后
+ 代码块{前
+ 关键字前：else, while, catch, finally
+ 关键字后：if, else, for, while, do, switch, case, catch, finally, with, return
+ 单行注释//后，多行注释/*后
+ 对象的属性值前
+ for 循环
+ 函数的参数之间
+ 运算符前后
+ 函数声明，函数表达式`(`前不要空格，`)`后空格

### 5.3.空行
+ 变量声明后
+ 注释前
+ 代码块后
+ 文件最后保留一个空行

### 5.4.引号
+ 最外层统一使用单引号

### 5.5.模板字符串
+ 需要拼接，尽量少用+，多使用模板字符串

```
let str1 = 'yang'

let string = `hello ${str1}`

```
### 5.6.命名规范
+ 变量命名：小驼峰命名
+ 参数名：小驼峰命名
+ 函数名：小驼峰命名
+ 方法/属性名：小驼峰命名
+ 类名开头大写
+ 私有属性、变量和方法以下划线 _ 开头。
+ 常量名：全部使用大写+下划线
+ 由多个单词组成的缩写词，在命名中，根据当前命名法和出现的位置，所有字母的大小写与首字母的大小写保持一致。
+ 变量应当使用名词，尽量符合当时语义
+ boolean类型应当使用 is , has 等开头
+ 点击事件命名方式：tap + onClick()

### 5.7.二元三元操作符
+ 可用于简单的if替代
+ 操作符始终写在前一行，避免产生预想不到的问题
```
// 不推荐 
var x = a ? b : c;

// 推荐 
var y = a ?
    longButSimpleOperandB : longButSimpleOperandC;

var z = a ?
        moreComplicatedB :
        moreComplicatedC;
```
### 5.8.&& 和 ||
+ 二元布尔操作符是可短路的, 只有在必要时才会计算到最后一项。
```
// 不推荐
function foo(opt_win) {
  var win;
  if (opt_win) {
    win = opt_win;
  } else {
    win = window;
  }
  // ...
}

if (node) {
  if (node.kids) {
    if (node.kids[index]) {
      foo(node.kids[index]);
    }
  }
}

// 推荐
function foo(opt_win) {
  var win = opt_win || window;
  // ...
}

var kid = node && node.kids && node.kids[index];
if (kid) {
  foo(kid);
}
```
### 5.9.声明规范
尽量使用 let 声明变量，少用 var 
### 5.10.分号
以下情况需要加分号
+ 表达式
+ return
+ throw
+ break
+ continue
+ do-while
### 5.11.注释规范
+ 单行注释，独占一行，//后跟空格
+ 多行注释，/*后跟空格
+ 函数/方法注释,注释必须包含函数声明，有参数和返回值时必须注释标识
+ 参数和返回类型必须包含类型信息和说明
+ 当函数是内部函数，外部不可访问时，可使用@inner标识
```
/** 
 * 函数描述
 * @param {string} p1 参数说明
 * @param {string} p2 参数2的说明，比较长
 *     那就换行了.
 * @param {number=} p3 参数3的说明（可选）
 * @return {Object} 返回值描述
/*
function foo(p1, p2, p3) {
    return {
        p1: p1,
        p2: p2
    }
}
```
+ 组件块和子组件之间的注释
```
/* ==========================================================================
   组件块
 ============================================================================ */

/* 子组件块
 ============================================================================ */
.selector {
  padding: 15px;
  margin-bottom: 15px;
}

/* 子组件块
 ============================================================================ */
.selector-secondary {
  display: block; /* 注释*/
```
## 6.代码提交规范

提交格式：` git commit -m <type>[optional scope]: <description> `

常用type类别说明：
 * build：主要目的是修改项目构建系统(例如 glup，webpack，rollup 的配置等)的提交
 * ci：主要目的是修改项目持续集成流程(例如 Travis，Jenkins，GitLab CI，Circle等)的提交
 * docs：文档更新
 * feat：新增功能
 * fix：bug 修复
 * perf：性能优化
 * refactor：重构代码(既没有新增功能，也没有修复 bug)
 * style：不影响程序逻辑的代码格式修改(修改空白字符，补全缺失的分号等)
 * test：新增测试用例或是更新现有测试
 * revert：回滚某个更早之前的提交
 * chore：不属于以上类型的其他类型(日常事务，增加依赖库、工具，添加注释等)