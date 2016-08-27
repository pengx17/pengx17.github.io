---
layout: post
keywords: blog
description: blog
title: "前端优化探讨：Cache"
categories: [frontend]
tags: [frontend]
group: archive
icon: file-o
---

增强单页应用操作体验的一个重要思路就是增加本地（浏览器）缓存的处理机制。

### 良好的缓存机制有以下好处
- 增加页面的响应速度
  - 如果之前访问过某资源，应直接返回缓存的该资源; 同时重新请求，并以静默的方式更新此资源
- 对于后端API错误相应更加强健
  - 对于轮询的页面，在已有数据的情况下，应该返回缓存的内容
- 减少API请求

### 由于增加缓存处理会给前端开发带来一些新的开发难度，现提供一些处理思路供参考：
- 针对单个GET请求进行全局性缓存
  - 有的API，比如静态资源如 user profile / feature flags，可以做成用户单次加载控制台后全局缓存。这些数据由于很多组件或页面经常调用，而且有些组件严重依赖它们，缓存这些数据可以带来比较显著的加载速度提升。
- 针对数据驱动的列表组件。稍后会重点讲一下分页列表组件的缓存处理思路
- 针对路由跳转
  - 从不同路由跳转时，上下文之间很可能可以把Object通过ui-router v1.0的参数传递
  - 例如
    - 点击资源列表某一项，跳转到资源的详情页

---

### Case Study: 列表

- 列表一般需要有三种状态:
  - Loading
  - Load Error
    - 只有在没有缓存数据，并且API出错时，才应该提示错误，并提供刷新按钮
  - 正常数据显示
- 切换分页页码，更新页面数据

#### 根据分析，我把列表抽象成如下的显示逻辑:

以 `pageno` 为键缓存资源列表数据

用页码切换`pageNoChange`来做页面内容刷新当前页面的数据

`pageNoChange`分成三步走：
- 载入当前缓存数据
- 刷新当前页数据，并更新缓存
  - 当API出错时，标记`loadError = true`
- 重新载入缓存数据

模版显示当前资源列表`vm.list`的数据

- 当`vm.list`为`undefined`时
 - loadError = false, 认为数据正在加载，显示loading mask
 - loadError = true，认为API出错，应提供刷新按钮，绑定在`pageNoChange`
- 当`vm.list`是数组时
  - 如果是空数组，显示空数据提示信息
  - 如果不为空，显示正常数据
  