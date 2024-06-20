---
title: 使用Ant Design Pro
date: 2024-04-22 14:57:50 +0800
categories: [编程, 前端]
tags:
- Ant Design Pro
image:
- path: https://pro.ant.design/static/background_normal.394f91a5.svg
- width: 100%
---

## 创建Ant Design Pro应用

参考：https://pro.ant.design/zh-CN/docs/getting-started

1. 初始化

```shell
npm i @ant-design/pro-cli -g
pro create szhome-admin-web
```

2. 选择umi

```shell
? 🐂 使用 umi@4 还是 umi@3 ? (Use arrow keys)
❯ umi@4
  umi@3
```

3. 安装依赖

```shell
cd szhome-admin-web
pnpm install
```

4. 启动应用

```shell
pnpm run dev
```

应用启动后默认访问http://localhost:8000即可看到应用页面。

## 定制化改动



## 常见问题

> 注：
>
> 遇事不决，先升级或降级node版本

### Uncaught Error: Absolute route path "/*" nested under path "/user" is not valid. An absolute child route path must start with the combined path of all its parent routes.

找到`routes.ts`路由配置文件，移除`404`相关配置即可：

![image-20240422151031392](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202404221510107.png)
