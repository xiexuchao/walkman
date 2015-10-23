# 深入理解createInjector

## 先从angularInit说起
> angularInit就是angular应用的初始化入口， 如果提供了ng-app之类的应用属性值，那么angular会自动调用bootstrap()启动angular应用。 angular除了自动引导外，还提供手动引导angular应用的方式， 即直接调用angular.bootstrap(document, ['appName']);

> 因此不管是自动引导还是手动引导， 核心都在于bootstrap, 而bootstrap的参数只有两个, element, modules.
1. element: 可以是document,也可以是具体的div或者其他元素。
2. modules: 一般只能一个，就是应用的名称，可以通过ng-app指定， 也可以通过数组['appName']提供

## bootstrap()分析
> bootstrap核心就一个resumeBootstrapInternal()方法。 在bootstrap中为其他调试工具、研发工具提供了调试的hook入口，如果window.name是以NG_DEFER_BOOTSTRAP开头的，也就是说需要延时引导的， 需要在这些工具完成一定的功能后，主动调用resumeBootstrap()来恢复正常的引导。 其他情况下都是调用resumeBootstrapInternal()方法。

## resumeBootstrapInternal()分析
```
  var resumeBootstrapInternal = function() {
    element = jqLite(element);
    modules = modules || []; // 这个时候modules一般就是['appName']
    modules.unshift(['$provide', function($provide) {
      $provide.value('$rootElement', element);
    }]);  // modules变成 [['$provide', function($provide){}], 'appName']
    modules.unshift('ng'); // modules变成['ng', ['$provide', function($provide){}], 'appName']
    var injector = createInjector(modules);
    injector.invoke(['$rootScope', '$rootElement', '$compile', '$injector',
       function(scope, element, compile, injector) {
        scope.$apply(function() {
          element.data('$injector', injector);
          compile(element)(scope);
        });   
      }]    
    );    
    return injector;
  };
```

### createInjector(modules)分析
> 兜了一大圈， 终于切入主题了。 下面我们详细分析下createInjector()吧。

```
function createInjector(modulesToLoad) {
    var INSTANTIATING = {},
      providerSuffix = 'Provider',
      path = [],
      loadedModules = new HashMap(),
      providerCache = {
          $provide: {
              provider: supportObject(provider),
              factory: supportObject(factory),
              service: supportObject(service),
              value: supportObject(value),
              constant: supportObject(constant),
              decorator: decorator
          }
      },
      providerInjector = createInternalInjector(providerCache, function() {
          throw Error("Unknown provider: " + path.join(' <- '));
      }),
      instanceCache = {},
      instanceInjector = (instanceCache.$injector =
          createInternalInjector(instanceCache, function(servicename) {
              var provider = providerInjector.get(servicename + providerSuffix);
              return instanceInjector.invoke(provider.$get, provider);
          }));

    forEach(loadModules(modulesToLoad), function(fn) { instanceInjector.invoke(fn || noop); });

    return instanceInjector;
    // 此处略去千万行
}
```

#### 重要变量
1. providerCache
2. providerInjector
3. instanceCache
4. instanceInjector

#### 重要方法
1. createInternalInjector
2. loadModules


#### createInternalInjector()
```
function createInternalInjector(cache, factory) {
  ...
  
  return {
    invoke: invoke,
    instantiate: instantiate,
    get: getService,
    annotate: annotate
  };
}
```
> createInternalInjector函数返回操作集合对象：包括invoke, instantiate, get, annotate方法。

##### 1. getService(serviceName): 获取给定服务
```
function getService(serviceName) {
  if (typeof serviceName !== 'string') {
    throw Error('Service name expected');
  }
  if (cache.hasOwnProperty(serviceName)) {
    if (cache[serviceName] === INSTANTIATING) {
      throw Error('Circular dependency: ' + path.join(' <- '));
    }
    return cache[serviceName];
  } else {
    try {
      path.unshift(serviceName);
      cache[serviceName] = INSTANTIATING;
      return cache[serviceName] = factory(serviceName);
    } finally {
      path.shift();
    }
  }
}
```
> angular通过缓存集中管理服务，获取服务的单个实例， 实现服务的单例模式。 其中provider在获取服务之前已经缓存起来， 因此要么获取到，要么抛出异常；而对于实例对象服务， 提供了创建服务的factory, 如果缓存中有， 则直接获取， 否则需要先通过factory产生一个实例，然后放入缓存并返回。

