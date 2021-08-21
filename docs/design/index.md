---
title: 模块设计
toc: content
order: 1
nav:
  path: /design
  title: 项目设计
  order: 1
---

## 1.项目目录结构

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

SPA 应用入口。以下文件涉及 SPA 应用的渲染。

### 2.1.应用入口

#### 2.1.1.`src/index.tsx`:

最顶部入口，应用被挂载至虚拟DOM，经 webpack 打包处理依赖后注入 `index.html`

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

loadServer(() => //读取 mockServiceWorker 服务器，启动 Mock 服务
  ReactDOM.render(
    <React.StrictMode>
      <AppProviders> //context provider
        <DevTools /> //启用开发者工具
        <App />     // App 组件
      </AppProviders>
    </React.StrictMode>,
    document.getElementById("root")
  )
);
```

#### 2.1.2.`src/App.tsx`:

根组件入口，依据权限动态切换认证页面或未认证页面。

```tsx | pure
import React from "react";
import "./App.css";
import { useAuth } from "context/auth-context";
import ErrorBoundary from "antd/lib/alert/ErrorBoundary";
import { FullPageLoading } from "components/lib";

const AuthenticatedApp = React.lazy(() => import("authenticated-app"));
const UnauthenticatedApp = React.lazy(() => import("unauthenticated-app"));
function App() {
  const { user } = useAuth();
  return (
    <div className="App">
      <ErrorBoundary>
        <React.Suspense fallback={<FullPageLoading />}>
          {user ? <AuthenticatedApp /> : <UnauthenticatedApp />}
        </React.Suspense>
      </ErrorBoundary>
    </div>
  );
}
export default App;
```

### 2.2.context

#### 2.2.1.`src/context/auth-context.tsx`:

注册全局唯一 context 实例，定义 context provider。

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
 * 获取当前 Context 中 Provider 提供的内容
 */
export const useAuth = () => {
  const context = React.useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth必须在AuthProvider中使用");
  }
  return context;
};
```

#### 2.2.2.`src/context/index.tsx`

AuthProvider 入口，包装 AuthProvider 导出为 AppProviders，并为 AppProviders 提供 react-query client。

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

#### 2.2.3.`src/auth-provider.ts`:

 AuthProvider 中请求方法的具体实现。

```ts | pure
import { User } from "types/user";
const apiUrl = process.env.REACT_APP_API_URL;
const localStorageKey = "__auth_provider_token__";

export const getToken = () => window.localStorage.getItem(localStorageKey);

export const handleUserResponse = ({ user }: { user: User }) => {
  window.localStorage.setItem(localStorageKey, user.token || "");
  return user;
};

export const login = (data: { username: string; password: string }) => {
  return fetch(`${apiUrl}/login`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify(data),
  }).then(async (response: Response) => {
    if (response.ok) {
      return handleUserResponse(await response.json());
    } else {
      return Promise.reject(await response.json());
    }
  });
};

export const register = (data: { username: string; password: string }) => {
  return fetch(`${apiUrl}/register`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify(data),
  }).then(async (response: Response) => {
    if (response.ok) {
      return handleUserResponse(await response.json());
    } else {
      return Promise.reject(await response.json());
    }
  });
};

export const logout = async () =>
  window.localStorage.removeItem(localStorageKey);

```

## 3.页面

在 react 应用中，每个页面都是是一系列组件的集合。以下组件直接构成页面的展示部分，故划分至页面层。

### 3.1.认证/未认证页面

#### 3.1.1.`src/authenticated-app.tsx`:

认证页面，根据当前路由动态引入页面组件。

```tsx | pure
import styled from "@emotion/styled";
import { ButtonNoPadding, Row } from "components/lib";
import { useAuth } from "context/auth-context";
import { ReactComponent as SoftwareLogo } from "assets/software-logo.svg";
import React from "react";
import { Avatar, Button, Dropdown, Menu } from "antd";
import { resetRoute } from "utils";
import { Navigate, Route, Routes } from "react-router";
import { BrowserRouter as Router } from "react-router-dom";
import { ProjectModal } from "screens/projectList/project-modal";
import { ProjectPopover } from "components/project-popover";
import cloud from "assets/cloud.svg";
import dayjs from "dayjs";
import { keyframes } from "@emotion/react";
import { UserPopover } from "components/user-popover";
import { CreateUser } from "components/create-user";
const ProjectScreen = React.lazy(() => import("screens/project"));
const ProjectListScreen = React.lazy(() => import("screens/projectList"));

const getTime = () => {
  const hour = dayjs().get("hour");
  return hour > 0 && hour <= 10
    ? { text: "🌅早上好", mainColor: "#FFCC99", subColor: "#FFFFFF" }
    : hour > 10 && hour <= 13
    ? { text: "🌞中午好", mainColor: "#FF6600", subColor: "#FFFFCC" }
    : hour > 13 && hour <= 17
    ? { text: "⛅下午好", mainColor: "#66CCCC", subColor: "#CCFF99" }
    : { text: "🌙晚上好", mainColor: "#6666CC", subColor: "#FF9999" };
};
const now = getTime();

const AuthenticatedApp = () => {
  return (
    <Container>
      <Router>
        <PageHeader />
        <Background>
          <Main>
            <Routes>
              <Route path={"/projects"} element={<ProjectListScreen />} />
              <Route
                path={"/projects/:projectId/*"}
                element={<ProjectScreen />}
              />
              <Navigate to={"/projects"} />
            </Routes>
          </Main>
        </Background>
        <ProjectModal />
      </Router>
    </Container>
  );
};

const PageHeader = () => {
  return (
    <Header between={true}>
      <HeaderLeft gap={true}>
        <ButtonNoPadding type={"link"} onClick={resetRoute}>
          <SoftwareLogo width={"18rem"} color={"rgb(38, 132, 255)"} />
        </ButtonNoPadding>
        <ProjectPopover />
        <UserPopover />
        <CreateUser />
      </HeaderLeft>
      <HeaderRight>
        <User />
      </HeaderRight>
    </Header>
  );
};

const User = () => {
  const { logout, user } = useAuth();

  return (
    <div
      style={{
        display: "flex",
        alignItems: "center",
        justifyContent: "center",
      }}
    >
      <span style={{ color: `${now.mainColor}` }}>{now.text}~</span>
      <Dropdown
        overlay={
          <Menu>
            <Menu.Item key={"logout"}>
              <Button type={"link"} onClick={logout}>
                登出
              </Button>
            </Menu.Item>
          </Menu>
        }
      >
        <div>
          <Button type={"link"} onClick={(e) => e.preventDefault()}>
            <SpinAvatar
              style={{
                backgroundColor: `${now.mainColor}`,
                color: `${now.subColor}`,
              }}
              size={"large"}
            >
              {user?.name}
            </SpinAvatar>
          </Button>
        </div>
      </Dropdown>
    </div>
  );
};

const Background = styled.div`
  width: 100%;
  height: calc(100vh - 6rem);
  display: flex;
  background-attachment: fixed;
  background-position: center;
  background-size: cover;
  background-image: url(${cloud});
`;

const Container = styled.div`
  display: grid;
  grid-template-rows: 6rem 1fr;
  height: 100vh;
`;

const Header = styled(Row)`
  padding: 2.2rem;
  box-shadow: 0 0 5px 0 rgba(7, 7, 7, 0.2);
  z-index: 1;
`;

const HeaderLeft = styled(Row)``;

const HeaderRight = styled.div``;

const Main = styled.main`
  width: 100%;
  display: flex;
  overflow: hidden;
  opacity: 0.85;
`;

const rotate = keyframes`
  from {
    transform: rotateY(0deg);
  }
  to {
    transform: rotateY(360deg);
  }
`;

const SpinAvatar = styled(Avatar)`
  background-color: #3ba5ec;
  color: #eaeeec;
  &:hover {
    animation: ${rotate} 1s linear;
  }
`;

export default AuthenticatedApp;

```

#### 3.1.2.`src/unauthenticated-app/index.tsx`:

未认证页面入口，主要由登录注册页面构成。

```tsx | pure
import { Card, Button, Divider } from "antd";
import { useState } from "react";
import { LoginScreen } from "./login";
import { RegisterScreen } from "./register";
import styled from "@emotion/styled";
import logo from "assets/logo.svg";
import left from "assets/left.svg";
import right from "assets/right.svg";
import { useDocumentTitle } from "utils";
import { ErrorBox } from "components/lib";

const UnauthenticatedApp = () => {
  const [isRegister, setIsRegister] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  useDocumentTitle("请登录注册以继续", false);
  return (
    <Container>
      <Header />
      <Background />
      <ShadowCard>
        <Title>{isRegister ? "请注册" : "请登录"}</Title>
        <ErrorBox error={error} />
        {isRegister ? (
          <RegisterScreen onError={setError} />
        ) : (
          <LoginScreen onError={setError} />
        )}
        <Divider />
        <Button type={"link"} onClick={() => setIsRegister(!isRegister)}>
          {isRegister ? "已经有账号？直接登录" : "还没有账号？立即注册"}
        </Button>
      </ShadowCard>
    </Container>
  );
};
export const LongButton = styled(Button)`
  width: 100%;
`;
const Title = styled.h2`
  margin-bottom: 2.4rem;
  color: rgb(59, 62, 218);
  text-shadow: rgba(131, 112, 219, 0.349) 0 0 10px;
`;

const Background = styled.div`
  position: absolute;
  width: 100%;
  height: 100%;
  background-repeat: no-repeat;
  background-attachment: fixed;
  background-position: left bottom, right, bottom;
  background-size: calc((100vw - 40rem) / 2 - 3.2rem),
    calc((100vw - 40rem) / 2 - 3.2rem), cover;
  background-image: url(${left}), url(${right});
`;

const ShadowCard = styled(Card)`
  width: 40rem;
  min-height: 56rem;
  padding: 3.2rem 4rem;
  border-radius: 0.3rem;
  box-sizing: border-box;
  box-shadow: rgba(0, 0, 0, 0.1) 0 0 10px;
  text-align: center;
