# Connect源码分析及使用介绍

  Connect是TJ Holowaychuck的力作， 也就是express的最初作者。 这两个Node插件都是node模块中依赖在前10的。Connect是作者后来从express中抽取出来的模块，同样依赖也名列前茅。
  
  Connect短小精悍，偏向基础设施的框架。Connect做的事情很简单，就是把一系列组件连接到一起，然后让http请求流经这些组件。 这些被connect串联起来的组件称为中间件(middleware)。 在connect中， http请求的处理流程被划分为一个个小片段， 每个小片段代表一项处理任务(如:请求body解析、session的维护等)， 由一个中间件负责， 前后片段之间靠request, response等对象传递中间数据。 connect逐渐对这些处理细节并不关心， 只知道将请求从一个中间件导给下一个中间件。
  
  下面我们先看看connect官网的hello world例子吧:
```
var connect = require('connect');
var http = require('http');

var app = connect();

var compression = require('compression');
app.use(compression);

var cookieSession = require('cookie-session');

app.use(cookieSession({
  keys: ['secret1', 'secret2']
});

var bodyParser = require('body-parser');
app.use(bodyParser.urlencoded());

app.use(function(req, res) {
  res.end('Hello from Connect!\n');
});

http.createServer(app).listen(3000);
```

  很简单，很清晰的一个链式调用代码。 风格和express的很像，不，应该说express的风格和它很像，总的有个先来后到，是吧。
  
## connect入口代码
  connect入口代码在connect模块的index.js中， 代码非常少， 带注释也不过300行。 主要定义了一个createServer函数。该函数被赋给module.exports, 当成connect模块暴露出去， 即对应于例子中的connect()函数。
  createServer函数的代码如下:
```
function createServer() {
  function app(req, res, next){ app.handle(req, res, next); }
  merge(app, proto);
  merge(app, EventEmitter.prototype);
  app.route = '/';
  app.stack = [];
  return app;
}
```
  这个函数主要定义了一个app的函数，然后使用merge将proto和EventEmitter.prototype相应属性合并到app上。让app具有proto和EventEmitter的特性。 类似其他语言中的mixin， 实现多继承， 但是需要注意，proto的属性会覆盖app原来的同名属性，EventEmitter的属性同样会覆盖掉前面具有的同名属性。
  
  这个时候app就是继承了proto和EventEmitter属性的对象。 就像app.handle, app.use是通过proto过来的。 app这个函数相对比较简单，就是调用app.handle来处理http请求。 随后，这个函数被当作回调函数传递给node原生的http.createServer()方法，成为http处理的源头。
  
  注意这里的app没有通过new关键字返回，因此不是由app作为构造函数而构建出来的一个纯粹的js对象，它依然是一个function, 所以它是可以作为回调函数传递给http.createServer方法的。
  
  此外，connect的createServer可以链式接受任意数量的中间件对象作为参数， 在函数的最后会逐一加载这些传递进来的中间件。
  
## 中间件的加载
  connect的最大灵活性在于它的中间件机制。 connect的中间件加载代码是app.use方法定义的。 use方法代码如下:
```
proto.use = function use(route, fn) {
  var handle = fn;
  var path = route;

  // default route to '/'
  if (typeof route !== 'string') {
    handle = route;
    path = '/';
  }

  // wrap sub-apps
  if (typeof handle.handle === 'function') {
    var server = handle;
    server.route = path;
    handle = function (req, res, next) {
      server.handle(req, res, next);
    };
  }

  // wrap vanilla http.Servers
  if (handle instanceof http.Server) {
    handle = handle.listeners('request')[0];
  }

  // strip trailing slash
  if (path[path.length - 1] === '/') {
    path = path.slice(0, -1);
  }

  // add the middleware
  debug('use %s %s', path || '/', handle.name || 'anonymous');
  this.stack.push({ route: path, handle: handle });

  return this;
};
```
### app.use(route, fn)
  * route: 中间件所使用的请求url的模式pattern, 默认为"/". connect会按照加载顺序，逐一执行pattern与请求url匹配的中间件处理函数。
  * fn: 中间件处理函数， 有两种形式:
  * 当然use可以用来包装子应用及普通的http.Server实例。

  1. function(req, res, next): 正常处理函数
  2. function(err, req, res, next): 异常处理函数

  * req: http模块的request对象
  * res: http模块的response对象
  * next: 触发后续流程的回调函数，带一个err参数。通过传递给next的err参数，告诉框架当前中间件处理出现异常。如果err为空， 则会按顺序执行后面的正常处理函数， 忽略异常处理函数；相反，如果err非空，则会按照顺序执行后续的异常处理函数，而忽略掉正常处理函数。
  
  在connect中，请求的处理流程是一个异步过程，fn的返回并不代表处理流程的结束，所以在这里需要用next回调的形式通知框架执行后续的流程。 但如果某一个中间件函数认为请求的流程到它那里已经处理完毕，无需再执行后面的流程，则可以直接返回，而不用再调用next()回调函数了(包括正常和异常处理函数)。如果请求遍历完中间件列表后人在调用next()函数，connect则会认为这个请求没有中间件认领。这时，如果next的err参数非空，则会给页面返回500错误，标识服务器出现了内部错误；如果err为空，则返回404错误，即访问的资源不存在。

  中间件之间没有直接的依赖关系，因此request, response就成为这些中间件传递信息的唯一桥梁。 前面的中间件处理完毕后，把结果附加到request或response之上，后面的中间件便可以从中获取到这些数据。 所以，中间件的加载顺序在connect中就非常重要，必须将被依赖的中间件放在依赖它的模块之前。 比如说cookie中间件应该放在session中间件之前，因为session id是通过cookie来传递的。
  
