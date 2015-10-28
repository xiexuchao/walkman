彻底搞懂injector -- loadModules
================================

> 模块加载通过下面的代码实现的:

```
  function loadModules(modulesToLoad) {
    assertArg(isUndefined(modulesToLoad) || isArray(modulesToLoad), 'modulesToLoad', 'not an array');
    var runBlocks = [], moduleFn;
    forEach(modulesToLoad, function(module) {
      if (loadedModules.get(module)) return;
      loadedModules.put(module, true);

      function runInvokeQueue(queue) {
        var i, ii;
        for (i = 0, ii = queue.length; i < ii; i++) {
          var invokeArgs = queue[i],
              provider = providerInjector.get(invokeArgs[0]);

          provider[invokeArgs[1]].apply(provider, invokeArgs[2]);
        }
      }

      try {
        if (isString(module)) {
          moduleFn = angularModule(module);
          runBlocks = runBlocks.concat(loadModules(moduleFn.requires)).concat(moduleFn._runBlocks);
          runInvokeQueue(moduleFn._invokeQueue);
          runInvokeQueue(moduleFn._configBlocks);
        } else if (isFunction(module)) {
            runBlocks.push(providerInjector.invoke(module));
        } else if (isArray(module)) {
            runBlocks.push(providerInjector.invoke(module));
        } else {
          assertArgFn(module, 'module');
        }
      } catch (e) {
        if (isArray(module)) {
          module = module[module.length - 1];
        }
        if (e.message && e.stack && e.stack.indexOf(e.message) == -1) {
          // Safari & FF's stack traces don't contain error.message content
          // unlike those of Chrome and IE
          // So if stack doesn't contain message, we create a new string that contains both.
          // Since error.stack is read-only in Safari, I'm overriding e and not e.stack here.
          /* jshint -W022 */
          e = e.message + '\n' + e.stack;
        }
        throw $injectorMinErr('modulerr', "Failed to instantiate module {0} due to:\n{1}",
                  module, e.stack || e.message || e);
      }
    });
    return runBlocks;
  }
```

> 该函数返回runBlocks数组--在加载之后即将被调用的函数集合。

> AngularJS中有三种方法定义模块。

###  方法1: 通过直接指定runBlock：
```
angular.module(function($httpProvider) {
  console.log('Module is now running');
});
```

### 方法2: 通过传入数组
```
angular.module(['$httpProvider', function($httpProvider) {
  ...
});
```

> 上面两种方法， 模块运行都是通过下面的一行代码实现:

```
runBlocks.push(providerInjector.invoke(module));
```

> 模块函数通过providerInjector被调用。 这就意味着我们可以注入难以琢磨的$provide对象到函数里边。 然后可用于直接注册服务。

```
var injector = angular.injector([function($provide) {
  $provide.value('anInterestingFact', 'An ant has two stomachs. One for its own food and another for food to share');
}]);

injector.get('anInterestingFact', [dependency]);
```

> 如果模块已经以这种方式定义了的话， 下面代码可运行:

```
moduleFn = angularModule(module);
runBlocks = runBlocks.concat(loadModules(moduleFn.requires)).concat(moduleFn._runBlocks);

for(invokeQueue = moduleFn._invokeQueue, i = 0, ii = invokeQueue.length; i < ii; i++) {
  var invokeArgs = invokeQueue[i],
      provider = providerInjector.get(invokeArgs[0]);
      
  provider[invokeArgs[1]].apply(provider, invokeArgs[2]);
}
```

> 首先我们通过angularModule检索模块对象。 这个函数是在injector外边定义的， 仅仅是angular.module函数的别名。 当模块设置的时候，产生了两个数组_runBlocks, _invokeQueue. （上面代码设置模块不是在injector模块内的， 因此我不会在此刻查找它。）

## _runBlocks
>  这个数组是在使用angular.module('myModule').run块指定的。 这个代码需要在模块加载的时候立即调用。 因此我们将它和我们的runBlocks连接起来。

## _invokeQueue
> _invokeQueue是使用每个服务产生的， 这些服务是通过熟悉的angular.module('myModule').controller, angular.module('myModule').directive等调用添加进去的. 这个队列中的每个元素都是一个具有三个元素的数组。 第一个元素为将调用服务的provider, 第二个元素为使用的provider方法， 第三个元素是传入服务的参数数组。

> 让我们看一个例子来了解它们是如何协作的。 假设我们通过下面方式设置模块:

```
angular.module('aNiceModule', [])
.run(function() {
  console.log('running ...);
})
.controller('aNiceController', function($scope) {
  console.log('setting up controller');
});
```

> 我们可以调用angular.bootstrap(window.document.body, ['aNiceModule'])来揭开模块加载的序幕。 aNiceModule模块将在_runBlocks具有一个入口。`function() {console.log('running ...);}`, 在_invokeQueue中有一个入口：`['$controllerProvider', 'register', ['aNiceModule', function($scope) {...}]]`.


> $controllerProvider是angular内置provider, 允许注册控制器。 调用register会将服务添加到可用的控制列表中。 注意不会向injector缓存添加任何东西。 这就意味着控制器不能被注入到服务中。 如果你确实需要访问控制器(在单元测试的时候)， 你将注入$controller provider, 然后通过调用get(controllerName)来获取控制器。

## 下面我们来创建一个factory服务
```
angular.module('aNiceModule', [])
.factory('aNiceFactory', function() {
  console.log('setting up factory');
  return {};
});
```

> 当我们加载这个模块， _invokeQueue将会有下面的入口: `['$provide', 'factory', ['aNiceModule', function(){...}]]`. 这个服务是通过使用内置$providerProvider注册的。 这个服务然后可以用到其他需要它的服务中去。