`;

const Header = styled.header`
  background: url(${logo}) no-repeat center;
  padding: 5rem 0;
  background-size: 8rem;
  width: 100%;
`;

const Container = styled.div`
  display: flex;
  flex-direction: column;
  align-items: center;
  min-height: 100vh;
`;

export default UnauthenticatedApp;

```

#### 3.1.3.`src/unauthenticated-app/login.tsx`:

登录页面。

```tsx | pure
import { useAuth } from "context/auth-context";
import { Form, Input } from "antd";
import { LongButton } from "unauthenticated-app";
import { useAsync } from "utils/use-async";

export const LoginScreen = ({
  onError,
}: {
  onError: (error: Error) => void;
}) => {
  const { login } = useAuth();
  const { run, isLoading } = useAsync(undefined, { throwOnError: true });

  const handleSubmit = async (values: {
    username: string;
    password: string;
  }) => {
    try {
      await run(login(values));
    } catch (e) {
      onError(e);
    }
  };
  return (
    <Form onFinish={handleSubmit}>
      <Form.Item
        name={"username"}
        rules={[{ required: true, message: "请输入用户名" }]}
      >
        <Input placeholder={"用户名"} type="text" id={"username"} />
      </Form.Item>
      <Form.Item
        name={"password"}
        rules={[{ required: true, message: "请输入密码" }]}
      >
        <Input placeholder={"密码"} type="password" id={"password"} />
      </Form.Item>
      <Form.Item>
        <LongButton loading={isLoading} htmlType={"submit"} type={"primary"}>
          登录
        </LongButton>
      </Form.Item>
    </Form>
  );
};

```

#### 3.1.4.`src/unauthenticated-app/register.tsx`:

注册页面。

```tsx | pure
import { useAuth } from "context/auth-context";
import { Form, Input } from "antd";
import { LongButton } from "unauthenticated-app";
import { useAsync } from "utils/use-async";

export const RegisterScreen = ({
  onError,
}: {
  onError: (error: Error) => void;
}) => {
  const { register } = useAuth();
  const { run, isLoading } = useAsync(undefined, { throwOnError: true });

  const handleSubmit = async ({
    cpassword,
    ...values
  }: {
    username: string;
    password: string;
    cpassword: string;
  }) => {
    if (cpassword !== values.password) {
      onError(new Error("请确认两次输入的密码相同"));
      return;
    }
    try {
      await run(register(values));
    } catch (e) {
      onError(e);
    }
  };
  return (
    <Form onFinish={handleSubmit}>
      <Form.Item
        name={"username"}
        rules={[{ required: true, message: "请输入用户名" }]}
      >
        <Input placeholder={"用户名"} type="text" id={"username"} />
      </Form.Item>
      <Form.Item
        name={"password"}
        rules={[{ required: true, message: "请输入密码" }]}
      >
        <Input placeholder={"密码"} type="password" id={"password"} />
      </Form.Item>
      <Form.Item
        name={"cpassword"}
        rules={[{ required: true, message: "请确认密码" }]}
      >
        <Input placeholder={"确认密码"} type="password" id={"cpassword"} />
      </Form.Item>
      <Form.Item>
        <LongButton loading={isLoading} htmlType={"submit"} type={"primary"}>
          注册
        </LongButton>
      </Form.Item>
    </Form>
  );
};
```

### 3.2.项目列表页面

#### 3.2.1.`src/screens/projectList/index.tsx`:

项目列表页面入口，包含 `List`，`SearchPanel`  组件。

```tsx | pure
import { useDebounce, useDocumentTitle } from "utils/index";
import { List } from "./list";
import { SearchPanel } from "./search-panel";
import { useProjects } from "utils/project";
import { useUsers } from "utils/user";
import { useProjectModal, useProjectsSearchParams } from "./util";
import {
  ButtonNoPadding,
  ErrorBox,
  Row,
  ScreenContainer,
} from "components/lib";
import React from "react";

const ProjectListScreen = React.memo(() => {
  useDocumentTitle("项目列表", false);

  const [param, setParam] = useProjectsSearchParams();
  // 只对 name 参数 debounce
  const debouncedName = useDebounce({ name: param.name }, 2000);

  const {
    isLoading,
    error,
    data: list,
  } = useProjects({ ...param, ...debouncedName });

  const { data: users } = useUsers();

  const { open } = useProjectModal();

  return (
    <ScreenContainer>
      <Row between={true}>
        <h1>项目列表</h1>
        <ButtonNoPadding
          type={"primary"}
          style={{ opacity: "0.8", borderRadius: "15px" }}
          size={"large"}
          onClick={open}
        >
          创建项目
        </ButtonNoPadding>
      </Row>

      <SearchPanel users={users || []} param={param} setParam={setParam} />
      <ErrorBox error={error} />
      <List
        loading={isLoading}
        dataSource={list || []}
        users={users || []}
      ></List>
    </ScreenContainer>
  );
});

export default ProjectListScreen;
```

#### 3.2.2.`src/screens/projectList/list.tsx`:

项目列表组件，接收数据以 Table 形式显示。

```tsx | pure
import { Project } from "types/project";
import { User } from "types/user";
import { Dropdown, Menu, Modal, Table, TableProps } from "antd";
import dayjs from "dayjs";
import { Link } from "react-router-dom";
import { Pin } from "components/pin";
import { useDeleteProject, useEditProject } from "utils/project";
import { ButtonNoPadding } from "components/lib";
import { useProjectModal, useProjectsQueryKey } from "./util";
interface ListProp extends TableProps<Project> {
  users: User[];
  refresh?: () => void;
}
export const List = ({ users, ...props }: ListProp) => {
  const { mutate: editProject } = useEditProject(useProjectsQueryKey());

  //柯里化函数形式
  const pinProject = (id: number) => (pin: boolean) => editProject({ id, pin });

  return (
    <Table
      pagination={false}
      columns={[
        {
          title: <Pin checked={true} disabled={true} />,
          render(value, project) {
            return (
              <Pin
                checked={project.pin}
                onCheckedChange={pinProject(project.id)}
              />
            );
          },
        },
        {
          title: "名称",
          sorter: (a, b) => a.name.localeCompare(b.name),
          render(value, project) {
            return <Link to={String(project.id)}>{project.name}</Link>;
          },
        },
        {
          title: "部门",
          dataIndex: "organization",
        },
        {
          title: "负责人",
          render(value, project) {
            return (
              <span>
                {users.find((user) => user.id === project.personId)?.name ||
                  "未知"}
              </span>
            );
          },
        },
        {
          title: "创建时间",
          render(value, project) {
            return (
              <span>
                {project.created
                  ? dayjs(project.created).format("YYYY-MM-DD")
                  : "无"}
              </span>
            );
          },
        },
        {
          render(value, project) {
            return <More project={project} />;
          },
        },
      ]}
      {...props}
    ></Table>
  );
};

const More = ({ project }: { project: Project }) => {
  const { startEdit } = useProjectModal();
  const editProject = (id: number) => () => startEdit(id);
  const { mutate: deleteProject } = useDeleteProject(useProjectsQueryKey());
  const confirmDeleteProject = (id: number) => {
    Modal.confirm({
      title: "🙃你确定要删除这个项目🐴？",
      content: "😡别怪我没提醒你",
      okText: "😶确定",
      cancelText: "😅算了算了",
      onOk() {
        deleteProject({ id });
      },
    });
  };

  return (
    <Dropdown
      overlay={
        <Menu>
          <Menu.Item key={"edit"}>
            <ButtonNoPadding onClick={editProject(project.id)} type={"link"}>
              编辑项目
            </ButtonNoPadding>
          </Menu.Item>
          <Menu.Item key={"delete"}>
            <ButtonNoPadding
              onClick={() => confirmDeleteProject(project.id)}
              type={"link"}
            >
              删除项目
            </ButtonNoPadding>
          </Menu.Item>
        </Menu>
      }
    >
      <ButtonNoPadding type={"link"}>...</ButtonNoPadding>
    </Dropdown>
  );
};
```

3.2.3.`src/screens/projectList/search-panel.tsx`:

项目搜索组件，包含一个 Input 输入框和 UserSelect 下拉选择器组件。

```tsx | pure
import { Form, Input } from "antd";
import { UserSelect } from "components/user-select";
import { Project } from "types/project";
import { User } from "types/user";
interface SearchPanelProps {
  users: User[];
  param: Partial<Pick<Project, "name" | "personId">>;
  setParam: (param: SearchPanelProps["param"]) => void;
}

export const SearchPanel = ({ param, setParam }: SearchPanelProps) => {
  return (
    <Form style={{ marginBottom: "2rem" }} layout={"inline"}>
      <Form.Item>
        <Input
          placeholder={"项目名"}
          type="text"
          value={param.name}
          onChange={(evt) =>
            setParam({
              ...param,
              name: evt.target.value,
            })
          }
        ></Input>
      </Form.Item>
      <Form.Item>
        <UserSelect
          defaultOptionName={"负责人"}
          value={param.personId}
          onChange={(value) => {
            setParam({
              ...param,
              personId: value,
            });
          }}
        ></UserSelect>
      </Form.Item>
    </Form>
  );
};
```

#### 3.2.3.`src/screens/projectList/project-modal.tsx`:

项目模态框。负责处理创建项目和编辑项目。

