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
如[上一篇博客](http://maples7.com/2016/09/05/build-qualified-restful-api-server/)所说，好的文档系统对API Server至关重要，本文介绍在Express框架中使用Swagger构建一个良好的项目文档系统的基本流程，同时明确一些实践过程中肯定会遇到的问题的解决方案。

{% fullimage http://oc3nlt0h2.bkt.clouddn.com/20160907.png, 图片显示错误, Swagger——The World's Most Popular Framework for APIs %}
<!-- more -->
## 目标
- 文档生成的「源」（或者说「依据」）与代码不分离，即直接用`jsdoc`生成文档；
- 可以用同样的「源」同时实现接口参数验证，最大化保证文档与后端代码具体实现的一致性；
- 文档在线可用性测试，并且可以完美解决跨域问题；
- 在后端接口还未完成时，可以Mock数据；              
- 最好能自动生成一些测试数据甚至自动进行测试；       // TODO

## 从 JsDoc 到可视化文档
### Step 1：定义接口模型
在每一条定义的Route的函数注释上按Swagger YAML语法定义接口模型，示例如下:
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

### Step 2：生成Swagger文档定义
用 [swagger-jsdoc](https://github.com/Surnet/swagger-jsdoc) 生成JSON格式的Swagger文档定义。
Demo：[mjhea0/node-swagger-api](https://github.com/mjhea0/node-swagger-api)。

### Step 3：用Swagger UI生成可视化文档
在线查看：打开[http://petstore.swagger.io/](http://petstore.swagger.io/)，在顶部的URL栏输入可以获取`Step 2`中生成的JSON格式的Swagger文档定义的URL，一般来说是http://localhost:3000/swagger.json。这种方式需要解决跨域请求的问题，详见后文。

本地离线查看：直接在本项目public（静态文件目录）下放置离线版`Swagger UI`，直接打开即可查看。详见`Step 2`中的Demo及作者的博文说明。

**！！！用后文swagger-tools中的Swagger UI中间件似乎可以直接生成，待考证**

## 一些常见问题

### 接口参数验证
使用`swagger-tools`中的Swagger Metadata和Swagger Validator中间件：
- [Quick Start](https://github.com/apigee-127/swagger-tools/blob/master/docs/QuickStart.md)；
- [Demo](https://github.com/apigee-127/swagger-tools/tree/master/examples)；

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

### Mock返回数据
使用`swagger-tools`中的[Swagger Router中间件](https://github.com/apigee-127/swagger-tools/blob/master/docs/Middleware.md#swagger-router)即可实现。

## 完整的Demo
**!!TODO**

## 上手必读
0. [swagger-spec](http://swagger.io/specification/)；
1. [swagger-tools/docs/QuickStart.md](https://github.com/apigee-127/swagger-tools/blob/master/docs/QuickStart.md)；
2. [swagger-tools/docs/Middleware.md](https://github.com/apigee-127/swagger-tools/blob/master/docs/Middleware.md)；
3. [swagger-tools/docs/API.md](https://github.com/apigee-127/swagger-tools/blob/master/docs/API.md)；
