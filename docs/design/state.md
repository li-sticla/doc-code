---
title: 状态设计
toc: menu
order: 2
---



## 1.Redux：

React-Redux配合redux使用，将redux定义的store数据注入到组件中，可以使组件轻松的拿到全局状态，方便组件间的通信。使react组价与redux数据中心（store）联系起来，调用dispatch函数修改数据状态后，触发通过subscribe注册更新视图的处理逻辑，包括需要渲染的数据和更新数据的函数。

它主要用于在入口处包裹需要用到Redux的组件。本质上是将store放入context里。