```tsx | pure
import styled from "@emotion/styled";
import { Button, Drawer, Form, Input, Spin } from "antd";
import { useForm } from "antd/lib/form/Form";
import { ErrorBox } from "components/lib";
import { UserSelect } from "components/user-select";
import { useEffect } from "react";
import { useAddProject, useEditProject } from "utils/project";
import { useProjectModal, useProjectsQueryKey } from "./util";
import access from "assets/access.svg";

export const ProjectModal = () => {
  const { projectModalOpen, close, editingProject, isLoading } =
    useProjectModal();
  const useMutateProject = editingProject ? useEditProject : useAddProject;

  const {
    mutateAsync,
    error,
    isLoading: mutateLoading,
  } = useMutateProject(useProjectsQueryKey());
  const [form] = useForm();
  const onFinish = (values: any) => {
    mutateAsync({ ...editingProject, ...values }).then(() => {
      form.resetFields();
      closeModal();
    });
  };
  const closeModal = () => {
    form.resetFields();
    close();
  };

  const title = editingProject ? "编辑项目" : "创建项目";

  useEffect(() => {
    form.setFieldsValue(editingProject);
  }, [form, editingProject]);

  return (
    <Drawer
      forceRender={true}
      onClose={closeModal}
      visible={projectModalOpen}
      width={"100%"}
    >
      <Background>
        <Container>
          {isLoading ? (
            <Spin size={"large"} />
          ) : (
            <>
              <h1>{title}</h1>
              <ErrorBox error={error} />
              <Form
                form={form}
                layout={"vertical"}
                style={{ width: "40rem" }}
                onFinish={onFinish}
              >
                <Form.Item
                  label={"名称"}
                  name={"name"}
                  rules={[
                    {
                      required: true,
                      message: "请输入项目名称",
                    },
                  ]}
                >
                  <Input placeholder={"请输入项目名称"} />
                </Form.Item>
                <Form.Item
                  label={"部门"}
                  name={"organization"}
                  rules={[
                    {
                      required: true,
                      message: "请输入部门名称",
                    },
                  ]}
                >
                  <Input placeholder={"请输入部门名称"} />
                </Form.Item>
                <Form.Item label={"负责人"} name={"personId"}>
                  <UserSelect defaultOptionName={"负责人"} />
                </Form.Item>
                <Form.Item style={{ textAlign: "center" }}>
                  <Button
                    loading={mutateLoading}
                    type={"primary"}
                    htmlType={"submit"}
                  >
                    提交
                  </Button>
                </Form.Item>
              </Form>
            </>
          )}
        </Container>
      </Background>
    </Drawer>
  );
};

const Container = styled.div`
  height: 80vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
`;
const Background = styled.div`
  width: 100%;
  height: 100%;
  background-attachment: fixed;
  background-position: left 32% bottom 60%;
  background-size: contain;
  background-repeat: no-repeat;
  opacity: 0.8;
  background-image: url(${access});
`;
```

### 3.3.项目详情页面

#### 3.3.1.`src/screens/project/index.tsx`:

项目详情页面入口，依据当前路由展示看板和任务组页面。默认展示看板页面。

```tsx | pure
import styled from "@emotion/styled";
import { Menu } from "antd";
import React from "react";
import { Link, Navigate, Route, Routes, useLocation } from "react-router-dom";
import { EpicScreen } from "screens/epic";
import { KanbanScreen } from "screens/kanban";

const useRouteType = () => {
  const units = useLocation().pathname.split("/");
  return units[units.length - 1];
};

const ProjectScreen = React.memo(() => {
  const routeType = useRouteType();
  return (
    <Container>
      <Aside>
        <Menu mode={"inline"} selectedKeys={[routeType]}>
          <Menu.Item key={"kanban"}>
            <Link to={"kanban"}>看板</Link>
          </Menu.Item>
          <Menu.Item key={"epic"}>
            <Link to={"epic"}>任务组</Link>
          </Menu.Item>
        </Menu>
      </Aside>
      <Main>
        <Routes>
          <Route path={"/kanban"} element={<KanbanScreen />} />
          <Route path={"/epic"} element={<EpicScreen />} />
          <Navigate to={window.location.pathname + "/kanban"} replace={true} />
        </Routes>
      </Main>
    </Container>
  );
});

const Aside = styled.aside`
  border-color: rgb(244, 245, 247);
  display: flex;
`;

const Main = styled.div`
  box-shadow: -5px 0 5px rgba(0, 0, 0, 0.1);
  display: flex;
  overflow: hidden;
`;

const Container = styled.div`
  display: grid;
  grid-template-columns: 16rem 1fr;
`;

export default ProjectScreen;
```

### 3.4.看板页面

#### 3.4.1.`src/screens/kanban/index.tsx`:

看板页面入口，包含一个可拖拽的 `KanbanColumn`组件以及`SearchPanel` ，`CreateKanban` 组件。

```tsx | pure
import styled from "@emotion/styled";
import { Spin } from "antd";
import { DragDropContext, DropResult } from "react-beautiful-dnd";
import { ScreenContainer } from "components/lib";
import { useDebounce, useDocumentTitle } from "utils";
import { useKanbans, useReorderKanban } from "utils/kanban";
import { useReorderTask, useTasks } from "utils/task";
import { CreateKanban } from "./create-kanban";
import { KanbanColumn } from "./kanban-column";
import { SearchPanel } from "./search-panel";
import { TaskModal } from "./task-modal";
import {
  useKanbansQueryKey,
  useKanbansSearchParams,
  useProjectInUrl,
  useTasksQueryKey,
  useTasksSearchParams,
} from "./util";
import { Drag, Drop, DropChild } from "components/drag-and-drop";
import { useCallback } from "react";

export const useDragEnd = () => {     // 拖拽结束后执行此回调
  const { data: kanbans } = useKanbans(useKanbansSearchParams());
  const { data: allTasks = [] } = useTasks(useTasksSearchParams());
  const { mutate: reorderKanban } = useReorderKanban(useKanbansQueryKey());
  const { mutate: reorderTask } = useReorderTask(useTasksQueryKey());
  return useCallback(
    ({ source, destination, type }: DropResult) => {
      if (!destination) {
        return;
      }
      //看板重排序
      if (type === "COLUMN") {
        const fromId = kanbans?.[source.index].id;
        const toId = kanbans?.[destination.index].id;

        if (!fromId || !toId || fromId === toId) {
          return;
        }
        const type = destination.index > source.index ? "after" : "before";
        reorderKanban({ fromId, referenceId: toId, type });
      }
      //任务重排序
      if (type === "ROW") {
        const fromKanbanId = +source.droppableId;
        const toKanbanId = +destination.droppableId;
        //不予许在同一看板下交换任务顺序
        if (fromKanbanId === toKanbanId) {
          return;
        }

        const fromTask = allTasks.filter(
          (task) => task.kanbanId === fromKanbanId
        )[source.index];
        const toTask = allTasks.filter((task) => task.kanbanId === toKanbanId)[
          destination.index
        ];

        if (fromTask?.id === toTask?.id) {
          return;
        }
        reorderTask({
          fromId: fromTask?.id,
          referenceId: toTask?.id,
          fromKanbanId,
          toKanbanId,
          type:
            fromKanbanId === toKanbanId && destination.index > source.index
              ? "after"
              : "before",
        });
      }
    },
    [kanbans, reorderKanban, allTasks, reorderTask]
  );
};

export const KanbanScreen = () => {
  useDocumentTitle("看板列表");

  const { data: kanbans, isLoading: kanbanIsLoading } = useKanbans(
    useKanbansSearchParams()
  );
  const { data: currentProject } = useProjectInUrl();

  const param = useTasksSearchParams();
  //只对 name参数 debounce
  const debouncedName = useDebounce({ name: param.name }, 2000);
  const { data: allTasks, isLoading: taskIsLoading } = useTasks({
    ...param,
    ...debouncedName,
  });

  const isLoading = taskIsLoading || kanbanIsLoading;

  const dragEnd = useDragEnd();
  return (
    <DragDropContext onDragEnd={dragEnd}>
      <ScreenContainer>
        <h1>{currentProject?.name}看板</h1>
        <SearchPanel />
        {isLoading ? (
          <Spin size={"large"} />
        ) : (
          <ColumnsContainer>
            <Drop
              type={"COLUMN"}
              direction={"horizontal"}
              droppableId={"kanban"}
            >
              <DropChild style={{ display: "flex" }}>
                {kanbans?.map((kanban, index) => (
                  <Drag
                    key={kanban.id}
                    draggableId={"kanban" + kanban.id}
                    index={index}
                  >
                    <KanbanColumn
                      kanban={kanban}
                      allTasks={allTasks}
                      key={kanban.id}
                    />
                  </Drag>
                ))}
              </DropChild>
            </Drop>
            <CreateKanban />
          </ColumnsContainer>
        )}
        <TaskModal />
      </ScreenContainer>
    </DragDropContext>
  );
};

export const ColumnsContainer = styled.div`
  display: flex;
  overflow-x: scroll;
  overflow-y: hidden;
  flex: 1;
`;
```

#### 3.4.2.`src/screens/kanban/kanban-column.tsx`:

看板列表组件，以列方式布局。每一列由可拖拽的任务列表及 `CreateTask `组件构成。

