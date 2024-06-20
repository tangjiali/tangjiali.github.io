---
title: Ant Design Pro改造点记录
date: 2024-04-22 14:57:50 +0800
categories: [编程, 前端]
tags:
- Ant Design Pro
image:
- path: https://pro.ant.design/static/background_normal.394f91a5.svg
- width: 100%
---

## 版本选择

我这里选择了V5版本来作为后台管理系统的模板，这样可以用到最新的特性，至于稳定性，反正也只是开发自己使用的小工具，倒是其次。

我们可以直接下载代码压缩包，也可以使用使用如下命令下载代码：

```shell
git clone https://github.com/ant-design/ant-design-pro.git 
```

当然，我们也可以使用`pro`命令创建工程：

```shell
pro create szhome-admin-react
```

## 安装依赖

我这里使用`pnpm`命令安装依赖：

```shell
pnpm install
```

正常此时我们可以使用如下命令运行工程：

```shell
pnpm run dev
```

然后我们就可以根据终端显示的地址访问工程，通常是http://localhost:8000。

## 着手改造

### 启用接口Mock

前面我们使用`pnpm run dev`启动工程，此时并没有启用Ant Design Pro提供的接口Mock能力。如果我们没有配置自己的接口地址，可能连登录都登录不了。

此时我们可以打开工程根目录下的`package.json`文件，找到`scripts`配置，可以看到`pnpm run dev`等价于`pnpm run start:dev`，而继续找到`start:dev`可以发现，它指定了`MOCK=none`，即不启用Mock能力。

现在我们知道`pnpm run dev`命令无法启用Mock能力，我们就可以在`scripts`中找到对应启用了Mock能力的命令：

```shell
pnpm run start
```

### 配置主题

正常情况下你可以在工程启动后访问页面，并在页面右侧中央位置看到一个设置按钮。点击按钮，即可配置主题风格。Ant Design Pro支持实时预览，待主题风格调整满意后，即可点击最下方的`拷贝设置`按钮，此时你将得到一份主题风格的配置JSON。

回到源码，找到工程目录下的`config/defaultSettings.ts`，替换其中对应的配置，即可完成默认风格的设置。

### 主题配置入口

当你想要配置主题时，可能你找不到这个配置入口。这是因为高版本中默认好像把这个配置入口的按钮给去掉了，但源码还在，只是注释掉了。

又或者你不希望用户可以调整布局主题风格，你也可以主动把这个配置入口屏蔽掉。

你可以在`src/app.tsx`文件中找到`childrenRender`配置项，删除`SettingDrawer`组件即可隐藏主题风格配置的入口，同样放开注释即可放出主题风格配置的入口。

### 支持国际化

Ant Design Pro支持国际化，但有很多我们可能根本用不到的语言。只需要删除对应的目录，语言切换菜单即会更新为剩余的语言列表。国际化配置相关的文件在工程根目录下的`src/locales`目录中，删除不需要的语言的ts文件以及目录即可。

通常我会保留以下几种语言：

- zh-CN：简体中文；
- zh-TW：繁体中文；
- en-US：英语；
- ja-JP：日语。

如果我们把所有的国际化配置都写在`src/locales`目录下的文件中，则略显臃肿，且条理不清晰。我们还可以再`src/pages`目录下的各个组件目录创建`locales`子目录，并添加对应语言的`ts`文件，其内容即可自动被国际化组件识别。比如在`src/pages/user`目录下创建`locales`目录，并创建`zh-CN.ts`文件，文件内容按照国际化配置编写，如：

```js
export default {
  'pages.user.btn.add': '新增用户',
};
```

如此即可在`src/pages/user`目录下的各个组件页面使用`pages.user.btn.add`国际化编码。

### 防截屏水印

Ant Design Pro默认情况下会给页面背景加一个水印，水印文本是当前登录用户的用户名，做了一定程度的倾斜。

但是也许你不想要水印，尤其是开发过程中，就是看水印不顺眼。这个水印的配置在工程根目录下的`src/app.tsx`文件中，你可以搜索`waterMarkProps`关键字，注释掉它的唯一一行`content: initialState?.currentUser?.name`，刷新页面就可以看到水印已经不见了。

