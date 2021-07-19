从零开始手写Koa2框架

## 01、介绍

- Koa -- 基于 Node.js 平台的下一代 web 开发框架
- Koa 是一个新的 web 框架，由 Express 幕后的原班人马打造， 致力于成为 web 应用和 API 开发领域中的一个更小、更富有表现力、更健壮的基石。
- 与其对应的 Express 来比，Koa 更加小巧、精壮，本文将带大家从零开始实现 Koa 的源码，从根源上解决大家对 Koa 的困惑

> 本文 Koa 版本为 2.7.0, 版本不一样源码可能会有变动

## 02、源码目录介绍

- Koa 源码目录截图  
    
- 通过源码目录可以知道，Koa主要分为4个部分，分别是：
    
    - application: Koa 最主要的模块, 对应 app 应用对象
    - context: 对应 ctx 对象
    - request: 对应 Koa 中请求对象
    - response: 对应 Koa 中响应对象
- 这4个文件就是 Koa 的全部内容了，其中 application 又是其中最核心的文件。我们将会从此文件入手，一步步实现 Koa 框架

## 03、实现一个基本服务器

- 代码目录
- my-application
    
    ```
    const {createServer} = require('http');
    
    module.exports = class Application {
      constructor() {
        
        this.middleware = [];
      }
      
      use(fn) {
        
        this.middleware.push(fn);
      }
      
      listen(...args) {
        
        const server = createServer((req, res) => {
          
          this.middleware.forEach((fn) => fn(req, res));
        })
        server.listen(...args);
      }
    }
    ```
    
- index.js
    
    ```
    
    const MyKoa = require('./js/my-application');
    
    const app = new MyKoa();
    
    app.use((req, res) => {
      console.log('中间件函数执行了~~~111');
    })
    app.use((req, res) => {
      console.log('中间件函数执行了~~~222');
      res.end('hello myKoa');
    })
    
    app.listen(3000, err => {
      if (!err) console.log('服务器启动成功了');
      else console.log(err);
    })
    ```
    
- 运行入口文件 `index.js` 后，通过浏览器输入网址访问 `http://localhost:3000/` , 就可以看到结果了~~
- 神奇吧！一个最简单的服务器模型就搭建完了。当然我们这个极简服务器还存在很多问题，接下来让我们一一解决

## 04、实现中间件函数的 next 方法

- 提取`createServer`的回调函数，封装成一个`callback`方法（可复用）
    
    ```
    
    listen(...args) {
      
      const server = createServer(this.callback());
      server.listen(...args);
    }
    callback() {
      const handleRequest = (req, res) => {
        this.middleware.forEach((fn) => fn(req, res));
      }
      return handleRequest;
    }
    ```
    
- 封装`compose`函数实现`next`方法
    
    ```
    
    function compose(middleware) {
      
      
      return (req, res) => {
        
        return dispatch(0);
        function dispatch(i) {
          
          let fn = middleware[i];
          
          if (!fn) return Promise.resolve();
          
          return Promise.resolve(fn(req, res, dispatch.bind(null, i + 1)));
        }
      }
    }
    ```
    
- 使用`compose`函数
    
    ```
    callback () {
      
      const fn = compose(this.middleware);
      
      const handleRequest = (req, res) => {
        
        
        fn(req, res).then(() => {
          // 在这里就是所有处理的函数的最后阶段，可以允许返回响应了~
        });
      }
      
      return handleRequest;
    }
    ```
    
- 修改入口文件 index.js 代码
    
    ```
    
    const MyKoa = require('./js/my-application');
    
    const app = new MyKoa();
    
    app.use((req, res, next) => {
      console.log('中间件函数执行了~~~111');
      
      next();
    })
    app.use((req, res, next) => {
      console.log('中间件函数执行了~~~222');
      res.end('hello myKoa');
      
      next();
    })
    
    app.listen(3000, err => {
      if (!err) console.log('服务器启动成功了');
      else console.log(err);
    })
    ```
    
- 此时我们实现了`next`方法，最核心的就是`compose`函数，极简的代码实现了功能，不可思议！

## 05、处理返回响应

- 定义返回响应函数`respond`
    
    ```
    function respond(req, res) {
      
      let body = res.body;
      
      if (typeof body === 'object') {
        
        body = JSON.stringify(body);
        res.end(body);
      } else {
        
        res.end(body);
      }
    }
    ```
    
- 在`callback`中调用
    
    ```
    callback() {
      const fn = compose(this.middleware);
      
      const handleRequest = (req, res) => {
        
        const handleResponse = () => respond(req, res);
        fn(req, res).then(handleResponse);
      }
      
      return handleRequest;
    }
    ```
    
- 修改入口文件 index.js 代码
    
    ```
    
    const MyKoa = require('./js/my-application');
    
    const app = new MyKoa();
    
    app.use((req, res, next) => {
      console.log('中间件函数执行了~~~111');
      next();
    })
    app.use((req, res, next) => {
      console.log('中间件函数执行了~~~222');
      
      res.body = 'hello myKoa';
    })
    
    app.listen(3000, err => {
      if (!err) console.log('服务器启动成功了');
      else console.log(err);
    })
    ```
    
