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

> 当angular创建injector的时候， 实际上创建了两个注入器providerInjector, instanceInjector. createInternalInjector()的第一个参数为查询实例(简单对象)的缓存。 第二个参数为工厂方法。 工厂方法用于在服务不在缓存的时候创建服务的。

### instanceInjector
> instanceInjector是当调用createInjector时候返回的注入器。 
>> instanceInjector保存了系统中初始化服务。是使用空对象初始化的。 providerInjector维护了未初始化服务。

> 看看instance注入器的工厂函数， 当我们仕途获取一个还没有初始化当服务的时候，我们将在providerInjector中使用名字servicename + providerSuffix的服务名。 providerSuffix是Provider. 当有的话， 就调用provider对象的$get函数。

> 这里有两个假设前提：

1. 所有的存储在providerInjector的服务都是以Provider为后缀的。
2. 所有存储在providerInjector里边的服务都是使用$get函数获取的对象。

### providerInjector
> 让我们看看providerInjector是如何设置的。

> providerInjector的缓存都是使用一个服务-- $provide来初始化的：

```
  providerCache = {
    provider: supportObject(provider),
    factory: supportObject(factory),
    service: supportObject(service),
    value: supportObject(value),
    constant: supportObject(constant),
    decorator: decorator
  }
```

> providerInject上面的$provide服务默认总是可用的。 其他的服务都是通过这个服务注册的。

> supportObject()方法

```
  function supportObject(delegate) {
    return function(key, value) {
      if (isObject(key)) {
        forEach(key, reverseParams(delegate));
      } else {
        return delegate(key, value);
      }
    };
  }
```

> 这是非常普通的javascript模式。 一个函数捕获一个变量(这里是delegate)， 然后返回另外一个函数。

> 从supportObject返回的函数会应用delegate到函数参数上， 或者传入的对象，将delegate应用到传入对象的每个字段上面。

> 传入的delegate在函数里边处理provider、factory、service、value或者constant的创建。 它们每个的工作稍微有点不同， 但是都是向providerCache添加一个命名为*Provider的服务，并且具有一个$get成员方法。

#### provider
```
  function provider(name, provider_) {
    assertNotHasOwnProperty(name, 'service');
    if (isFunction(provider_) || isArray(provider_)) {
      provider_ = providerInjector.instantiate(provider_);
    }   
    if (!provider_.$get) {
      throw $injectorMinErr('pget', "Provider '{0}' must define $get factory method.", name);
    }   
    return providerCache[name + providerSuffix] = provider_;
  }
```

> 如果给定的provider是函数或者数组，那么这个provider被调用。 然后我们验证确保provider具有$get属性， 然后该对象被添加到providerCache中去。

#### factory
```
  function factory(name, factoryFn, enforce) {
    return provider(name, {
      $get: enforce !== false ? enforceReturnValue(name, factoryFn) : factoryFn
    }); 
  }
```

> factory仅仅是创建一个带有$get属性的provider, 这个属性指向传入的factoryFn.

#### service
```
  function service(name, constructor) {
    return factory(name, ['$injector', function($injector) {
      return $injector.instantiate(constructor);
    }]);
  }
```
> 服务service创建了一个factory. 该方法是备注了去检索$injector对象的。 注入器用于初始化传入service()的constructor的实例。

#### value
```function value(name, val) { return factory(name, valueFn(val), false); }```

> value创建使用返回传入值val的函数创建一个工厂。

#### constant
```
  function constant(name, value) {
    assertNotHasOwnProperty(name, 'constant');
    providerCache[name] = value;
    instanceCache[name] = value;
  }
```

> constant打破了$get和provider后缀的规则。 给定的value直接同时设置到provider和instance缓存， 因为初始化constant没有任何需要处理的。

> 那么为什么它们要实现这样的双胞胎注入器解决方案呢? 它们不能把初始化和未初始化的服务合并到一个注入器吗? 它们到名字已经通过provider后缀在未初始化的服务上做了区别。

> 其中一个原因就是隐藏功能性。 假设你想要使用注入器直接注册一个服务。 如果试图这样做， 你可能会这样做:

```
var injector = angular.injector();
injector.get('$provider').value('myValue', 3.14);
// Error: [$injector:unpr] Unknown provider: $providerProvider <- $provider
```

> 你只能访问instanceInjector. 当在providerInjector查找的时候， 那么就会在名字上面添加Provider后缀。 那么就会尝试查找$providerProvider, 那么当然不存在了。

> 如果我们不能访问$provider, 那么我们怎么能使用注入器注册我们自己的服务呢？

> 这些都是通过模块完成的。 angular中所有的服务设置必须附加到特定到模块， 这也是唯一的使用注册器注册服务的方式。

> [loadModules](https://github.com/walkerqiao/walkman/blob/master/docs/angular/angular-load-modules.md)
