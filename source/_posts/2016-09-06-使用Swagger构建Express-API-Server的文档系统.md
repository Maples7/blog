---
title: 使用Swagger构建Express API Server的文档系统
subtitle: build-doc-system-of-express-api-server-with-swagger
tags:
  - Swagger
  - Express
  - Node.js
  - API
categories: 一只代码狗的自我修养
date: 2016-09-06 22:16:11
---
如[上一篇博客](http://maples7.com/2016/09/05/build-qualified-restful-api-server/)所说，好的文档系统对API Server至关重要，本文介绍在Express框架中使用Swagger构建一个良好的项目文档系统的基本流程，同时明确一些实践过程中肯定会遇到的问题的解决方案。本文遵循`Swagger 2.0`使用规范。

{% fullimage http://oc3nlt0h2.bkt.clouddn.com/20160907.png, 图片显示错误, Swagger——The World's Most Popular Framework for APIs %}
<!-- more -->
## 目标
- 文档生成的「源」（或者说「依据」）与代码不分离，即直接用`jsdoc`注释生成文档；
- 可以用同样的「源」同时实现对接口输入输出参数的验证，最大化保证文档与后端具体实现之间的一致性；
- 文档在线可用性测试，并且可以完美解决跨域请求的问题；
- 在后端接口还未完成时，可以Mock返回数据；              
- 最好能自动生成一些测试数据甚至自动进行测试； 

## 从 JSDoc 到可视化文档
### Step 1：定义接口模型
在Controller层每一条路由的函数注释上（具体来说，Routes目录下或Controller目录下均可，只要配置好`Step 2`中的`swagger-jsdoc`，明确「源」所在的目录即可）按`Swagger YAML`语法定义接口模型，示例如下:
```js
/**
 * @swagger
 * definition:
 *   Puppy:
 *     properties:
 *       name:
 *         type: string
 *       breed:
 *         type: string
 *       age:
 *         type: integer
 *       sex:
 *         type: string
 */

/**
 * @swagger
 * /api/puppies:
 *   get:
 *     tags:
 *       - Puppies
 *     description: Returns all puppies
 *     produces:
 *       - application/json
 *     responses:
 *       200:
 *         description: An array of puppies
 *         schema:
 *           $ref: '#/definitions/Puppy'
 */
router.get('/api/puppies', db.getAllPuppies);

/**
 * @swagger
 * /api/puppies:
 *   post:
 *     tags:
 *       - Puppies
 *     description: Creates a new puppy
 *     produces:
 *       - application/json
 *     parameters:
 *       - name: puppy
 *         description: Puppy object
 *         in: body
 *         required: true
 *         schema:
 *           $ref: '#/definitions/Puppy'
 *     responses:
 *       200:
 *         description: Successfully created
 */
router.post('/api/puppies', db.createPuppy);
```

一些抽象出来的`definition`直接在前面的`/** */`注释中定义即可。

在生成的完整配置中，同一个路径下的配置（比如上述示例中`/api/puppies`下的`get`与`post`）会合并在这同一个路径之下，所以同一个路径下的全局配置只用写一遍就行了（比如下文`Mock返回数据`小节中使用`Swagger Router`中间件需要的`x-swagger-router-controller`配置）。

### Step 2：生成 Swagger 接口定义
用 [swagger-jsdoc](https://github.com/Surnet/swagger-jsdoc) 生成JSON格式的Swagger接口定义。
Demo：[mjhea0/node-swagger-api](https://github.com/mjhea0/node-swagger-api)。

这里不用把生成的JSON保存在本地磁盘上，直接用一个变量引用即可。

### Step 3：用 Swagger UI 生成可视化文档
在线查看：打开[http://petstore.swagger.io/](http://petstore.swagger.io/)，在顶部的URL栏输入可以获取`Step 2`中生成的JSON格式的Swagger文档定义的URL，~~一般来说可以是http://localhost:3000/swagger.json （需要自己手动在代码中书写路由和请求的返回）~~ 在使用了后文要说明的[Swagger UI中间件](https://github.com/apigee-127/swagger-tools/blob/master/docs/Middleware.md#swagger-ui)之后按照默认配置是 http://localhost:3000/api-docs （注意最后没有`/`）。这种方式需要解决跨域请求的问题，详见后文。

~~本地离线查看：直接在本项目public（静态文件目录）下放置离线版`Swagger UI`，直接打开即可查看。详见`Step 2`中的Demo及作者的博文说明。~~

本地离线查看：使用`swagger-tools`中的[Swagger UI中间件](https://github.com/apigee-127/swagger-tools/blob/master/docs/Middleware.md#swagger-ui)，如果你直接使用默认配置：
- http://localhost:3000/docs/ 可以看到完整的可视化文档；
- http://localhost:3000/api-docs （注意最后没有`/`）可以看到自己在`Step 2`中自动生成的Swagger接口定义。

这个中间件可以让你在开发时再也无需操心可视化文档的前端实现和如何查看自动生成Swagger接口定义（以便确定是否符合规范和自己的需求）的问题。

## 一些常见问题
### 解决跨域请求的问题
官方说明见：[https://github.com/swagger-api/swagger-ui#cors-support](https://github.com/swagger-api/swagger-ui#cors-support)。
在`app.use('/', routes);`之前加一个如下的中间件设置一些cors相关的头可以解决问题: 
```js
app.use((req, res, next) => {
  res.setHeader('Access-Control-Allow-Origin', 'http://petstore.swagger.io');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, DELETE, PUT, PATCH, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, api_key, Authorization');
  next();
});
```

### 接口参数验证
使用`swagger-tools`中的[Swagger Metadata](https://github.com/apigee-127/swagger-tools/blob/master/docs/Middleware.md#swagger-metadata)和[Swagger Validator](https://github.com/apigee-127/swagger-tools/blob/master/docs/Middleware.md#swagger-validator)中间件：  
- [swagger-tools Quick Start](https://github.com/apigee-127/swagger-tools/blob/master/docs/QuickStart.md)；
- [Demo](https://github.com/apigee-127/swagger-tools/tree/master/examples/2.0)；

其中Swagger Metadata中间件做了匹配请求路由与Swagger定义路由以及解析参数的工作，Swagger Validator中间件做了验证参数类型和其他已定义的参数限制的工作。

### Mock 返回数据
使用`swagger-tools`中的[Swagger Router中间件](https://github.com/apigee-127/swagger-tools/blob/master/docs/Middleware.md#swagger-router)即可实现。

正常情况下，根据你的Swagger定义会返回`Response Code`为`200`（当Swagger定义中已定义`200`的返回时）的类似这样的数据：
```
[
  {
    "name": "Sample text",
    "breed": "Sample text",
    "age": 1,
    "sex": "Sample text"
  }
]
```

### 自动生成测试代码
使用[apigee-127/swagger-test-templates](https://github.com/apigee-127/swagger-test-templates)可以根据你的Swagger定义自动生成对所有接口功能测试的脚手架代码（基本可以自动确定的地方都自动生成了），在你把自动生成的代码写入磁盘文件后，只需修改极少量的地方（一般是提供一些需要测试的参数）就可以使用测试。

自动生成的代码长成这样：
```js
'use strict';
var chai = require('chai');
var ZSchema = require('z-schema');
var validator = new ZSchema({});
var supertest = require('supertest');
var api = supertest('http://localhost:3000'); // supertest init;

chai.should();

describe('/api/puppies', function() {
  describe('get', function() {
    it('should respond with 200 An array of puppies', function(done) {
      /*eslint-disable*/
      var schema = {
        "type": "array",
        "items": {
          "$ref": "#/definitions/Puppy"
        }
      };

      /*eslint-enable*/
      api.get('/api/puppies')
      .set('Accept', 'application/json')
      .expect(200)
      .end(function(err, res) {
        if (err) return done(err);

        validator.validate(res.body, schema).should.be.true;
        done();
      });
    });

  });

  describe('post', function() {
    it('should respond with 200 Successfully created', function(done) {
      api.post('/api/puppies')
      .set('Accept', 'application/json')
      .send({
        puppy: 'DATA GOES HERE'
      })
      .expect(200)
      .end(function(err, res) {
        if (err) return done(err);

        res.body.should.equal(null); // non-json response or no schema
        done();
      });
    });

  });

});
```

之后可以考虑用gulp把测试串联起来进行自动化测试。npm上已经有[这样的包](https://www.npmjs.com/package/gulp-swagger-test-templates)，不过我还没有试过。

目前这个包对于`Swagger 2.0`的支持还不是很完全，尤其是对`$ref`不能自动解析，这样的话需要手动改动的测试代码多一些，不过他们正在着力解决这个问题，估计下一个版本（1.3.0）就会加入对`$ref`的解析，详细的讨论可以[看这个issue](https://github.com/apigee-127/swagger-test-templates/issues/104)。同时，他们还在考虑添加通过JSON-Schema自动批量生成测试数据的功能，目测[也将在下一个版本中推出](https://github.com/apigee-127/swagger-test-templates/pull/107)，值得期待。

### Mock 或 Swagger UI 失效
由于异步的问题，如果你把`app.lieten`写在`swaggerTools.initializeMiddleware`的回调函数外面，那很可能在你的应用已经启动时，`swagger-tools`的中间件并没有加载完毕，导致中间件失效（不会报错）。

鉴于此，应该尽量把`swaggerTools.initializeMiddleware`写在中间件链的后面部分，然后把位于其后的`app.use`（比如`app.use('/', routes);`）和`app.listen`写在`swaggerTools.initializeMiddleware`的回调函数内部。所以，`Express 4`中提倡的用`./bin/www`来启动应用的要求在这里可能无法被遵循了。

更详细的讨论[看这个issue](https://github.com/apigee-127/swagger-tools/issues/328)。

另外，Mock失效还有可能是你已经提供了对应Controller来Handle对应的Route请求。并不是将Swagger Router中间件中的`useStubs`设为`true`就一定会启动Mock，官方对此说明是：
> Stubs only work for requests where the controller and/or controller method is missing. Since you have a working controller method, enabling stub mode doesn't do anything. It's working as designed.

更详细的讨论可以[看这个issue](https://github.com/apigee-127/swagger-tools/issues/48)。

### 调试技巧
使用`DEBUG=swagger-tools* node app`启动项目，控制台会输出更多详细的信息。

## 完整的 Demo
[Maples7/swagger-express-demo](https://github.com/Maples7/swagger-express-demo)

将前面所讲的内容整合进了一个小示例中，以供参考。

## 上手必读
0. [swagger-spec](http://swagger.io/specification/)；
1. [swagger-tools/docs/QuickStart.md](https://github.com/apigee-127/swagger-tools/blob/master/docs/QuickStart.md)；
2. [swagger-tools/docs/Middleware.md](https://github.com/apigee-127/swagger-tools/blob/master/docs/Middleware.md)；
3. [swagger-tools/docs/API.md](https://github.com/apigee-127/swagger-tools/blob/master/docs/API.md)；
