---
title: 用ES6特性实现一个标准的轻量级Promise
tags:
  - JavaScript
  - ES6
  - Promise
  - Node.js
categories: 一只代码狗的自我修养
---
Promise应该是目前JavaScript中最流行的异步流程控制解决方案，本文将介绍如何使用JavaScript ES6的语言特性，实现一个轻量级的通过了所有[官方测例](https://github.com/promises-aplus/promises-tests)标准的Promise库。了解其原理，深入其实现。实际上，Promise也早已被写入JavaScript ES6的标准，作为官方支持的标准异步流程控制解决方案之一。用ES6语言实现一个ES6中的Promise

为了您的最佳阅读体验，在阅读本文之前，建议您已经做到如下事情：
- 可以熟练使用至少任意一个Promise库，如bluebird、Q等；
- 了解Promise标准，可以[看这里](https://promisesaplus.com/)；
- 熟悉主要的JavaScript ES6特性；    

<!-- more -->
### 构造函数之前的准备

我们都知道，Promise总共有三种状态：pending、fullfilled（resolved）和rejected。所以我们对于每一个Promise实例都需要一个变量记录其现有的状态。

然后还需要一个变量记录其settled之后的结果。另外，Promise作为一个异步流程控制库，在上游的Promise还处于pending状态时下游Promise是不能执行的，所以我们至少需要一个数组来记录当前Promise还未settled时它后续的一些操作（你可以用两个数组分别记录resolved和rejected之后不同的操作；也可以用一个数组，然后每个元素都是包含两个属性的对象，分别记录resolved和rejected之后不同的操作）。

另外，借用面向对象的说法，这些变量对Promise而言应该是私有的，即不应该对外界暴露（这也是符合标准的）。所以待会儿构造函数之中应该定义一些私有变量，而ES6的Symbol类型则可以完美实现我们所需的私有变量。

代码如下：

```js
// 定义 Promise 状态
const STATUS = {
    PENDING: 0,
    RESOLVED: 1,
    REJECTED: 2
};

const _status = Symbol('status'); // 用于 status 私有变量的 Symbol
const _result = Symbol('result'); // 用于 result 私有变量的 Symbol
const _callbacks = Symbol('callbacks'); // 用于 callbacks 私有变量的 Symbol
```

这里我们将Promise状态定义到一个对象之中，并且属性名

### References
1. [剖析Promise内部结构，一步一步实现一个完整的、能通过所有Test case的Promise类 #3 ](https://github.com/xieranmaya/blog/issues/3)；
2. [Promises/A+](https://promisesaplus.com/)；