### OpenAPI文档链接

左侧菜单中有一个**OpenAPI文档**的菜单，它固定于左侧边栏的下方，如果你不想要这个菜单，也可以在`src/app.tsx`文件中`layout`配置项的`links`配置，置空即可去掉此链接。

### 多页签

Ant Design Pro V6版本终于更新了多页签功能的支持，而且使用方法特别简单。你只需要在`config/config.ts`中的配置加上如下配置即使用多页签：

```ts
keepalive: [/./],
tabsLayout: {
	hasDropdown: true,
	hasFixedHeader: true
},
```

嗯，就是这么简单。

至于效果嘛，我觉得不够美观或者简洁。你也可以在`src/global.less`中调整页签的样式。当然，有了页签之后，页面的面包屑和标题可能会有人觉得多余，你也可以使用如下方式去掉：

```tsx
<PageContainer breadcrumb={{}} title={false}>
    ...
</PageContainer>
```

这种方式需要在每个页面都加上这样的配置，比较麻烦。任何问题我们总是应该想一些比较简单的、全局性的方案，比如在`src/global.tsx`中加入下面这样的代码：

```tsx
PageContainer.defaultProps = {
  breadcrumb: {},
  title: false,
  ghost: true,
}
```

注意，`defaultProps`这是个已标记过期的属性，通过VSCode点击可以看到相关说明。如果你有时间，可以仔细阅读说明文档，修改为最新的实现方式。

### 使用自己的服务端接口

#### 停用Mock

正如前面提到的如何启用Mock功能，反之，使用`pnpm run dev`就可以以不使用Mock功能的方式启动工程。或者明确指定停用Mock：

```shell
pnpm run start:no-mock
## 或
pnpm run start MOCK=none
```

#### 声明代理

停用Mock后，我们需要指定自己的服务端接口地址，首先是`config/proxy.ts`中的代理配置，将dev环境的配置放开：

```tsx
dev: {
  // localhost:8000/api/** -> http://localhost:8080
  '/api/': {
    // 要代理的地址
    target: 'http://localhost:8080',
    // 配置了这个可以从 http 代理到 https
    // 依赖 origin 的功能可能需要这个，比如 cookie
    changeOrigin: true,
    pathRewrite: {'/api': ''}
  },
},
```

如上，这意味着前端工程在请求后端`localhost:8000/api/**`接口时，实际上后端提供的是`localhost:8080/**`接口。

#### 修改baseURL

最后需要指定Ant Design Pro请求后端接口的基础路径，找到`src/app.tsx`，拉倒最后几行代码，可以看到`RequestConfig`配置项，将`baseURL`修改为你的前端工程访问地址。如：

```tsx
export const request: RequestConfig = {
  // baseURL: 'https://proapi.azurewebsites.net',
  baseURL: 'http://localhost:8080',
  ...errorConfig,
};
```

#### 请求示例

现在我们就可以编写请求后端接口的方法了：

```tsx
import { request } from '@umijs/max';

export async function detail(id: Number | string) {
  return request(`/api/user/${id}`, {
    method: 'GET'
  });
}
```

如上，我们在请求后端接口时只是使用了`/user/${id}`，实际上Ant Design Pro根据配置 ，首先会将`baseURL`补上，即请求`http://localhost:8080/api/user/${id}`，然后再根据代理配置，实际请求的接口地址则是`http://localhost:8080/user/${id}`。

### 接口响应数据结构

接口响应定义通常有着一致数据结构，Ant Design Pro的要求的后端接口响应数据结构如下：

```json
// 1. 请求处理失败
{
    success: false,
    error: "5001",
    message: "请求参数错误"
}

// 2. 请求处理成功，返回数组，无分页
{
    success: true,
    data: [{
        name: "admin"
    }]
}

// 3. 请求处理成功，返回对象
{
    success: true,
    data: {
        name: "admin"
    }
}

// 4. 请求处理成功，分页
{
    success: true,
    data: [{
        name: "admin"
    }],
    total: 32,
	pageSize: 10,
	current: 1
}
```

