---
title: 设计模式
toc: menu
order: 3
---

## 1.单例模式：

### 2.1.介绍：

单例模式定义了一个对象的创建过程，此对象只有一个单独的实例，并提供一个访问它的全局访问点。

### 2.2.项目中的使用场景:

#### 2.2.1.React-Redux：

 *redux*状态管理器，实质上就是一个*单例模式*。在一个应用中，数据是唯一的，但可以有不同的 UI 使用这份唯一的数据。

**创建全局唯一 store：**

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

**通过 state 全局访问点访问切片状态：**

```ts | pure
export const selectUser = (state: RootState) => state.auth.user;
```

#### 2.2.2.context API：

React v16.3.0 新版本发布的正式 Context 功能 API。类似于 redux，是一个在任意地方都可以访问的单一 *store*。

**使用 React.createContext 在项目中创建唯一实例：**

```tsx | pure
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
```

**使用 useAuth hook 作为 context 全局访问点。通过调用 useAuth Hook 即可在此 context 下的任意处访问：**

```tsx | pure
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

## 2.控制反转：

### 2.1.介绍：

**依赖倒置**（Dependency inversion principle，缩写为 DIP）是面向对象六大基本原则之一。它是指一种特定的解耦形式，使得高层次的模块不依赖于低层次的模块的实现细节，依赖关系被颠倒（反转），从而使得低层次模块依赖于高层次模块的需求抽象。

该原则规定：

- 高层次的模块不应该依赖于低层次的模块，两者都应该依赖于抽象接口。
- 抽象接口不应该依赖于具体实现。而具体实现则应该依赖于抽象接口。



**控制反转**（Inversion of Control，缩写为 IOC）是面向对象编程中的一种设计原则，用来降低计算机代码之间的耦合度。**是实现依赖倒置原则的一种代码设计思路**。其中最常见的方式叫做依赖注入，还有一种方式叫依赖查找。

### 2.2.项目中的使用场景：

#### 2.2.1.React 的[组件组合（component composition）](https://zh-hans.reactjs.org/docs/composition-vs-inheritance.html)：

组件可以通过 JSX 嵌套，将任意组件作为子组件传递给它们。

**以本项目中的 AuthenticatedApp 组件为例：**

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

 在 AuthenticatedApp 中，projectButton 组件中的所有内容都会作为一个 prop 传递给 PageHeader 组件。而PageHeader 组件则将其传递给 ProjectPopover。因为 ProjectPopover 将 `{props.projectButton} `渲染，被传递的子组件 projectButton 最终会出现在输出结果中。

由于 ProjectPopover 组件需要使用 AuthenticatedApp 组件中的方法 setProjectModalOpen，传统模式下必须通过prop drilling 的方式把 setProjectModalOpen 从源组件传递到深层嵌套组件，在组件需求变化同时也会增加组件的复杂度，为避免此情况让 PageHeader 以组件组合方式接收了另一个组件 projectButton，并且将其向下传递，这使得实际渲染 projectButton 的子组件 ProjectPopover 无需关心 projectButton 组件的实现细节，而控制 projectButton 如何展示的逻辑实际写在 AuthenticatedApp 中。

> 这种对组件的*控制反转*减少了在应用中要传递的 props 数量，这在很多场景下会使得代码更加干净，使得对根组件有更多的把控。但是，这并不适用于每一个场景：这种行为将逻辑提升到组件树的更高层次来处理，会使得这些高层组件变得更复杂，并且会强行将低层组件适应这样的形式，props drilling 的问题也没有得到解决。

## 3.Context 模式：

### 3.1.介绍：

React 有 [*context*](https://zh-hans.reactjs.org/docs/context.html) 的概念。每个 React 组件都可以访问 *context* 。它有些类似于 [事件总线](https://github.com/krasimir/EventBus) ，但是为数据而生。可以把它想象成在任意地方都可以访问的单一 *store* 。

> React 的 context 是提供者模式的具体实现。

### 3.2.项目中的使用场景：

#### 3.2.1.context API:

React v16.3.0 新版本发布的正式 Context 功能 API。

**用 AuthProvider 封装处理逻辑并暴露给子组件：**

```tsx | pure
/**
 * 提供者，为消费者提供 context 之内的数据
 * @param 需要渲染的子组件
 * @returns 一个具有组件接口和数据的父容器
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

  //为方便使用，将register,login,logout 方法封装并暴露出去
  const register = (form: AuthForm) => auth.register(form).then(setUser);
  const login = (form: AuthForm) => auth.login(form).then(setUser);
  const logout = () => auth.logout().then(() => setUser(null));

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
```

**将`AppProviders` 挂载至 ReactDOM：**

`src/context/index.tsx`

```tsx | pure
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

`src/index.tsx`

```tsx | pure
loadServer(() =>
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

此后`<App>`组件内的任意组件都可以通过 `useAuth` 获取 *context* 的状态。



