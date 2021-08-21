---
title: 状态设计
toc: menu
order: 2
---



## 1.小场面

通常是单一组件内部的状态，不涉及跨组件层级传递。这类**状态**大多存在于父子组件之间。

### 1.1.状态提升：

#### 1.1.1.介绍：

兄弟组件共享状态不符合 React 自上而下传递状态的范式。通常，多个兄弟组件需要反映相同的变化数据，这时建议将共享状态提升到最近的共同父组件中去。

> 状态提升多数情况下是为了解决兄弟组件之间的数据传递。

#### 1.1.1.最佳实践：

本项目的 ProjectListScreen 组件

**将 SearchPanel，List 组件数据提升至他们的共同父组件 ProjectListScreen，再以 props 形式传递下去，解决兄弟组件通信的同时减少了请求频率和冗余代码，提高应用性能：**

```tsx | pure
export const ProjectListScreen = () => {
  useDocumentTitle("项目列表", false);

  const [param, setParam] = useProjectsSearchParams();

  const debouncedParam = useDebounce(param, 2000);

  const { isLoading, error, retry, data: list } = useProjects(debouncedParam);

  const { data: users } = useUsers();

  const dispatch = useDispatch();

  return (
    <Container>
      <Row between={true}>
        <h1>项目列表</h1>
        <ButtonNoPadding
          onClick={() => dispatch(projectListActions.openProjectModal())}
          type={"link"}
        >
          创建项目
        </ButtonNoPadding>
      </Row>

      <SearchPanel users={users || []} param={param} setParam={setParam} />
      {error ? (
        <Typography.Text type={"danger"}>{error.message}</Typography.Text>
      ) : null}
      <List
        refresh={retry}
        loading={isLoading}
        dataSource={list || []}
        users={users || []}
      ></List>
    </Container>
  );
};
```

### 1.2.组合组件：

#### 1.2.1.介绍：

React 有着十分强大的组合模式, 用以实现组件间代码的重用。

与 vue 中的 slot 插槽类似，利用组件可以 JSX 嵌套的特性，我们可以将组件连同其内部状态一并传递下去，以解决组件嵌套过深时 props 传递的麻烦。

#### 1.2.2.最佳实践：

本项目中的 AuthenticatedApp 组件

**将 projectButton 连同其中的回调函数作为子组件传递，避免子组件中额外的 props 类型处理，同时使得组件具备可扩展性：**

```tsx | pure
//AuthenticatedApp
export const AuthenticatedApp = () => {
  const [projectModalOpen, setProjectModalOpen] = useState(false);

  return (
    <Container>
      <PageHeader
        projectButton={ //容器组件 projectButton
          <ButtonNoPadding
            onClick={() => setProjectModalOpen(true)} //将 projectModalOpen 状态设为 true
            type={"link"}
          >
            创建项目
          </ButtonNoPadding>
        }
      />
     ......    
    </Container>
  );
};
```

```tsx | pure
// AuthenticatedApp > PageHeader
const PageHeader = (props: { projectButton: JSX.Element }) => {
  return (
    <Header between={true}>
      <HeaderLeft gap={true}>
        <ButtonNoPadding type={"link"} onClick={resetRoute}>
          <SoftwareLogo width={"18rem"} color={"rgb(38, 132, 255)"} />
        </ButtonNoPadding>
        <ProjectPopover {...props} /> // 接收props：projectButton 
        <span>用户</span>
      </HeaderLeft>
      <HeaderRight>
        <User />
      </HeaderRight>
    </Header>
  );
};
```

```tsx | pure
// PageHeader > ProjectPopover
export const ProjectPopover = (props: { projectButton: JSX.Element }) => {
  const { data: projects } = useProjects();
  const pinnedProjects = projects?.filter((project) => project.pin);
  const content = (
    <ContentContainer>
      <Typography.Text type={"secondary"}>收藏项目</Typography.Text>
      <List>
        {pinnedProjects?.map((project) => (
          <List.Item>
            <List.Item.Meta title={project.name} />
          </List.Item>
        ))}
      </List>
      <Divider />
      {props.projectButton} //在此处渲染 projectButton
    </ContentContainer>
  );
  return (
    <Popover placement={"bottom"} content={content}>
      <span>项目</span>
    </Popover>
  );
};
```

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

#### 3.2.1.介绍：