## 接口改造

### 登录

登录是使用Ant Design Pro的第一步，登录页实现在`src/pages/user/login`目录。在该目录的`index.tsx`文件中，你可以找到`handleSubmit`方法，这里是处理登录的逻辑。

```tsx
const handleSubmit = async (values: API.LoginParams) => {
  try {
    // 登录
    const msg = await login({ ...values, type });
    if (msg.status === 'ok') {
      const defaultLoginSuccessMessage = intl.formatMessage({
        id: 'pages.login.success',
        defaultMessage: '登录成功！',
      });
      message.success(defaultLoginSuccessMessage);
      await fetchUserInfo();
      const urlParams = new URL(window.location.href).searchParams;
      window.location.href = urlParams.get('redirect') || '/';
      return;
    }
    console.log(msg);
    // 如果失败去设置用户错误信息
    setUserLoginState(msg);
  } catch (error) {
    const defaultLoginFailureMessage = intl.formatMessage({
      id: 'pages.login.failure',
      defaultMessage: '登录失败，请重试！',
    });
    console.log(error);
    message.error(defaultLoginFailureMessage);
  }
};
```

从上面的代码中可以看出来，它首先请求了`login`方法，如果该方法返回值的`status`属性值为`ok`，表示登录成功。登录成功时先提示了消息，然后继续请求`fetchUserInfo`方法加载当前登录用户信息，然后跳转页面。如果登录失败，则设置用户登录失败的状态。这期间如果发生异常，也会给出提示消息。

再来看`login`方法，它定义在`src/services/ant-design-pro/api.ts`文件中：

```tsx
export async function login(body: API.LoginParams, options?: { [key: string]: any }) {
  return request<API.LoginResult>('/api/login/account', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    data: body,
    ...(options || {}),
  });
}
```

可以看到它就是简单的请求了`/api/login/account`这样一个方法。因此当我们要使用自己的服务端接口时，我们需要定义一个登录方法，响应结果和`/api/login/account`一样即可。你可以通过F12查看开启Mock情况下的该接口返回结构，也可以在`mock/user.ts`文件中找到该接口的响应定义。

然后是`fetchUserInfo`方法，这个方法就定义在登录页中：

```tsx
const { initialState, setInitialState } = useModel('@@initialState');

// ...

const fetchUserInfo = async () => {
  const userInfo = await initialState?.fetchUserInfo?.();
  if (userInfo) {
    flushSync(() => {
      setInitialState((s) => ({
        ...s,
        currentUser: userInfo,
      }));
    });
  }
};
```