```tsx | pure
import { Kanban } from "types/kanban";
import { useTaskTypes } from "utils/task-type";
import { useKanbansQueryKey, useTaskModal, useTasksSearchParams } from "./util";
import taskIcon from "assets/task.svg";
import bugIcon from "assets/bug.svg";
import styled from "@emotion/styled";
import { Button, Card, Dropdown, Menu, Modal, Tooltip } from "antd";
import { CreateTask } from "./create-task";
import { Task } from "types/task";
import { Mark } from "components/mark";
import { useDeleteKanban } from "utils/kanban";
import { Row } from "components/lib";
import React from "react";
import { Drag, Drop } from "components/drag-and-drop";
import { useUsers } from "utils/user";
import { useEpics } from "utils/epic";
import { useEpicsSearchParams } from "screens/epic/util";

interface KanbanColumnProps {
  kanban: Kanban;
  allTasks: Task[] | undefined;
}

const TaskTypeIcon = ({ id }: { id: number }) => {
  const { data: taskTypes } = useTaskTypes();
  const name = taskTypes?.find((taskType) => taskType.id === id)?.name;
  if (!name) {
    return null;
  }
  return <img src={name === "task" ? taskIcon : bugIcon} alt={"taskType"} />;
};

const TaskCard = ({ task }: { task: Task }) => {
  const { startEdit } = useTaskModal();
  const { name: keyword } = useTasksSearchParams();
  const { data: users } = useUsers();
  const { data: epics } = useEpics(useEpicsSearchParams());

  return (
    <Tooltip
      mouseEnterDelay={0.8}
      placement="topLeft"
      title="点击修改，拖动进行布局"
      color={"cyan"}
      arrowPointAtCenter
    >
      <div>
        <ShadowCard onClick={() => startEdit(task.id)} key={task.id}>
          <Mark name={`任务：📝${task.name}`} keyword={keyword} />
          <div style={{ fontFamily: "Tahoma" }}>
            type：
            <TaskTypeIcon id={task.typeId} />
            <div>
              经办人：🤵
              {users?.find((user) => task?.processorId === user.id)?.name ||
                "未知"}
            </div>
            <div>
              任务组：🗃️
              {epics?.find((epic) => task.epicId === epic.id)?.name || "未知"}
            </div>
          </div>
        </ShadowCard>
      </div>
    </Tooltip>
  );
};

const More = ({ kanban }: { kanban: Kanban }) => {
  const { mutate: deleteKanban } = useDeleteKanban(useKanbansQueryKey());
  const confirmDeleteKanban = () => {
    Modal.confirm({
      title: "🙃你确定要删除这个看板🐴？",
      content: "😡别怪我没提醒你",
      okText: "😶确定",
      cancelText: "😅算了算了",
      onOk() {
        deleteKanban({ id: kanban.id });
      },
    });
  };
  const overlay = (
    <Menu>
      <Menu.Item>
        <Button type={"link"} onClick={confirmDeleteKanban}>
          删除
        </Button>
      </Menu.Item>
    </Menu>
  );
  return (
    <Dropdown overlay={overlay}>
      <Button type={"link"}>...</Button>
    </Dropdown>
  );
};

export const KanbanColumn = React.forwardRef<HTMLDivElement, KanbanColumnProps>(
  ({ kanban, allTasks, ...props }, ref) => {
    const tasks = allTasks?.filter((task) => task.kanbanId === kanban.id);

    return (
      <Container ref={ref} {...props}>
        <Row between={true}>
          <h2>{kanban.name} 💡</h2>
          <More kanban={kanban} key={kanban.id} />
        </Row>
        <Drop
          type={"ROW"}
          direction={"vertical"}
          droppableId={String(kanban.id)}
        >
          <TaskContainer>
            {tasks?.map((task, index) => (
              <Drag key={task.id} index={index} draggableId={"task" + task.id}>
                <div ref={ref}>
                  <TaskCard task={task} key={task.id} />
                </div>
              </Drag>
            ))}
            <CreateTask kanbanId={kanban.id} />
          </TaskContainer>
        </Drop>
      </Container>
    );
  }
);

export const Container = styled.div`
  min-width: 27rem;
  border-radius: 6px;
  background-color: rgb(244, 245, 247);
  display: flex;
  flex-direction: column;
  padding: 0.7rem 0.7rem 1rem;
  margin-right: 1.5rem;
  transition: all 0.4s ease-in-out;
  &:hover {
    background-color: #caf2f5;
    transform: scale(0.98);
    box-shadow: inset -4px -4px 10px rgba(255, 255, 255, 0.5),
      inset 4px 4px 10px rgba(0, 0, 0, 0.1);
  }
`;

const TaskContainer = styled.div`
  min-height: 5px;
  overflow: scroll;
  flex: 1;
  ::-webkit-scrollbar {
    display: none;
  }
`;
const ShadowCard = styled(Card)`
  cursor: pointer;
  border-radius: 10px;
  margin-bottom: 0.5em;
  transition: all 0.5s linear;
  &:hover {
    color: #2acfdb;
    transform: scale(0.97);
    box-shadow: inset -4px -4px 10px rgba(164, 237, 240, 0.5),
      inset 4px 4px 10px rgba(4, 196, 221, 0.1);
  }
`;
```

#### 3.4.3.`src/screens/kanban/search-panel.tsx`:

任务搜索组件。由 Input 输入框以及 UserSelect、TaskTypeSelect、EpicSelect 下拉选择器构成的筛选器。

```tsx | pure
import { Button, Input } from "antd";
import { EpicSelect } from "components/epic-select";
import { Row } from "components/lib";
import { TaskTypeSelect } from "components/task-type-select";
import { UserSelect } from "components/user-select";
import { useSetUrlSearchParam } from "utils/url";
import { useTasksSearchParams } from "./util";

export const SearchPanel = () => {
  const searchParams = useTasksSearchParams();
  const setSearchParams = useSetUrlSearchParam();
  const reset = () => {
    setSearchParams({
      typeId: undefined,
      processorId: undefined,
      epicId: undefined,
      name: undefined,
    });
  };
  return (
    <Row marginBottom={4} gap={true}>
      <Input
        style={{ width: "20rem" }}
        placeholder={"任务名"}
        value={searchParams.name}
        onChange={(evt) =>
          setSearchParams({ ...searchParams, name: evt.target.value })
        }
      />
      <UserSelect
        defaultOptionName={"经办人"}
        value={searchParams.processorId}
        onChange={(value) => setSearchParams({ processorId: value })}
      />
      <TaskTypeSelect
        defaultOptionName={"类型"}
        value={searchParams.typeId}
        onChange={(value) => setSearchParams({ typeId: value })}
      />
      <EpicSelect
        defaultOptionName={"任务组"}
        value={searchParams.epicId}
        onChange={(value) => setSearchParams({ epicId: value })}
      />
      <Button onClick={reset}>清除筛选器</Button>
    </Row>
  );
};
```

#### 3.4.4.`src/screens/kanban/create-kanban.tsx`:

创建看板组件。包含一个 Input 输入框。

```tsx | pure
import { Input } from "antd";
import { useState } from "react";
import { useAddKanban } from "utils/kanban";
import { Container } from "./kanban-column";
import { useKanbansQueryKey, useProjectIdInUrl } from "./util";

export const CreateKanban = () => {
  const [name, setName] = useState("");
  const projectId = useProjectIdInUrl();
  const { mutateAsync: addKanban } = useAddKanban(useKanbansQueryKey());

  const submit = async () => {
    await addKanban({ name, projectId });
    setName("");
  };
  return (
    <Container>
      <Input
        size={"large"}
        placeholder={"新建看板👨‍💻"}
        onPressEnter={submit}
        value={name}
        onChange={(evt) => setName(evt.target.value)}
      />
    </Container>
  );
};
```

#### 3.4.5.`src/screens/kanban/create-task.tsx`:

创建任务组件。包含一个 Input 输入框，失去焦点时展示按钮。

```tsx | pure
import { Button, Card, Input, Tooltip } from "antd";
import { useEffect, useState } from "react";
import { useAddTask } from "utils/task";
import {
  useProjectIdInUrl,
  useTasksQueryKey,
  useTasksSearchParams,
} from "./util";

export const CreateTask = ({ kanbanId }: { kanbanId: number }) => {
  const [name, setName] = useState("");
  const { mutateAsync: addTask } = useAddTask(useTasksQueryKey());
  const projectId = useProjectIdInUrl();
  const { processorId, epicId } = useTasksSearchParams();
  const [inputMode, setInputMode] = useState(false);

  const submit = async () => {
    await addTask({ projectId, name, kanbanId, processorId, epicId });
    setInputMode(false);
    setName("");
  };

  const toggle = () => setInputMode((mode) => !mode);

  useEffect(() => {
    if (!inputMode) {
      setName("");
    }
  }, [inputMode]);
  if (!inputMode) {
    return (
      <Tooltip
        placement="topLeft"
        title="初始值为当前筛选器，类型默认为task"
        color={"cyan"}
        arrowPointAtCenter
      >
        <Button style={{ borderRadius: 6 }} type={"dashed"} onClick={toggle}>
          👉创建事务
        </Button>
      </Tooltip>
    );
  }
  return (
    <Card>
      <Input
        onBlur={toggle}
        placeholder={"需要做些什么"}
        onPressEnter={submit}
        value={name}
        onChange={(evt) => setName(evt.target.value)}
      />
    </Card>
  );
};

```

### 3.5.任务组页面

#### 3.5.1.`src/screens/epic/index.tsx`:

任务组页面入口。包含任务组列表，以栅格形式布局。

```tsx | pure
import styled from "@emotion/styled";
import { Button, Card, List, Modal } from "antd";
import { Row, ScreenContainer } from "components/lib";
import dayjs from "dayjs";
import { useState } from "react";
import { Link } from "react-router-dom";
import { useProjectInUrl } from "screens/kanban/util";
import { Epic } from "types/epic";
import { useDeleteEpic, useEpics } from "utils/epic";
import { useTasks } from "utils/task";
import { CreateEpic } from "./create-epic";
import { useEpicsQueryKey } from "./util";

export const EpicScreen = () => {
  const { data: currentProject } = useProjectInUrl();
  const { data: epics } = useEpics({ projectId: currentProject?.id });
  const { data: tasks } = useTasks({ projectId: currentProject?.id });

  const { mutate: deleteEpic } = useDeleteEpic(useEpicsQueryKey());
  const [epicCreateOpen, setEpicCreateOpen] = useState(false);

  const confirmDeleteEpic = (epic: Epic) => {
    Modal.confirm({
      title: "🙃你确定要删除这个任务组🐴？",
      content: "😡别怪我没提醒你",
      okText: "😶确定",
      cancelText: "😅算了算了",
      onOk() {
        deleteEpic({ id: epic.id });
      },
    });
  };
  return (
    <ScreenContainer>
      <Row gap={5} marginBottom={3}>
        <h1>{currentProject?.name}任务组</h1>
        <Button
          style={{ borderRadius: "10px", opacity: "0.9" }}
          type={"primary"}
          onClick={() => setEpicCreateOpen(true)}
        >
          创建任务组
        </Button>
      </Row>
      <List
        grid={{ gutter: 50, column: 4 }}
        dataSource={epics}
        itemLayout={"vertical"}
        renderItem={(epic) => (
          <List.Item>
            <ShadowCard style={{ minWidth: "250px" }}>
              <List.Item.Meta
                title={
                  <Row between={true}>
                    <span>{epic.name}</span>
                    <Button
                      style={{
                        backgroundColor: "#2fcff7",
                        color: "white",
                        opacity: "0.8",
                      }}
                      onClick={() => confirmDeleteEpic(epic)}
                      type={"ghost"}
                    >
                      删除
                    </Button>
                  </Row>
                }
                description={
                  <div>
                    <div>
                      开始时间：{dayjs(epic.start).format("YYYY-MM-DD")}
                    </div>
                    <div>结束时间：{dayjs(epic.end).format("YYYY-MM-DD")}</div>
                  </div>
                }
                avatar={"🗃️"}
              />
              <div>
                任务：
                {tasks
                  ?.filter((task) => task.epicId === epic.id)
                  .map((task) => (
                    <Link
                      style={{ display: "flex" }}
                      to={`/projects/${currentProject?.id}/kanban?editingTaskId=${task.id}`}
                      key={task.id}
                    >
                      {`📝 ${task.name}`}
                    </Link>
                  ))}
              </div>
            </ShadowCard>
          </List.Item>
        )}
      />
      <CreateEpic
        onClose={() => setEpicCreateOpen(false)}
        visible={epicCreateOpen}
      />
    </ScreenContainer>
  );
};

const ShadowCard = styled(Card)`
  border-radius: 5px;
  margin-bottom: 0.5em;
  transition: all 0.5s ease-in-out;
  &:hover {
    color: #6d43e2;
    background-color: #bcc7fc;
    transform: scale(1.02);
    box-shadow: inset -4px -4px 10px rgba(164, 237, 240, 0.5),
      inset 4px 4px 10px rgba(4, 196, 221, 0.1);
  }