React-Redux 配合 redux 使用，将 redux 定义的 store 数据注入到组件中，可以使组件轻松的拿到全局状态，方便组件间的通信。使 react 组件与 redux 数据中心（store）联系起来，调用 dispatch 函数修改数据状态后，触发 react 更新视图的处理逻辑，包括需要渲染的数据和更新数据的函数。

> 它主要用于在入口处包裹需要用到 Redux 的组件。本质上是将 store 放入 context 里。

#### 3.2.2.最佳实践：

**定义全局 store：**

```ts | pure
import { configureStore } from "@reduxjs/toolkit";
import { projectListSlice } from "screens/projectList/project-list.slice";
import { authSlice } from "./auth.slice";

export const rootReducer = {
  //reducer 的切片
  projectList: projectListSlice.reducer,
  auth: authSlice.reducer,
};

/**
 * 根状态树
 */
export const store = configureStore({
  reducer: rootReducer,
});
export type AppDispatch = typeof store.dispatch;
export type RootState = ReturnType<typeof store.getState>;
```

**为组件定义对应的 reducer 切片：**

```ts | pure
import { createSlice } from "@reduxjs/toolkit";
import { User } from "types/user";
import * as auth from "auth-provider";
import { AuthForm, bootstrapUser } from "context/auth-context";
import { AppDispatch, RootState } from "store";

interface State {
  user: User | null;
}

const initialState: State = {
  user: null,
};
export const authSlice = createSlice({
  name: "auth",
  initialState,
  reducers: {
    setUser(state, action) {
      state.user = action.payload;
    },
  },
});

const { setUser } = authSlice.actions;

export const selectUser = (state: RootState) => state.auth.user;
// dispatch中如果传入的action为函数，则会由 redux-thunk 接管
// dispatch(login(form)) 执行异步操作
export const login = (form: AuthForm) => (dispatch: AppDispatch) =>
  auth.login(form).then((user) => dispatch(setUser(user)));
export const register = (form: AuthForm) => (dispatch: AppDispatch) =>
  auth.register(form).then((user) => dispatch(setUser(user)));
export const logout = () => (dispatch: AppDispatch) =>
  auth.logout().then(() => dispatch(setUser(null)));
export const bootstrap = () => (dispatch: AppDispatch) =>
  bootstrapUser().then((user) => dispatch(setUser(user)));
```

**将 store 注入 context：**

```tsx | pure
import { ReactNode } from "react";
import { QueryClient, QueryClientProvider } from "react-query";
import { AuthProvider } from "context/auth-context";
import { Provider } from "react-redux";
import { store } from "store";
export const AppProviders = ({ children }: { children: ReactNode }) => {
  return (
    <Provider store={store}>
      <QueryClientProvider client={new QueryClient()}>
        <AuthProvider>{children}</AuthProvider>
      </QueryClientProvider>
    </Provider>
  );
};
```

**使用 redux 的权限管理版本：**

```tsx | pure
import { ReactNode, useCallback } from "react";
import * as auth from "auth-provider";
import { User } from "types/user";
import { http } from "utils/http";
import { useMount } from "utils";
import { useAsync } from "utils/use-async";
import { FullPageErrorFallback, FullPageLoading } from "components/lib";
import * as authStore from "store/auth.slice";
import { useDispatch, useSelector } from "react-redux";

export interface AuthForm {
  username: string;
  password: string;
}
/**
 * 获取 user 信息
 * @returns user
 */
export const bootstrapUser = async () => {
  let user = null;
  const token = auth.getToken();
  if (token) {
    const data = await http("me", { token });
    user = data.user;
  }
  return user;
};

export const AuthProvider = ({ children }: { children: ReactNode }) => {
  const { error, isLoading, isIdle, isError, run } = useAsync<User | null>();
  const dispatch: (...args: unknown[]) => Promise<User> = useDispatch();

  useMount(() => {
    run(dispatch(authStore.bootstrap()));
  });
  if (isIdle || isLoading) {
    return <FullPageLoading />;
  }
  if (isError) {
    return <FullPageErrorFallback error={error} />;
  }
  return <div>{children}</div>;
};

export const useAuth = () => {
  const dispatch: (...args: unknown[]) => Promise<User> = useDispatch();
  const user = useSelector(authStore.selectUser);
  const login = useCallback(
    (form: AuthForm) => dispatch(authStore.login(form)),
    [dispatch]
  );
  const register = useCallback(
    (form: AuthForm) => dispatch(authStore.register(form)),
    [dispatch]
  );
  const logout = useCallback(() => dispatch(authStore.logout()), [dispatch]);
  return {
    user,
    login,
    register,
    logout,
  };
};
```

