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
> instanceInjector是当调用createInjector时候返回的注入器。 instanceInjector保存了系统中初始化服务。是使用空对象初始化的。 providerInjector维护了未初始化服务。

> 
