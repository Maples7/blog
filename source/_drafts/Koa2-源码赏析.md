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

有了 Express 的积累，以 [TJ](https://github.com/tj) 为首的 Express 开发团队对于 Koa 的设计也更加的得心应手、游刃有余。从源代码来看，Koa 看起来甚至比 Express 更加简洁和灵活，然而在功能上却丝毫没有让步，甚至更胜一筹。一方面，这得益于 JavaScript 越来越方便好用的各种「语法糖」；另一方面，也在于 Koa 本身简洁强大的设计：它不再绑定任何特定的中间件，也去掉了其他任何多余的设计（连路由系统都抽象成了第三方中间件），而只是简单的提供了一个优雅管理各种中间件的约束系统，用户的所有挂载都是中间件。

TJ 大神的代码一向简洁强大，Koa 的源码也是如此。如果你[去 GitHub 上查看 Koa 的源码](https://github.com/koajs/koa)，你同样会被其简洁所震撼，核心代码不过 4 个文件，平均每个文件代码行数也就四百来行，看似简简单单，却天才般的把 Express 线性的中间件控制流转变为「洋葱体」结构，从而解锁了更多的姿势和玩法。本文就直接深入 Koa 的源码（v2.2.0），来欣赏 Koa 的曼妙身姿。



## 「洋葱体」带来了什么

你固然还是可以像在 Express 中一样把 `next();` 都放在每个中间件的末尾来线性的传递控制权，但「回形针」式的控制流带来了更多的可能。一个最典型的的例子就是 `response-time` 中间件的实现。

在 GitHub 上用 `response-time` 关键字搜索，前两个 Repo 就分别是 Express 和 Koa 中对这个中间件的实现。

Express 中的实现其实是 hack 了被 Express 用到的 Node.js 内部 HTTP 模块的 `res.writeHead` 方法（实际实现细节在 `response-time` 中间件调用的 `on-headers` 包中），使得在这一层注入了一小段代码用于在数据返回前计算时间差并写入 Response Headers。这样虽然可以实现，但显然不够好。它 hack 了框架底层的一个内部方法，虽然也巧妙，但代码本身并不是在给它天生就安排好的合适的地方来实现的，可以视为晦涩的「奇技淫巧」，而且与 Express 的内部实现强耦合，指不定哪天 Express 改了 `res.writeHead` 调用时机，这个中间件的返回值可能就有所不同了（当然按实际情况来说 Express 此处应该也不会改了，而且 Express 的 `response-time` 最初也是 TJ 写的）。

Koa 的 `response-time` 实现自不必多说，官方文档上就有，来欣赏它的简洁优雅：
```js
app.use(async function (ctx, next) {
  const start = new Date();
  await next();
  const ms = new Date() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
});
```

总结来说，Koa 的「洋葱体」结构使得**每个中间件能够在同一次请求的前后对称的部分提供相同的上下文环境**，这样就让实现像 `response-time` 这样的中间件变得相当简单。

## 推荐阅读
- [编写可维护代码之“中间件模式”](https://zhuanlan.zhihu.com/p/26063036)
