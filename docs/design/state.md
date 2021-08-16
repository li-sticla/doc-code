---
title: 状态设计
toc: menu
order: 2
---



## 1.小场面

通常是单一组件内部的状态，不涉及跨组件层级传递。这类**状态**大多存在于父子组件之间。

### 1.1.状态提升：

### 1.2.组合组件：

## 2.缓存状态：

不同于交互的中间状态，服务端状态更应被归类为**缓存**，有如下性质：

+ 通常以**异步**的形式请求、更新
+ **状态**由请求的数据源控制，不由前端控制
+ **状态**可以由不同组件共享

作为可以由不同组件共享的**缓存**，还需要考虑更多问题，比如：

- 缓存失效
- 缓存更新

### 2.1.React-Query：

#### 2.1.1.介绍：

本项目使用`React-Query`管理服务端状态。

> 这是一个适用于 React Hooks 的请求库。 这个库将帮助你获取、同步、更新和缓存你的远程数据， 提供两个简单的 hooks，就能完成增删改查等操作。
>
> React-Query 使用声明式管理服务端状态，可以使用零配置开箱即用地处理缓存，后台更新和陈旧数据。无需繁琐的配置，编写 useReduce，以及维护全局状态，只要知道如何使用 Promise 或 async / await ，传递一个可解析数据（或引发错误）的函数，剩下的交给 React-Query ，使用更少的代码获得更高的效率。

>另一个可选方案是[SWR](https://swr.vercel.app/zh-CN)。

#### 2.1.2.最佳实践：

本项目中实现的 `useConfig` Hook

**说明：**

基于`React-Query`实现乐观更新(*optimistic updates*)、错误回滚。

**具体设计：**

`useConfig`:

通用配置，返回 `reqct-query` 需要用到的回调 `onSuccess`、`onMutate`、`onError`。

参数：`queryKey`: 查询的键值，`callback`：`onMutate`回调中执行的函数。

```ts
export const useConfig = (
  queryKey: QueryKey,
  callback: (target: any, old?: any[]) => any[]
) => {
  const queryClient = useQueryClient();

  return {
    onSuccess: () => {
      queryClient.invalidateQueries(queryKey);
      message.success("操作成功", 3);
    },

    async onMutate(target: any) {
      const previousItems = queryClient.getQueryData(queryKey);
      queryClient.setQueryData(queryKey, (old?: any[]) => {
        return callback(target, old);
      });
      return { previousItems };
    },

    onError(error: any, newItem: any, context: any) {
      queryClient.setQueryData(queryKey, context.previousItems);
      message.error(error.message);
    },
  };
};
```

`useEditConfig`:

扩展`useConfig`配置，在 `onMutate `回调中实现编辑替换操作。

参数：`queryKey`: 查询的键值。

```ts
export const useEditConfig = (queryKey: QueryKey) =>
  useConfig(
    queryKey,
    (target, old) =>
      old?.map((item) => //查找缓存中与目标 id 相同的键值对列表，并进行覆盖替换
        item.id === target.id ? { ...item, ...target } : item
      ) || []
  );
```

**具体使用:**

`useEditProject`:

编辑项目列表，在发送请求后执行`onMutate` 回调，请求成功则执行 `onSuccess`，同时操作保留，请求失败则执行 `onError` 回滚操作。

```ts
export const useEditProject = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation(
    (params: Partial<Project>) =>
      client(`projects/${params.id}`, {
        data: params,
        method: "PATCH",
      }),
    useEditConfig(queryKey)
  );
};
```

```ts
const { mutate: editProject } = useEditProject(useProjectsQueryKey());
const pinProject = (id: number) => (pin: boolean) => editProject({ id, pin });
```

## 3.客户端状态：

主要是指客户端内部逻辑上相距较远、层级跨度较大的组件之间传递的状态。

此类**状态**适合全局管理。

### 3.1.URL：

#### 3.1.1.介绍：

使用 `URL` 和 `react-router` 也可以管理全局状态。

将应用程序的状态保存在 URL 当中，最大的优势在于：当你复制，粘贴链接的时候，由于状态包含在 URL 当中，我们的应用程序在首次加载的时候，将会保持这个状态。

> 我们可以使用浏览器提供的 [URLSearchParams](https://developer.mozilla.org/zh-CN/docs/Web/API/URLSearchParams) API 来处理 URLSearchParams。

本项目使用 `react-router-dom` 提供的 `useSearchParams` 通过 URLSearchParams 接口读取和写入搜索参数，

结合 React 自定义 Hook 进行使用，能够最大化地发挥 URL 管理的优势：轻量便捷的状态管理机制。

#### 3.1.2.最佳实践：

本项目中实现的 `useUrlQueryParam` Hook。

**说明：**

负责处理从 URL 中读取或写入指定的参数 。

**具体设计：**

`useUrlQueryParam`:

参数：键值对`keys`: `[]`，如：`["name", "personId"]`。

```ts
export const useUrlQueryParam = <K extends string>(keys: K[]) => { 
  const [searchParams] = useSearchParams();
  const setSearchParams = useSetUrlSearchParam();
  const [stateKeys] = useState(keys);
  return [
    useMemo(
      () =>         //返回searchParams对象中包含stateKeys的键值对
        subset(Object.fromEntries(searchParams), stateKeys) as { 
          [key in K]: string;
        },
      [searchParams, stateKeys]
    ),
    (params: Partial<{ [key in K]: unknown }>) => {
      return setSearchParams(params);
    },
  ] as const;
};
```

`useSetUrlSearchParam`:

将 set 方法单独抽离出来方便使用。

参数：对象`param`:`{ key: value }` 如：`{ name: "Jack", personId: 1 }`。

```ts
export const useSetUrlSearchParam = () => {
  const [searchParams, setSearchParam] = useSearchParams();
  return (params: { [key in string]: unknown }) => {
    const o = cleanObject({
      ...Object.fromEntries(searchParams),
      ...params, // 用params覆盖searchParams中对应键值对,并清理值为空的属性
    }) as URLSearchParamsInit;
    return setSearchParam(o);
  };
};
```

### 3.2.Redux：

React-Redux配合redux使用，将redux定义的store数据注入到组件中，可以使组件轻松的拿到全局状态，方便组件间的通信。使react组价与redux数据中心（store）联系起来，调用dispatch函数修改数据状态后，触发通过subscribe注册更新视图的处理逻辑，包括需要渲染的数据和更新数据的函数。

它主要用于在入口处包裹需要用到Redux的组件。本质上是将store放入context里。

### 3.3.Context：