要想理解这个方法，务必先看官方文档：[全局初始数据](https://pro.ant.design/zh-CN/docs/initial-state)，否则可能会云里雾里，搞不清楚这些变量、方法都是哪里来的，在干什么。而一旦阅读完这篇篇幅极短的文档后，你就会豁然开朗，原来这里兜了个圈子，真正加载当前登录用户信息的实现在`src/app.tsx`文件中的`getInitialState`方法中。

```tsx
export async function getInitialState(): Promise<{
  settings?: Partial<LayoutSettings>;
  currentUser?: API.CurrentUser;
  loading?: boolean;
  fetchUserInfo?: () => Promise<API.CurrentUser | undefined>;
}> {
  const fetchUserInfo = async () => {
    try {
      const msg = await queryCurrentUser({
        skipErrorHandler: true,
      });
      return msg.data;
    } catch (error) {
      history.push(loginPath);
    }
    return undefined;
  };
  // 如果不是登录页面，执行
  const { location } = history;
  if (![loginPath, '/user/register', '/user/register-result'].includes(location.pathname)) {
    const currentUser = await fetchUserInfo();
    return {
      fetchUserInfo,
      currentUser,
      settings: defaultSettings as Partial<LayoutSettings>,
    };
  }
  return {
    fetchUserInfo,
    settings: defaultSettings as Partial<LayoutSettings>,
  };
}
```

这里显示监测是否为登录、注册、注册结果页，如果都不是，则需要调用`queryCurrentUser`方法查询当前登录用户信息，然后将结果返回，继而在登录页的`fetchUserInfo`方法将返回的用户信息写入到全局数据中。

>  这里有个意外之喜，那就是我们可以看到这里的全局数据中有一个叫`setting`的，它的取值来自`config/defaultSetting.ts`，是配置页面主题风格的。那么如果我们对这一块逻辑进行修改，从后端返回每个人的风格配置，是不是就可以实现每个人不同的页面风格的能力。

所以，真是的登录用户信息还得看`queryCurrentUser`方法的实现，在`src/services/ant-design-pro/api.ts`中：

```tsx
export async function currentUser(options?: { [key: string]: any }) {
  return request<{
    data: API.CurrentUser;
  }>('/api/currentUser', {
    method: 'GET',
    ...(options || {}),
  });
}
```

然后我们就看到这个方法仅仅是调用后端`/api/currentUser`接口，它的响应结果所包含的字段，可以查看`API.CurrentUser`类型，该类型的定义在`src/services/ant-design-pro/typings.d.ts`中：

```tsx
declare namespace API {
  type CurrentUser = {
    name?: string;
    avatar?: string;
    userid?: string;
    email?: string;
    signature?: string;
    title?: string;
    group?: string;
    tags?: { key?: string; label?: string }[];
    notifyCount?: number;
    unreadCount?: number;
    country?: string;
    access?: string;
    geographic?: {
      province?: { label?: string; key?: string };
      city?: { label?: string; key?: string };
    };
    address?: string;
    phone?: string;
  };
}
```

嗯，That's all。当然，如果你喜欢，你完全可以自己写一个登录页，毕竟Ant Design Pro的登录页也不太好看。

### 请求拦截器

当你改写登录接口时也许不会想到需要拦截所有的请求，一旦登录成功后，你就会有在所有请求发起前往Header中加入token的需求。

我们可以给我们所使用的`reqeust`加上一个拦截器，它的相关配置在`src/requestErrorConfig.ts`中。在这个文件中搜索`requestInterceptors`，你会看到这里已经有一个拦截器，它给所有的接口请求都加了参数`?token=123`。这是一个示例，我们把它移除，加上我们自己的拦截器逻辑即可：

```tsx
// 请求拦截器
requestInterceptors: [
  (url: string, options: RequestConfig) => {
    let tokens: any = {'Trace-id': uuidv4()};

    if(url !== '/system/login' && url !== '/system/captcha'){
      const token = localStorage.getItem('token');
      tokens['Authorization'] = 'Bearer ' + token;
    }

    return {
      url,
      options: { ...options, interceptors: true, headers: tokens },
    };
  }
],
```

如上， 我在拦截器实现了在每个请求的请求头中放入了`Authorization`和`Trace-id`头信息的功能。这样后端就可以完成认证以及日志追踪了。

### 退出登录

由于我自定义了登录页面，使用完全不同的登录接口，甚至数据结构都有变化。登录虽然成功了，但登出却出现了问题。因此我还需要改造退出登录的功能。

退出登录的视线在`src/components/RightContent/AvatarDropdown.tsx`中，可以看到这个文件中有一个`loginOut`方法，只需要修改此方法，即可完成退出登录的改造。这个可能需要结合你自己的登录实现来做调整，比如你在登录时保存了`token`，则在登出时将`token`清除即可。

## 常见问题

### Absolute route path "/*" nested under path "/user" is not valid

登录成功后可能会看到白屏，F12打开开发者工具，会看到如下报错：

![image-20240522174918050](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202405221749103.png)

解决方法：找到工程根目录下的`config/routes.ts`，把`/user`路由下的`404`路由删掉即可。

### export 'useSyncExternalStore' (imported as 'React2') was not found in 'react'

如下图，工程编译通过，但访问页面时报如下错误：

![image-20240522172826870](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202405221728972.png)

以`export 'useSyncExternalStore' (imported as 'React2') was not found in 'react'`为关键字搜索解决方案，在Stackoverflow上找到如下解决方案：

```shell
npm install react@latest react-dom@latest
npm install -d @types/react@latest @types/react-dom@latest
```

根本原因就是React版本低了，升级到React 18，问题解决。

