---
title: Koa2 源码赏析
subtitle: koa2-src
tags:
 - Koa
 - Node.js
categories: 一只代码狗的自我修养
---
## 前言
随着 Node 新版本开始支持 async/await 异步控制写法，Koa 也相继发布了它的 2.0 版本。用 async/await 写法的 Koa 来开发项目，Node 开发者再也没有任何理由不「拥抱变化」——从 Express 转到 Koa 上来。实际上，对于普通 Node 开发者——Express 框架的用户——而言，从 Express 转到 Koa 没有任何技术壁垒，当然前提是你至少得知道 ES2017 中 async/await 是个什么东西。

{% fullimage http://oc3nlt0h2.bkt.clouddn.com/koa2-src.png, 图片显示错误, Koa——next generation web framework for node.js %}
<!-- more -->

有了 Express 的积累，以 [TJ](https://github.com/tj) 为首的 Express 开发团队对于 Koa 的设计也更加的得心应手、游刃有余。从源代码来看，Koa 看起来甚至比 Express 更加简洁和灵活，然而在功能上却丝毫没有让步。一方面，这得益于 JavaScript 越来越方便好用的各种「语法糖」；另一方面，也在于 Koa 本身简洁强大的设计：它不再绑定任何特定的中间件，而只是简单的提供一个优雅管理各种中间件的约束系统。

TJ 大神的代码一向简洁强大，Koa 的源码也是如此。如果你[去 GitHub 上查看 Koa 的源码](https://github.com/koajs/koa)，你同样会被其简洁所震撼，核心代码不过 4 个文件，平均每个文件代码行数也就四百来行，看似简简单单，却天才般的把 Express 线性的中间件控制流转变为「洋葱体」结构，从而解锁了更多的姿势和玩法。本文就直接深入 Koa 的源码（v2.2.0），来欣赏 Koa 的曼妙身姿。




## 推荐阅读
- [编写可维护代码之“中间件模式”](https://zhuanlan.zhihu.com/p/26063036)