`;
```

#### 3.5.2.`src/screens/epic/create-epic.tsx`:

创建任务组模态框。

```tsx | pure
import styled from "@emotion/styled";
import { Button, Drawer, DrawerProps, Form, Input, Spin } from "antd";
import { ErrorBox } from "components/lib";
import { useEffect } from "react";
import { useProjectIdInUrl } from "screens/kanban/util";
import { useAddEpic } from "utils/epic";
import { useEpicsQueryKey } from "./util";
import access from "assets/access.svg";

export const CreateEpic = (
  props: Pick<DrawerProps, "visible"> & { onClose: () => void }
) => {
  const { mutate: addEpic, isLoading, error } = useAddEpic(useEpicsQueryKey());
  const [form] = Form.useForm();
  const projectId = useProjectIdInUrl();

  const onFinish = async (values: any) => {
    await addEpic({ ...values, projectId });
    props.onClose();
  };

  useEffect(() => {
    form.resetFields();
  }, [form, props.visible]);

  return (
    <Drawer
      visible={props.visible}
      onClose={props.onClose}
      forceRender={true}
      destroyOnClose={true}
      width={"100%"}
    >
      <Background>
        <Container>
          {isLoading ? (
            <Spin size={"large"} />
          ) : (
            <>
              <h1>创建任务组</h1>
              <ErrorBox error={error} />
              <Form
                form={form}
                layout={"vertical"}
                style={{ width: "40rem" }}
                onFinish={onFinish}
              >
                <Form.Item
                  label={"名称"}
                  name={"name"}
                  rules={[{ required: true, message: "请输入任务组名" }]}
                >
                  <Input placeholder={"请输入任务组名称"} />
                </Form.Item>

                <Form.Item style={{ textAlign: "right" }}>
                  <Button
                    loading={isLoading}
                    type={"primary"}
                    htmlType={"submit"}
                  >
                    提交
                  </Button>
                </Form.Item>
              </Form>
            </>
          )}
        </Container>
      </Background>
    </Drawer>
  );
};

const Container = styled.div`
  height: 80vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
`;

const Background = styled.div`
  width: 100%;
  height: 100%;
  background-attachment: fixed;
  background-position: left 32% bottom 60%;
  background-size: contain;
  background-repeat: no-repeat;
  opacity: 0.8;
  background-image: url(${access});
`;
```

## 4.组件

与划分至页面层的组件不同，此类组件通常是可复用的组件，与页面组件耦合度较低，可以在不同的页面中使用。

### 4.1.基础组件

#### 4.1.1.`src/components/lib.tsx`:

项目中常用的 tsx 组件及 css 组件。

```tsx | pure
import styled from "@emotion/styled";
import { Button, Spin, Typography } from "antd";
import { DevTools } from "jira-dev-tool";
import { isError } from "utils";
import space from "assets/space.svg";

/* tsx组件块
 ============================================================================ */
export const FullPageLoading = () => (
  <FullPage>
    <Spin size={"large"} />
  </FullPage>
);

export const FullPageErrorFallback = ({ error }: { error: Error | null }) => (
  <FullPage>
    <DevTools />
    <ErrorBox error={error} />
  </FullPage>
);

export const ErrorBox = ({ error }: { error: unknown }) => {
  if (isError(error)) {
    return <Typography.Text type={"danger"}>{error?.message}</Typography.Text>;
  }
  return null;
};

/* CSS组件块
 ============================================================================ */
export const Row = styled.div<{
  gap?: number | boolean;
  between?: boolean;
  marginBottom?: number;
}>`
  display: flex;
  align-items: center;
  justify-content: ${(props) => (props.between ? "space-between" : undefined)};
  margin-bottom: ${(props) => props.marginBottom + "rem"};
  > * {
    margin-top: 0 !important;
    margin-bottom: 0 !important;
    margin-right: ${(props) =>
      typeof props.gap === "number"
        ? props.gap + "rem"
        : props.gap
        ? "2rem"
        : undefined};
  }
`;

const FullPage = styled.div`
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  background-image: url(${space});
`;

export const ButtonNoPadding = styled(Button)`
  padding: 0;
`;

export const ScreenContainer = styled.div`
  padding: 3.2rem;
  width: 100vw;
  display: flex;
  flex-direction: column;
  overflow: scroll;
  ::-webkit-scrollbar {
    display: none;
  }
`;
```

#### 4.1.2.`src/components/mark.tsx`:

关键词高亮组件。

```tsx | pure
/**
 * keyword 高亮
 * @param name
 * @param keyword
 */
export const Mark = ({ name, keyword }: { name: string; keyword: string }) => {
  if (!keyword) {
    return <>{name}</>;
  }
  const arr = name.split(keyword);
  return (
    <>
      {arr.map((str: string, index: number) => (
        <span key={index}>
          {str}
          {index === arr.length - 1 ? null : (
            <span style={{ color: "#257AFD" }}>{keyword}</span>
          )}
        </span>
      ))}
    </>
  );
};
```

#### 4.1.3.`src/components/pin.tsx`:

点亮收藏组件。

```tsx | pure
import { Rate } from "antd";
import React from "react";

interface PinProps extends React.ComponentProps<typeof Rate> {
  checked: boolean;
  onCheckedChange?: (checked: boolean) => void;
}
export const Pin = ({ checked, onCheckedChange, ...restProps }: PinProps) => {
  return (
    <Rate
      count={1}
      value={checked ? 1 : 0}
      onChange={(num) => onCheckedChange?.(!!num)}
      {...restProps}
    ></Rate>
  );
};
```

#### 4.1.4.`src/components/drag-and-drop.tsx`:

拖拽组件。

```tsx | pure
import React, { ReactNode } from "react";
import {
  Droppable,
  Draggable,
  DroppableProvided,
  DroppableProps,
  DraggableProps,
  DroppableProvidedProps,
} from "react-beautiful-dnd";

type DropProps = Omit<DroppableProps, "children"> & { children: ReactNode };

type DropChildProps = Partial<
  { provided: DroppableProvided } & DroppableProvidedProps
> &
  React.HtmlHTMLAttributes<HTMLDivElement>;

type DragProps = Omit<DraggableProps, "children"> & { children: ReactNode };

export const Drop = ({ children, ...props }: DropProps) => {
  return (
    <Droppable {...props}>
      {(provided) => {
        if (React.isValidElement(children)) {
          //以子元素为样板克隆并返回新的React元素，强制为所有子元素添加指定的props
          return React.cloneElement(children, {
            ...provided.droppableProps,
            ref: provided.innerRef,
            provided,
          });
        }
        return <div />;
      }}
    </Droppable>
  );
};
//创建一个React组件，这个组件能够将其接受的 ref 属性转发到其组件树下的另一个组件中。
export const DropChild = React.forwardRef<HTMLDivElement, DropChildProps>(
  ({ children, ...props }, ref) => (
    <div ref={ref} {...props}>
      {children}
      {props.provided?.placeholder}
    </div>
  )
);

export const Drag = ({ children, ...props }: DragProps) => {
  return (
    <Draggable {...props}>
      {(provided) => {
        if (React.isValidElement(children)) {
          return React.cloneElement(children, {
            ...provided.draggableProps,
            ...provided.dragHandleProps,
            ref: provided.innerRef,
          });
        }
        return <div />;
      }}
    </Draggable>
  );
};
```

### 4.2.user 相关组件

#### 4.2.1.`src/components/user-popover.tsx`:

user 气泡卡片。展示组员列表。