```
proto.handle = function handle(req, res, out) {
  var index = 0;
  var protohost = getProtohost(req.url) || '';
  var removed = '';
  var slashAdded = false;
  var stack = this.stack;

  // final function handler
  var done = out || finalhandler(req, res, {
    env: env,
    onerror: logerror
  });

  // store the original URL
  req.originalUrl = req.originalUrl || req.url;

  function next(err) {
    if (slashAdded) {
      req.url = req.url.substr(1);
      slashAdded = false;
    }

    if (removed.length !== 0) {
      req.url = protohost + removed + req.url.substr(protohost.length);
      removed = '';
    }

    // next callback 取出栈中下一个处理函数
    var layer = stack[index++];

    // all done
    if (!layer) { // 已经到栈底，交给默认的处理方法defer来处理
      defer(done, err);
      return;
    }

    // route data
    var path = parseUrl(req).pathname || '/';
    var route = layer.route;

    // skip this layer if the route doesn't match
    if (path.toLowerCase().substr(0, route.length) !== route.toLowerCase()) { // 不匹配pattern直接跳过该处理函数
      return next(err);
    }

    // skip if route match does not border "/", ".", or end
    var c = path[route.length];
    if (c !== undefined && '/' !== c && '.' !== c) {
      return next(err);
    }
    // trim off the part of the url that matches the route
    if (route.length !== 0 && route !== '/') {
      removed = route;
      req.url = protohost + req.url.substr(protohost.length + removed.length);

      // ensure leading slash
      if (!protohost && req.url[0] !== '/') {
        req.url = '/' + req.url;
        slashAdded = true;
      }
    }

    // call the layer handle 调用具体的处理函数
    call(layer.handle, route, err, req, res, next);
  }

  next();
};
```
  上面代码中layer就是栈中保存的每一个处理函数的对象信息，结构为{route: route, handle: fn}. 
```
function call(handle, route, err, req, res, next) {
  var arity = handle.length;
  var error = err;
  var hasError = Boolean(err);

  debug('%s %s : %s', handle.name || '<anonymous>', route, req.originalUrl);

  try {
    if (hasError && arity === 4) {
      // error-handling middleware
      handle(err, req, res, next);
      return;
    } else if (!hasError && arity < 4) {
      // request-handling middleware
      handle(req, res, next);
      return;
    }
  } catch (e) {
    // replace the error
    error = e;
  }

  // continue
  next(error);
}
```
  从call方法知道调用具体的handle的时候，会做一些判断。
  * 判断是否传入err, 即上一个中间件是否传入错误，如果传入错误，当前的处理函数又接受四个参数，那么就使用该处理函数进行异常处理。进行异常流处理。
  * 如果上一个中间件没有错误，而且当前处理函数接受参数小于4，那么接着正常流执行。
  * 如果上一个中间件有错误，但当前还不是异常处理函数，即接受参数不为4， 那么将异常推后到下一个中间件中。
  * 而处理到中间件末尾，交由默认的defer来处理。

```
    if (!layer) {
      defer(done, err);
      return;
    }
    
var defer = typeof setImmediate === 'function'
  ? setImmediate
  : function(fn){ process.nextTick(fn.bind.apply(fn, arguments)) }
  
  var done = out || finalhandler(req, res, {
    env: env,
    onerror: logerror
  });
```

  最后交由finalhandler模块来处理， 猜测应该是处理error等异常流的情况， 暂时不深入到finalhandler模块研究，后续在深入进去看看。
  
## 总结
  到现在为止，对connect的流程及基本作用基本了解， 在具体项目中实施起来也有谱了。
  
  正常流处理都需要接受2~3个参数作为处理函数的参数， 而异常处理函数需要接受4个参数， 第一个参数即上一个中间件给出的err信息，即next传入的错误信息。
  
## 应用举例

### 什么也不做的中间件
```
  function uselessMiddleware(req, res, next){
    next();
  }
  
  app.use(uselessMiddleware);
```

### 异常处理中间件
```
  function errorHandleMiddleware(err, req, res, next){
    if(err) {
      // handle error
    }
    
    // 是否需要继续正常流呢?? 需要就next()
  }
  app.use(errorHandleMiddleware);
```

### 抛出异常的中间件
```
  function throwErrorMiddleware(req, res, next) {
    // do something useful
    next("here is some error msg");
  }
  
  app.use(throwErrorMiddleware);
```


### 代码集中到一起如下
```
  app.use(uselessMiddleware);
  app.use(throwErrorMiddleware); // 抛出异常
  app.use(errorHandleMiddleware); // 处理上一部中的异常
  ....
```


## 参考链接
  * [connect源码分析--基础架构](https://cnodejs.org/topic/4fb79b0e06f43b56112b292c)
  * [A short guide to Connect Middleware](http://stephensugden.com/middleware_guide/)