##### 2. annotate()
```
var FN_ARGS = /^function\s*[^\(]*\(\s*([^\)]*)\)/m;
var FN_ARG_SPLIT = /,/;
var FN_ARG = /^\s*(_?)(\S+?)\1\s*$/;
var STRIP_COMMENTS = /((\/\/.*$)|(\/\*[\s\S]*?\*\/))/mg;
function annotate(fn) {
  var $inject,
      fnText,
      argDecl,
      last; 

  if (typeof fn == 'function') {
    if (!($inject = fn.$inject)) {
      $inject = []; 
      fnText = fn.toString().replace(STRIP_COMMENTS, '');
      argDecl = fnText.match(FN_ARGS);
      forEach(argDecl[1].split(FN_ARG_SPLIT), function(arg){
        arg.replace(FN_ARG, function(all, underscore, name){
          $inject.push(name);
        });   
      });   
      fn.$inject = $inject;
    }
  } else if (isArray(fn)) {
    last = fn.length - 1;
    assertArgFn(fn[last], 'fn');
    $inject = fn.slice(0, last);
  } else {
    assertArgFn(fn, 'fn', true);
  }
  return $inject;
}
```

##### 3. invoke(): 
```
function invoke(fn, self, locals){
  var args = [], 
      $inject = annotate(fn),
      length, i,
      key;  

  for(i = 0, length = $inject.length; i < length; i++) {
    key = $inject[i];
    args.push(
      locals && locals.hasOwnProperty(key)
      ? locals[key]
      : getService(key)
    );    
  }
  if (!fn.$inject) {
    // this means that we must be an array.
    fn = fn[length];
  }     


  // Performance optimization: http://jsperf.com/apply-vs-call-vs-invoke
  switch (self ? -1 : args.length) {
    case  0: return fn(); 
    case  1: return fn(args[0]);
    case  2: return fn(args[0], args[1]);
    case  3: return fn(args[0], args[1], args[2]);
    case  4: return fn(args[0], args[1], args[2], args[3]);
    case  5: return fn(args[0], args[1], args[2], args[3], args[4]);
    case  6: return fn(args[0], args[1], args[2], args[3], args[4], args[5]);
    case  7: return fn(args[0], args[1], args[2], args[3], args[4], args[5], args[6]);
    case  8: return fn(args[0], args[1], args[2], args[3], args[4], args[5], args[6], args[7]);
    case  9: return fn(args[0], args[1], args[2], args[3], args[4], args[5], args[6], args[7], args[8]);
    case 10: return fn(args[0], args[1], args[2], args[3], args[4], args[5], args[6], args[7], args[8], args[9]);
    default: return fn.apply(self, args);
  }
}
```
##### 4. instantiate()
```
function instantiate(Type, locals) {
  var Constructor = function() {},
      instance, returnedValue;

  // Check if Type is annotated and use just the given function at n-1 as parameter
  // e.g. someModule.factory('greeter', ['$window', function(renamed$window) {}]);
  Constructor.prototype = (isArray(Type) ? Type[Type.length - 1] : Type).prototype;
  instance = new Constructor();
  returnedValue = invoke(Type, instance, locals);

  return isObject(returnedValue) ? returnedValue : instance;
}
```

#### loadModules
```
  function loadModules(modulesToLoad){
    var runBlocks = [];
    forEach(modulesToLoad, function(module) {
      if (loadedModules.get(module)) return;
      loadedModules.put(module, true);
      if (isString(module)) {
        var moduleFn = angularModule(module);
        runBlocks = runBlocks.concat(loadModules(moduleFn.requires)).concat(moduleFn._runBlocks);

        try {
          for(var invokeQueue = moduleFn._invokeQueue, i = 0, ii = invokeQueue.length; i < ii; i++) {
            var invokeArgs = invokeQueue[i],
                provider = invokeArgs[0] == '$injector'
                    ? providerInjector
                    : providerInjector.get(invokeArgs[0]);

            provider[invokeArgs[1]].apply(provider, invokeArgs[2]);
          }
        } catch (e) {
          if (e.message) e.message += ' from ' + module;
          throw e;
        }
      } else if (isFunction(module)) {
        try {
          runBlocks.push(providerInjector.invoke(module));
        } catch (e) {
          if (e.message) e.message += ' from ' + module;
          throw e;
        }
      } else if (isArray(module)) {
        try {
          runBlocks.push(providerInjector.invoke(module));
        } catch (e) {
          if (e.message) e.message += ' from ' + String(module[module.length - 1]);
          throw e;
        }
      } else {
        assertArgFn(module, 'module');
      }
    });
    return runBlocks;
  }
```