```tsx | pure
import styled from "@emotion/styled";
import { Popover, Typography, List, Divider, Button, Modal } from "antd";
import { User } from "types/user";
import { useSetUrlSearchParam, useUrlQueryParam } from "utils/url";
import { useDeleteUser, useUsers, useUsersQueryKey } from "utils/user";
import { ButtonNoPadding } from "./lib";

export const useUserModal = () => {
  const [{ userCreate }, setUserCreate] = useUrlQueryParam(["userCreate"]);

  const setUrlParams = useSetUrlSearchParam();

  const open = () => setUserCreate({ userCreate: true });
  const close = () => setUrlParams({ userCreate: "" });

  return {
    userModalOpen: userCreate === "true",
    open,
    close,
  };
};

export const UserPopover = () => {
  const { data: users } = useUsers();
  const { open } = useUserModal();

  const { mutate: deleteUser } = useDeleteUser(useUsersQueryKey());
  const confirmDeleteUser = (user: User) => {
    Modal.confirm({
      title: "🙃你确定要删除这个组员🐴？",
      content: "😡别怪我没提醒你",
      okText: "😶确定",
      cancelText: "😅算了算了",
      onOk() {
        deleteUser({ id: user.id });
      },
    });
  };

  const content = (
    <ContentContainer>
      <Typography.Text type={"secondary"}>组员列表</Typography.Text>
      <List>
        {users?.map((user) => (
          <List.Item key={user.id}>
            <List.Item.Meta title={user.name} />
            <List.Item.Meta title={user.organization} />
            <Button onClick={() => confirmDeleteUser(user)}>删除</Button>
          </List.Item>
        ))}
      </List>
      <Divider />
      <ButtonNoPadding onClick={open} type={"link"}>
        添加组员
      </ButtonNoPadding>
    </ContentContainer>
  );

  return (
    <Popover placement={"bottom"} content={content}>
      <span style={{ cursor: "pointer" }}>组员</span>
    </Popover>
  );
};

const ContentContainer = styled.div`
  min-width: 30rem;
`;
```

#### 4.2.2.`src/components/create-user.tsx`:

添加组员模态框。

```tsx | pure
import styled from "@emotion/styled";
import { Button, Drawer, Form, Input, Spin } from "antd";
import { ErrorBox } from "components/lib";
import { useEffect } from "react";
import access from "assets/access.svg";
import { useAddUser, useUsersQueryKey } from "utils/user";
import { useUserModal } from "./user-popover";

export const CreateUser = () => {
  const { userModalOpen, close } = useUserModal();
  const { mutate: addUser, isLoading, error } = useAddUser(useUsersQueryKey());
  const [form] = Form.useForm();

  const onFinish = async (values: any) => {
    await addUser({ ...values });
    close();
  };

  useEffect(() => {
    form.resetFields();
  }, [form]);

  return (
    <Drawer
      visible={userModalOpen}
      onClose={close}
      forceRender={true}
      destroyOnClose={true}
      width={"100%"}
    >
      <Background>
        <Container>
          {isLoading ? (
            <Spin size={"large"} />
          ) : (
            <>
              <h1>新增组员</h1>
              <ErrorBox error={error} />
              <Form
                form={form}
                layout={"vertical"}
                style={{ width: "40rem" }}
                onFinish={onFinish}
              >
                <Form.Item
                  label={"名称"}
                  name={"name"}
                  rules={[{ required: true, message: "请输入组员名" }]}
                >
                  <Input placeholder={"请输入组员名称"} />
                </Form.Item>
                <Form.Item
                  label={"部门"}
                  name={"organization"}
                  rules={[{ required: true, message: "请输入部门名" }]}
                >
                  <Input placeholder={"请输入所属部门"} />
                </Form.Item>

                <Form.Item style={{ textAlign: "right" }}>
                  <Button
                    loading={isLoading}
                    type={"primary"}
                    htmlType={"submit"}
                  >
                    提交
                  </Button>
                </Form.Item>
              </Form>
            </>
          )}
        </Container>
      </Background>
    </Drawer>
  );
};

const Container = styled.div`
  height: 80vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
`;

const Background = styled.div`
  width: 100%;
  height: 100%;
  background-attachment: fixed;
  background-position: left 32% bottom 60%;
  background-size: contain;
  background-repeat: no-repeat;
  opacity: 0.8;
  background-image: url(${access});
`;
```

### 4.3.Select 相关组件

#### 4.3.1.`src/components/id-select.tsx`:

所有 select 组件的基础。根据传入的 options 展示不同数据。

```tsx | pure
import { Select } from "antd";
import { Raw } from "types";
import { toNumber } from "utils";

type SelectProps = React.ComponentProps<typeof Select>;

interface IdSelectProps
  extends Omit<SelectProps, "value" | "onChange" | "options"> {
  value?: Raw | null | undefined;
  onChange?: (value?: number) => void;
  defaultOptionName?: string;
  options?: { name: string; id: number }[];
}

/**
 * value可传入多种类型的值；
 * onChange只回调 number | undefined 类型；
 * 当 isNaN(Number(value)) 为 true 时代表选择默认类型;
 * 选择默认类型时， onChange 回调 undefined;
 * @param props
 */
export const IdSelect = (props: IdSelectProps) => {
  const { value, onChange, defaultOptionName, options, ...restProps } = props;
  return (
    <Select
      value={options?.length ? toNumber(value) : 0}
      onChange={(value) => onChange?.(toNumber(value) || undefined)}
      {...restProps}
    >
      {defaultOptionName ? (
        <Select.Option value={0}>{defaultOptionName}</Select.Option>
      ) : null}
      {options?.map((option) => (
        <Select.Option key={option.id} value={option.id}>
          {option.name}
        </Select.Option>
      ))}
    </Select>
  );
};
```

#### 4.3.2.`src/components/user-select.tsx`:

user 下拉选择器。在 id-select 组件的基础上封装，传入的 options 为 users。

```tsx | pure
import React from "react";
import { useUsers } from "utils/user";
import { IdSelect } from "./id-select";

export const UserSelect = (props: React.ComponentProps<typeof IdSelect>) => {
  const { data: users } = useUsers();
  return <IdSelect options={users || []} {...props}></IdSelect>;
};
```

#### 4.3.3.`src/components/task-type-select.tsx`:

taskType 下拉选择器。在 id-select 组件的基础上封装，传入的 options 为 taskTypes。

```tsx | pure
import { useTaskTypes } from "utils/task-type";
import { IdSelect } from "./id-select";

export const TaskTypeSelect = (
  props: React.ComponentProps<typeof IdSelect>
) => {
  const { data: taskTypes } = useTaskTypes();
  return <IdSelect options={taskTypes || []} {...props}></IdSelect>;
};
```

#### 4.3.4.`src/components/epic-select.tsx`:

epic 下拉选择器。在 id-select 组件的基础上封装，传入的 options 为 epics。

```tsx | pure
import { useEpicsSearchParams } from "screens/epic/util";
import { useEpics } from "utils/epic";
import { IdSelect } from "./id-select";

export const EpicSelect = (props: React.ComponentProps<typeof IdSelect>) => {
  const { data: epics } = useEpics(useEpicsSearchParams());
  return <IdSelect options={epics || []} {...props}></IdSelect>;
};
```

### 4.4.project 相关组件

#### 4.4.1.`src/components/project-popover.tsx`:

project 气泡卡片。展示收藏的项目列表。

```tsx | pure
import styled from "@emotion/styled";
import { Popover, Typography, List, Divider } from "antd";
import { Link } from "react-router-dom";
import { useProjectModal } from "screens/projectList/util";
import { useProjects } from "utils/project";
import { ButtonNoPadding } from "./lib";

export const ProjectPopover = () => {
  const { data: projects } = useProjects();
  const { open } = useProjectModal();
  const pinnedProjects = projects?.filter((project) => project.pin);
  const content = (
    <ContentContainer>
      <Typography.Text type={"secondary"}>收藏项目</Typography.Text>
      <List>
        {pinnedProjects?.map((project) => (
          <List.Item key={project.id}>
            <List.Item.Meta title={project.name} />
            <Link to={"projects/" + project.id}>👉查看</Link>
          </List.Item>
        ))}
      </List>
      <Divider />
      <ButtonNoPadding onClick={open} type={"link"}>
        创建项目
      </ButtonNoPadding>
    </ContentContainer>
  );

  return (
    <Popover placement={"bottom"} content={content}>
      <span style={{ cursor: "pointer" }}>项目</span>
    </Popover>
  );
};

const ContentContainer = styled.div`
  min-width: 30rem;
`;
```

## 5.工具

项目中使用的抽象方法和 custom hooks。

### 5.1.基本工具

#### 5.1.1.`src/utils/index.ts`:

```ts | pure
// 自定义工具库
import { useEffect, useRef, useState } from "react";

/**
 * 数字类型转换，无法转换默认为 0
 * @param {unknown} value
 * @returns 0 | Number(value)
 */
export const toNumber = (value: unknown) =>
  isNaN(Number(value)) ? 0 : Number(value);

/**
 * 布尔类型转换,且排除 value为 0 的特殊情况
 * @param {unknown} value
 * @returns  对 value 取反
 */
export const isFalsy = (value: unknown) => (value === 0 ? false : !value);

/**
 * 判断值有无意义
 * @param {unknown} value
 */
export const isVoid = (value: unknown) =>
  value === undefined || value === null || value === "";

/**
 * 类型守卫
 * 如果为 Error类型，应具有message属性
 * @param value
 */
export const isError = (value: any): value is Error => value?.message;

/**
 * 清理对象空属性
 * @param {object} object
 * @returns {object}
 */

export const cleanObject = (object?: { [key: string]: unknown }) => {
  if (!object) {
    return {};
  }
  //在函数里改变传入的对象是不好的行为,Object.assign({},object)仍会发生浅拷贝，不推荐
  const result = JSON.parse(JSON.stringify(object));
  Object.keys(result).forEach((key) => {
    const value = result[key];
    if (isVoid(value)) {
      delete result[key];
    }
  });
  return result;
};

/**
 * 利用custom hook实现mount钩子
 * @param {function} callback
 */
export const useMount = (callback: () => void) => {
  useEffect(() => {
    callback();
    //eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);
};

/**
 * 利用custom hook 实现的 debounce
 * @template V
 * @param {V} value
 * @param {number} delay
 * @returns {V} debouncedValue
 */
export const useDebounce = <V>(value: V, delay?: number) => {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    // value变化后设定计时器
    const timeout = setTimeout(() => setDebouncedValue(value), delay);

    // 上一个useEffect处理完后执行
    return () => clearTimeout(timeout);
  }, [value, delay]);

  return debouncedValue;
};

/**
 * 设置页面标题
 * @param {string} title
 * @param {boolean} keepOnUnmount 卸载后保持标题不变
 */
export const useDocumentTitle = (
  title: string,
  keepOnUnmount: boolean = true
) => {
  const oldTitle = useRef(document.title).current;
  useEffect(() => {
    document.title = title;
  }, [title]);

  useEffect(() => {
    return () => {
      if (!keepOnUnmount) {
        document.title = oldTitle;
      }
    };
  }, [keepOnUnmount, oldTitle]);
};

/**
 * 重置为根路由
 */
export const resetRoute = () => (window.location.href = window.location.origin);

/**
 * 传入一个对象，和键集合，返回对应的对象中的键值对
 * @param obj
 * @param keys
 */
export const subset = <
  O extends { [key in string]: unknown },
  K extends keyof O
>(
  obj: O,
  keys: K[]
) => {
  const filteredEntries = Object.entries(obj).filter(([key]) =>
    keys.includes(key as K)
  );
  return Object.fromEntries(filteredEntries) as Pick<O, K>;
};
/**
 * 返回组件挂载状态;
 * @returns false:未挂载或已卸载 | true: 已挂载
 */
export const useMountedRef = () => {
  const mountedRef = useRef(false);

  useEffect(() => {
    mountedRef.current = true;
    return () => {
      mountedRef.current = false;
    };
  });
  return mountedRef;
};
```

