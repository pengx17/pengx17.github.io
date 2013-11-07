---
layout: post
keywords: Bootstrap3
description: blog
title: "博客的前端框架升级至Bootstrap3"
categories: [Blog]
tags: [Blog]
group: archive
icon: bullhorn
---

[Bootstrap](http://getbootstrap.com/getting-started/#migration)框架最近从2.x版本升级到了3.0。
这次大规模更新引入了不少[新特性和组件](http://blog.getbootstrap.com/2013/08/19/bootstrap-3-released/)，同时令所有前端开发者头疼的是，
这个版本不向下兼容2.x版本，所以在版本迁移的时候，简单的替换CDN路径会使得CSS样式表几近失效，让原来的界面面目全非，尤其是`nav`和`navbar`相关的组件；
与此同时[Font-awesome](http://fontawesome.io/whats-new/)图标插件也升级到了4.0，很多图标的名字有了新的名字和命名规范。

xp捣鼓了好几小时，终于差不多搞定了这两个更新带来的一些兼容性问题-。-
不过很多修改只是把一些原来2.x版本的CSS样式表规范拷贝到本地的CSS里面。短期内先这么放着吧！<del>xp你怎么这么懒</del>。
