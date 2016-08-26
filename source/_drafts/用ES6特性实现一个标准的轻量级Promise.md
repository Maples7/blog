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


### References
1. [剖析Promise内部结构，一步一步实现一个完整的、能通过所有Test case的Promise类 #3 ](https://github.com/xieranmaya/blog/issues/3)；
2. [Promises/A+](https://promisesaplus.com/)；