### 5.2.请求

#### 5.2.1.`src/utils/http.ts`:

可扩展的自定义 fetch 请求。是所有请求的基础。

```ts | pure
import qs from "qs";
import * as auth from "auth-provider";
import { useAuth } from "context/auth-context";
import { useCallback } from "react";
const apiUrl = process.env.REACT_APP_API_URL;

interface Config extends RequestInit {
  token?: string;
  data?: object;
}
/**
 * 自定义 fetch 请求封装
 * @param endpoint 接口地址
 * @param Config 自定义配置
 * @returns Promise
 */
export const http = async (
  endpoint: string,
  { data, token, headers, ...customConfig }: Config = {}
) => {
  const config = {
    method: "GET",
    headers: {
      Authorization: token ? `Bearer ${token}` : "",
      "Content-Type": data ? "application/json" : "",
    },
    ...customConfig,
  };

  if (config.method.toUpperCase() === "GET") {
    endpoint += `?${qs.stringify(data)}`;
  } else {
    config.body = JSON.stringify(data || {});
  }

  return window
    .fetch(`${apiUrl}/${endpoint}`, config)
    .then(async (response) => {
      if (response.status === 401) {
        await auth.logout();
        window.location.reload();
        return Promise.reject({ message: "请重新登录" });
      }
      const data = await response.json();
      if (response.ok) {
        return data;
      } else {
        return Promise.reject(data);
      }
    });
};

/**
 * 自动获取当前用户的 token 加入 headers
 * @returns Promise
 */
export const useHttp = () => {
  const { user } = useAuth();
  return useCallback(
    (...[endpoint, config]: Parameters<typeof http>) =>
      http(endpoint, { ...config, token: user?.token }),
    [user?.token]
  );
};
```

#### 5.2.2.`src/utils/use-async.ts`:

useAsync hook 处理异步请求，根据所处阶段返回不同状态。

```ts | pure
import { useCallback, useReducer, useState } from "react";
import { useMountedRef } from "utils";

interface State<D> {
  error: Error | null;
  data: D | null;
  stat: "idle" | "loading" | "error" | "success";
}

const defaultConfig = {
  throwOnError: false,
};

/**
 * 自动判断组件是否挂载，挂载则返回可调用 dispatch 的回调函数
 * @param dispatch
 * @returns callback
 */
const useSafeDispatch = <T>(dispatch: (...args: T[]) => void) => {
  const mountedRef = useMountedRef();
  return useCallback(
    (...args: T[]) => (mountedRef.current ? dispatch(...args) : void 0),
    [dispatch, mountedRef]
  );
};

const defaultIinitialState: State<null> = {
  stat: "idle",
  data: null,
  error: null,
};
/**
 * 处理异步请求
 * @param {State<D>} initialState
 * @param {boolean} initialConfig
 */
export const useAsync = <D>(
  initialState?: State<D>,
  initialConfig?: typeof defaultConfig
) => {
  const config = { ...defaultConfig, ...initialConfig };
  const [state, dispatch] = useReducer(
    (state: State<D>, action: Partial<State<D>>) => ({ ...state, ...action }),
    {
      ...defaultIinitialState,
      ...initialState,
    }
  );
  const [retry, setRetry] = useState(() => () => {});
  const safeDispatch = useSafeDispatch(dispatch);

  const setData = useCallback(
    (data: D) =>
      safeDispatch({
        data,
        stat: "success",
        error: null,
      }),
    [safeDispatch]
  );

  const setError = useCallback(
    (error: Error) =>
      safeDispatch({
        error,
        stat: "error",
        data: null,
      }),
    [safeDispatch]
  );

  /**
   * 触发异步请求
   * @param {Promise<D>} Promise
   */
  const run = useCallback(
    (promise: Promise<D>, runConfig?: { retry: () => Promise<D> }) => {
      if (!promise || !promise.then) {
        throw new Error("请传入 Promise 类型数据");
      }
      setRetry(() => () => {
        if (runConfig?.retry) {
          run(runConfig?.retry(), runConfig);
        }
      });
      safeDispatch({ stat: "loading" });
      return promise
        .then((data) => {
          setData(data);
          return data;
        })
        .catch((error) => {
          setError(error);
          if (config.throwOnError) {
            //主动抛出异常
            return Promise.reject(error);
          }
          return error;
        });
    },
    [config.throwOnError, safeDispatch, setData, setError]
  );
  return {
    isIdle: state.stat === "idle",
    isLoading: state.stat === "loading",
    isError: state.stat === "error",
    isSuccess: state.stat === "success",
    run,
    retry,
    setData,
    setError,
    ...state,
  };
};
```

### 5.3.URL

#### 5.3.1.`src/utils/url.ts`:

useUrlQueryParam hook。基于 url 的状态管理。

```ts | pure
import { useMemo, useState } from "react";
import { URLSearchParamsInit, useSearchParams } from "react-router-dom";
import { cleanObject, subset } from "utils";

export const useUrlQueryParam = <K extends string>(keys: K[]) => {
  const [searchParams] = useSearchParams();
  const setSearchParams = useSetUrlSearchParam();
  const [stateKeys] = useState(keys);
  return [
    useMemo(
      () =>
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

export const useSetUrlSearchParam = () => {
  const [searchParams, setSearchParam] = useSearchParams();
  return (params: { [key in string]: unknown }) => {
    const o = cleanObject({
      ...Object.fromEntries(searchParams),
      ...params,
    }) as URLSearchParamsInit;
    return setSearchParam(o);
  };
};
```

### 5.4.乐观更新

#### 5.4.1.`src/utils/use-optimistic-options.ts`：

基于 react-query 的乐观更新配置。不同配置对应不同操作。

```ts | pure
import { message } from "antd";
import { QueryKey, useQueryClient } from "react-query";
import { Task } from "types/task";
import { reorder } from "./reorder";

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

export const useDeleteConfig = (queryKey: QueryKey) =>
  useConfig(
    queryKey,
    (target, old) => old?.filter((item) => item.id !== target.id) || []
  );

export const useEditConfig = (queryKey: QueryKey) =>
  useConfig(
    queryKey,
    (target, old) =>
      old?.map((item) =>
        item.id === target.id ? { ...item, ...target } : item
      ) || []
  );

export const useAddConfig = (queryKey: QueryKey) =>
  useConfig(queryKey, (target, old) => (old ? [...old, target] : []));

export const useReorderKanbanConfig = (queryKey: QueryKey) =>
  useConfig(queryKey, (target, old) => reorder({ list: old, ...target }));

export const useReorderTaskConfig = (queryKey: QueryKey) =>
  useConfig(queryKey, (target, old) => {
    const orderedList = reorder({ list: old, ...target }) as Task[];
    return orderedList.map((item) =>
      item.id === target.fromId
        ? { ...item, kanbanId: target.toKanbanId }
        : item
    );
  });
```

#### 5.4.2.`src/utils/reorder.ts`：

重排序乐观更新的具体实现。

```ts | pure
/**
 * 在本地对排序进行乐观更新
 * @param fromId 要排序的项目的id
 * @param type 'before' | 'after'
 * @param referenceId 参照id
 * @param list 要排序的列表, 比如tasks, kanbans
 */
export const reorder = ({
  fromId,
  type,
  referenceId,
  list,
}: {
  list: { id: number }[];
  fromId: number;
  type: "after" | "before";
  referenceId: number;
}) => {
  const copiedList = [...list];
  // 找到fromId对应项目的下标
  const movingItemIndex = copiedList.findIndex((item) => item.id === fromId);
  if (!referenceId) {
    return insertAfter([...copiedList], movingItemIndex, copiedList.length - 1);
  }
  const targetIndex = copiedList.findIndex((item) => item.id === referenceId);
  const insert = type === "after" ? insertAfter : insertBefore;
  return insert([...copiedList], movingItemIndex, targetIndex);
};

/**
 * 在list中，把from放在to的前边
 * @param list
 * @param from
 * @param to
 */
const insertBefore = (list: unknown[], from: number, to: number) => {
  const toItem = list[to];
  const removedItem = list.splice(from, 1)[0];
  const toIndex = list.indexOf(toItem);
  list.splice(toIndex, 0, removedItem);
  return list;
};

/**
 * 在list中，把from放在to的后面
 * @param list
 * @param from
 * @param to
 */
const insertAfter = (list: unknown[], from: number, to: number) => {
  const toItem = list[to];
  const removedItem = list.splice(from, 1)[0];
  const toIndex = list.indexOf(toItem);
  list.splice(toIndex + 1, 0, removedItem);
  return list;
};
```

### 5.5.项目列表相关

#### 5.5.1.`src/utils/project.ts`：

关于 project 数据的增删查改请求。