- 此时我们就能根据不同响应内容做出处理了~当然还是比较简单的，可以接着去扩展~

## 06、定义 Request 模块

```

const parse = require('parseurl');
const qs = require('querystring');

module.exports = {
  
  get headers() {
    return this.req.headers;
  },
  
  set headers(val) {
    this.req.headers = val;
  },
  
  get query() {
    
    const querystring = parse(this.req).query;
    
    return qs.parse(querystring);
  }
}
```

## 07、定义 Response 模块

```
module.exports = {
  
  set(key, value) {
    this.res.setHeader(key, value);
  },
  
  get status() {
    return this.res.statusCode;
  },
  
  set status(code) {
    this.res.statusCode = code;
  },
  
  get body() {
    return this._body;
  },
  
  set body(val) {
    
    this._body = val;
    
    this.status = 200;
    
    if (typeof val === 'object') {
      this.set('Content-Type', 'application/json');
    }
  },
}
```

## 08、定义 Context 模块

```

const delegate = require('delegates');

const proto = module.exports = ;


delegate(proto, 'response')
  .method('set')    
  .access('status') 
  .access('body')  


delegate(proto, 'request')
  .access('query')
  .getter('headers')  

```

## 09、揭秘 delegates 模块

```
module.exports = Delegator;


function Delegator(proto, target) {
  
  if (!(this instanceof Delegator)) return new Delegator(proto, target);
  
  this.proto = proto;
  
  this.target = target;
  
  this.methods = [];
  
  this.getters = [];
  
  this.setters = [];
}


Delegator.prototype.method = function(name){
  
  var proto = this.proto;
  
  var target = this.target;
  
  this.methods.push(name);
  
  proto[name] = function(){
    
    return this[target][name].apply(this[target], arguments);
  };
  
  return this;
};


Delegator.prototype.access = function(name){
  return this.getter(name).setter(name);
};


Delegator.prototype.getter = function(name){
  var proto = this.proto;
  var target = this.target;
  this.getters.push(name);
  
  proto.__defineGetter__(name, function(){
    return this[target][name];
  });

  return this;
};


Delegator.prototype.setter = function(name){
  var proto = this.proto;
  var target = this.target;
  this.setters.push(name);
  
  proto.__defineSetter__(name, function(val){
    return this[target][name] = val;
  });

  return this;
};
```

## 10、使用 ctx 取代 req 和 res

- 修改 my-application
    
    ```
    const {createServer} = require('http');
    const context = require('./my-context');
    const request = require('./my-request');
    const response = require('./my-response');
    
    module.exports = class Application {
      constructor() {
        this.middleware = [];
        
        this.context = Object.create(context);
        this.request = Object.create(request);
        this.response = Object.create(response);
      }
      
      use(fn) {
        this.middleware.push(fn);
      }
        
      listen(...args) {
        
        const server = createServer(this.callback());
        server.listen(...args);
      }
      
      callback() {
        const fn = compose(this.middleware);
        
        const handleRequest = (req, res) => {
          
          const ctx = this.createContext(req, res);
          const handleResponse = () => respond(ctx);
          fn(ctx).then(handleResponse);
        }
        
        return handleRequest;
      }
      
      
      createContext(req, res) {
        
        const context = Object.create(this.context);
        const request = context.request = Object.create(this.request);
        const response = context.response = Object.create(this.response);
        context.app = request.app = response.app = this;
        context.req = request.req = response.req = req;
        context.res = request.res = response.res = res;
        request.ctx = response.ctx = context;
        request.response = response;
        response.request = request;
        
        return context;
      }
    }
    
    function compose(middleware) {
      return (ctx) => {
        return dispatch(0);
        function dispatch(i) {
          let fn = middleware[i];
          if (!fn) return Promise.resolve();
          return Promise.resolve(fn(ctx, dispatch.bind(null, i + 1)));
        }
      }
    }
    
    function respond(ctx) {
      let body = ctx.body;
      const res = ctx.res;
      if (typeof body === 'object') {
        body = JSON.stringify(body);
        res.end(body);
      } else {
        res.end(body);
      }
    }
    ```
    
- 修改入口文件 index.js 代码
    
    ```
    
    const MyKoa = require('./js/my-application');
    
    const app = new MyKoa();
    
    app.use((ctx, next) => {
      console.log('中间件函数执行了~~~111');
      next();
    })
    app.use((ctx, next) => {
      console.log('中间件函数执行了~~~222');
      
      console.log(ctx.headers);
      
      console.log(ctx.query);
      
      ctx.set('content-type', 'text/html;charset=utf-8');
      
      ctx.body = '<h1>hello myKoa</h1>';
    })
    
    app.listen(3000, err => {
      if (!err) console.log('服务器启动成功了');
      else console.log(err);
    })
    ```
    

> 到这里已经写完了 Koa 主要代码，有一句古话 - 看万遍代码不如写上一遍。 还等什么，赶紧写上一遍吧~  
> 当你能够写出来，再去阅读源码，你会发现源码如此简单~