### 3.3.Context：

#### 3.3.1.介绍：

Context 提供了一个无需为每层组件手动添加 props，就能在组件树间进行数据传递的方法。在一个典型的 React 应用中，数据是通过 props 属性自上而下（由父及子）进行传递的，但这种做法对于某些类型的属性而言是极其繁琐的（例如：地区偏好，UI 主题），这些属性是应用程序中许多组件都需要的。Context 提供了一种在组件之间共享此类值的方式，而不必显式地通过组件树逐层传递 props。

> 上下文**没有**自己的 state。它只是数据的管道。你必须将值传递给 `Provider`，然后这个确切的值会被传递给任何知道如何获取它的 `Consumer`（Consumer 和 Provider 绑定的是同一个上下文）。

> Context API 和 Redux 都解决了props存在的 “只要是子组件需要的信息，即使父组件不需要，也必须先传给父组件然后一层层传到子组件” 的问题。
>
> 相比之下，Context API 更适合轻量级的项目，拥有略优于 redux 的性能。缺点是数据安全性不及redux，同时也不能使用redux的中间件，比如thunk/saga，在一些异步的情况需要自己来处理。

#### 3.3.2.最佳实践：

本项目中的 auth-context

**使用 context 的权限管理版本：**

```tsx | pure
import React, { ReactNode } from "react";
import * as auth from "auth-provider";
import { User } from "types/user";
import { http } from "utils/http";
import { useMount } from "utils";
import { useAsync } from "utils/use-async";
import { FullPageErrorFallback, FullPageLoading } from "components/lib";
import { useQueryClient } from "react-query";

interface AuthForm {
  username: string;
  password: string;
}
/**
 * 获取 user 信息
 * @returns user
 */
const bootstrapUser = async () => {
  let user = null;
  const token = auth.getToken();
  if (token) {
    const data = await http("me", { token });
    user = data.user;
  }
  return user;
};
/**
 * 一个上下文对象;
 * 对外暴露给提供者 Provider(通常在组件树中上层的位置)和消费者(上下文内任何获取数据的组件)；
 * 在上下文之内的所有子组件都可以访问这个上下文环境之内的数据，并且不用通过props；
 */
const AuthContext = React.createContext<
  | {
      user: User | null;
      register: (form: AuthForm) => Promise<void>;
      login: (form: AuthForm) => Promise<void>;
      logout: () => Promise<void>;
    }
  | undefined
>(undefined);
AuthContext.displayName = "AuthContext";

/**
 * 提供者，为消费者提供 context 之内的数据
 * @param 需要渲染的子组件
 * @returns 一个具有组件和数据的父容器
 */
export const AuthProvider = ({ children }: { children: ReactNode }) => {
  const {
    data: user,
    error,
    isLoading,
    isIdle,
    isError,
    run,
    setData: setUser,
  } = useAsync<User | null>();
  const queryClient = useQueryClient();

  //为方便使用，将register,login,logout方法封装并暴露出去
  const register = (form: AuthForm) => auth.register(form).then(setUser);
  const login = (form: AuthForm) => auth.login(form).then(setUser);
  const logout = () =>
    auth.logout().then(() => {
      setUser(null);
      queryClient.clear();
    });

  useMount(() => {
    run(bootstrapUser());
  });
  if (isIdle || isLoading) {
    return <FullPageLoading />;
  }
  if (isError) {
    return <FullPageErrorFallback error={error} />;
  }
  return (
    <AuthContext.Provider
      children={children}
      value={{ user, login, register, logout }}
    />
  );
};

/**
 * 获取当前Context 中 Provider 提供的内容
 */
export const useAuth = () => {
  const context = React.useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth必须在AuthProvider中使用");
  }
  return context;
};
```

```tsx | pure
import { ReactNode } from "react";
import { QueryClient, QueryClientProvider } from "react-query";
import { AuthProvider } from "context/auth-context";
export const AppProviders = ({ children }: { children: ReactNode }) => {
  return (
    <QueryClientProvider client={new QueryClient()}>
      <AuthProvider>{children}</AuthProvider>
    </QueryClientProvider>
  );
};
```