```ts | pure
import { QueryKey, useMutation, useQuery } from "react-query";
import { Project } from "types/project";
import { cleanObject } from "utils";
import { useHttp } from "./http";
import {
  useAddConfig,
  useDeleteConfig,
  useEditConfig,
} from "./use-optimistic-options";

export const useProjects = (param?: Partial<Project>) => {
  const client = useHttp();

  return useQuery<Project[]>(["projects", cleanObject(param)], () =>
    client("projects", { data: param })
  );
};

/**
 * 编辑项目
 * @returns mutate--调用其处理请求，并乐观更新
 */
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
/**
 * 添加项目
 * @returns mutate--调用其处理请求，并乐观更新
 */
export const useAddProject = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation(
    (params: Partial<Project>) =>
      client(`projects`, {
        data: params,
        method: "POST",
      }),
    useAddConfig(queryKey)
  );
};
/**
 * 删除项目
 * @returns mutate--调用其处理请求，并乐观更新
 */
export const useDeleteProject = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation(
    ({ id }: { id: number }) =>
      client(`projects/${id}`, {
        method: "DELETE",
      }),
    useDeleteConfig(queryKey)
  );
};

/**
 * 获取项目详情
 * @param id
 */
export const useProject = (id?: number) => {
  const client = useHttp();
  return useQuery<Project>(
    ["project", { id }],
    () => client(`projects/${id}`),
    {
      enabled: Boolean(id),
    }
  );
};
```

#### 5.5.2.`src/screens/projectList/util.ts`:

项目列表业务相关工具。

```ts | pure
import { useMemo } from "react";
import { useProject } from "utils/project";
import { useSetUrlSearchParam, useUrlQueryParam } from "utils/url";
/**
 * 项目列表搜索参数
 * @returns [param, setParam]
 */
export const useProjectsSearchParams = () => {
  const [param, setParam] = useUrlQueryParam(["name", "personId"]);
  return [
    useMemo(
      () => ({
        ...param,
        personId: Number(param.personId) || undefined,
      }),
      [param]
    ),
    setParam,
  ] as const;
};
/**
 * 根据当前 url 获取 query 缓存键值
 * @returns ["projects", params];
 */
export const useProjectsQueryKey = () => {
  const [params] = useProjectsSearchParams();
  return ["projects", params];
};
/**
 * 获取项目模态框状态
 */
export const useProjectModal = () => {
  const [{ projectCreate }, setProjectCreate] = useUrlQueryParam([
    "projectCreate",
  ]);
  const [{ editingProjectId }, setEditingProjectId] = useUrlQueryParam([
    "editingProjectId",
  ]);
  const { data: editingProject, isLoading } = useProject(
    Number(editingProjectId)
  );
  const setUrlParams = useSetUrlSearchParam();

  const open = () => setProjectCreate({ projectCreate: true });
  const close = () => setUrlParams({ projectCreate: "", editingProjectId: "" });

  const startEdit = (id: number) =>
    setEditingProjectId({ editingProjectId: id });
  return {
    projectModalOpen: projectCreate === "true" || Boolean(editingProjectId),
    open,
    close,
    startEdit,
    editingProject,
    isLoading,
  };
};
```

### 5.6.user 相关

#### 5.6.1.`src/utils/user.ts`:

组员查询、添加、删除请求。

```tsx | pure
import { QueryKey, useMutation, useQuery } from "react-query";
import { User } from "types/user";
import { cleanObject } from "utils";
import { useHttp } from "./http";
import { useAddConfig, useDeleteConfig } from "./use-optimistic-options";

export const useUsers = (param?: Partial<User>) => {
  const client = useHttp();

  return useQuery<User[]>(["users", cleanObject(param)], () =>
    client("users", { data: param })
  );
};
export const useUsersQueryKey = () => {
  return ["users", {}];
};
/**
 * 添加项目
 * @returns mutate--调用其处理请求，并乐观更新
 */
export const useAddUser = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation(
    (params: Partial<User>) =>
      client(`users`, {
        data: params,
        method: "POST",
      }),
    useAddConfig(queryKey)
  );
};
/**
 * 删除项目
 * @returns mutate--调用其处理请求，并乐观更新
 */
export const useDeleteUser = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation(
    ({ id }: { id: number }) =>
      client(`users/${id}`, {
        method: "DELETE",
      }),
    useDeleteConfig(queryKey)
  );
};
```

### 5.7.看板、任务相关

#### 5.7.1.`src/utils/kanban.ts`:

看板添加、删除、重排序请求。

```ts | pure
import { QueryKey, useMutation, useQuery } from "react-query";
import { Kanban } from "types/kanban";
import { SortProps } from "types/sort";
import { cleanObject } from "utils";
import { useHttp } from "./http";
import {
  useAddConfig,
  useDeleteConfig,
  useReorderKanbanConfig,
} from "./use-optimistic-options";

export const useKanbans = (param?: Partial<Kanban>) => {
  const client = useHttp();

  return useQuery<Kanban[]>(["kanbans", cleanObject(param)], () =>
    client("kanbans", { data: param })
  );
};
/**
 * 添加看板
 * @returns mutate--调用其处理请求，并乐观更新
 */
export const useAddKanban = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation(
    (params: Partial<Kanban>) =>
      client(`kanbans`, {
        data: params,
        method: "POST",
      }),
    useAddConfig(queryKey)
  );
};

/**
 * 删除看板
 * @returns mutate--调用其处理请求，并乐观更新
 */
export const useDeleteKanban = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation(
    ({ id }: { id: number }) =>
      client(`kanbans/${id}`, {
        method: "DELETE",
      }),
    useDeleteConfig(queryKey)
  );
};
/**
 * 看板重新排序
 *@returns mutate--调用其处理请求，并乐观更新
 */
export const useReorderKanban = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation((params: SortProps) => {
    return client("kanbans/reorder", {
      data: params,
      method: "POST",
    });
  }, useReorderKanbanConfig(queryKey));
};
```

#### 5.7.2.`src/utils/task.ts`:

任务增删查改、重排序请求。

```ts | pure
import { QueryKey, useMutation, useQuery } from "react-query";
import { SortProps } from "types/sort";
import { Task } from "types/task";
import { useHttp } from "./http";
import {
  useAddConfig,
  useDeleteConfig,
  useEditConfig,
  useReorderTaskConfig,
} from "./use-optimistic-options";

export const useTasks = (param?: Partial<Task>) => {
  const client = useHttp();

  return useQuery<Task[]>(["tasks", param], () =>
    client("tasks", { data: param })
  );
};
/**
 * 添加 task
 * @returns mutate--调用其处理请求，并乐观更新
 */
export const useAddTask = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation(
    (params: Partial<Task>) =>
      client(`tasks`, {
        data: params,
        method: "POST",
      }),
    useAddConfig(queryKey)
  );
};
/**
 * 编辑 task
 * @returns mutate--调用其处理请求，并乐观更新
 */
export const useEditTask = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation(
    (params: Partial<Task>) =>
      client(`tasks/${params.id}`, {
        data: params,
        method: "PATCH",
      }),
    useEditConfig(queryKey)
  );
};
/**
 * 删除 task
 * @returns mutate--调用其处理请求，并乐观更新
 */
export const useDeleteTask = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation(
    ({ id }: { id: number }) =>
      client(`tasks/${id}`, {
        method: "DELETE",
      }),
    useDeleteConfig(queryKey)
  );
};
/**
 * 获取task详情
 * @param id
 */
export const useTask = (id?: number) => {
  const client = useHttp();
  return useQuery<Task>(["task", { id }], () => client(`tasks/${id}`), {
    enabled: Boolean(id),
  });
};
/**
 * task重新排序
 * @returns mutate--调用其处理请求，并乐观更新
 */
export const useReorderTask = (queryKey: QueryKey) => {
  const client = useHttp();
  return useMutation((params: SortProps) => {
    return client("tasks/reorder", {
      data: params,
      method: "POST",
    });
  }, useReorderTaskConfig(queryKey));
};
```

#### 5.7.3.`src/utils/task-type.ts`:

taskType 查询请求。

```ts | pure
import { useQuery } from "react-query";
import { TaskType } from "types/task-type";
import { useHttp } from "./http";

export const useTaskTypes = () => {
  const client = useHttp();

  return useQuery<TaskType[]>(["taskTypes"], () => client("taskTypes"));
};
```

#### 5.7.4.`src/screens/kanban/util.ts`:

看板、任务业务相关工具。

```ts | pure
import { useCallback, useMemo } from "react";
import { useLocation } from "react-router";
import { useProject } from "utils/project";
import { useTask } from "utils/task";
import { useUrlQueryParam } from "utils/url";
/**
 * 获取当前 projectId
 */
export const useProjectIdInUrl = () => {
  const { pathname } = useLocation();
  const id = pathname.match(/projects\/(\d+)/)?.[1];
  return Number(id);
};
/**
 * 根据当前projectId获取project数据
 */
export const useProjectInUrl = () => useProject(useProjectIdInUrl());

export const useKanbansSearchParams = () => ({
  projectId: useProjectIdInUrl(),
});

export const useKanbansQueryKey = () => ["kanbans", useKanbansSearchParams()];

export const useTasksSearchParams = () => {
  const [param] = useUrlQueryParam(["name", "typeId", "processorId", "epicId"]);
  const projectId = useProjectIdInUrl();
  return useMemo(
    () => ({
      projectId,
      typeId: Number(param.typeId) || undefined,
      processorId: Number(param.processorId) || undefined,
      epicId: Number(param.epicId) || undefined,
      name: param.name,
    }),
    [projectId, param]
  );
};

export const useTasksQueryKey = () => ["tasks", useTasksSearchParams()];

export const useTaskModal = () => {
  const [{ editingTaskId }, setEditingTaskId] = useUrlQueryParam([
    "editingTaskId",
  ]);
  const { data: editingTask, isLoading } = useTask(Number(editingTaskId));
  const startEdit = useCallback(
    (id: number) => setEditingTaskId({ editingTaskId: id }),
    [setEditingTaskId]
  );
  const close = useCallback(
    () => setEditingTaskId({ editingTaskId: "" }),
    [setEditingTaskId]
  );

  return {
    editingTaskId,
    editingTask,
    startEdit,
    close,
    isLoading,
  };
};
```

