---
title: 设计模式
toc: menu
order: 3
---

## 1.装饰器模式：

### 1.1.介绍：

允许向一个现有的对象添加新的功能，同时又不改变其结构，动态的给对象添加一些额外的属性或行为。相比于使用继承，装饰器模式更加灵活。从本质上看装饰器模式是一个包装模式（(Wrapper Pattern），它是通过封装其他对象达到设计的目的。

### 1.2.项目中的使用场景：

#### 1.2.1.React 的高阶组件 （HOC）:

很长一段时期内，高阶组件都是增强和组合 React 元素的最流行的方式。它们看上去与 [装饰器模式](http://robdodson.me/javascript-design-patterns-decorator/) 十分相似，因为它是对组件的包装与增强。

高阶组件通常是函数，它接收原始组件并返回原始组件的增强/填充版本。

#### 1.2.2.react-redux 中的 connect：



## 2.单例模式：

### 2.1.介绍：

单例模式定义了一个对象的创建过程，此对象只有一个单独的实例，并提供一个访问它的全局访问点。

### 2.2.项目中的使用场景:

#### 2.2.1.Redux：

#### 2.2.2.React context：

## 3.控制反转：

### 3.1.介绍：

**依赖倒置**（Dependency inversion principle，缩写为 DIP）是面向对象六大基本原则之一。它是指一种特定的解耦形式，使得高层次的模块不依赖于低层次的模块的实现细节，依赖关系被颠倒（反转），从而使得低层次模块依赖于高层次模块的需求抽象。

该原则规定：

- 高层次的模块不应该依赖于低层次的模块，两者都应该依赖于抽象接口。
- 抽象接口不应该依赖于具体实现。而具体实现则应该依赖于抽象接口。



**控制反转**（Inversion of Control，缩写为 IOC）是面向对象编程中的一种设计原则，用来降低计算机代码之间的耦合度。**是实现依赖倒置原则的一种代码设计思路**。其中最常见的方式叫做依赖注入，还有一种方式叫依赖查找。

### 3.2.项目中的使用场景：

#### 3.2.1.React 的[组件组合（component composition）](https://zh-hans.reactjs.org/docs/composition-vs-inheritance.html)：

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



由于 ProjectPopover 组件需要使用另一个嵌套组件提供的数据，于是 PageHeader 接收了另一个组件 projectButton，并且将其向下层层传递，这使得实际渲染 projectButton 的子组件 ProjectPopover 无需关心 projectButton 组件的实现细节，而控制 projectButton 如何展示的逻辑实际写在AuthenticatedApp 中。



这种对组件的*控制反转*减少了在应用中要传递的 props 数量，这在很多场景下会使得代码更加干净，使得对根组件有更多的把控。但是，这并不适用于每一个场景：这种行为将逻辑提升到组件树的更高层次来处理，会使得这些高层组件变得更复杂，并且会强行将低层组件适应这样的形式，props drilling 的问题也没有得到解决。

#### 3.2.2.React context：

React 有 [*context*](https://zh-hans.reactjs.org/docs/context.html) 的概念。每个 React 组件都可以访问 *context* 。它有些类似于 [事件总线](https://github.com/krasimir/EventBus) ，但是为数据而生。可以把它想象成在任意地方都可以访问的单一 *store* 。

`src/context/auth-context.tsx`:

```js
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

## 4.观察者模式：

### 4.1.介绍：

在观察者模式中，观察者需要直接订阅目标事件；在目标发出内容改变的事件后，直接接收事件并作出响应。很多库都有该模式的实现，比如vue、redux等。

