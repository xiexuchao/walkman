彻底搞懂setupModuleLoader(window)函数
----------------------------------------
> angularjs源码中setupModuleLoader函数从名称上理解就是设置模块的加载器。 源代码相对比较复杂， 使用了多层级的闭包实现。 下面仅仅抽取重要代码， 忽略掉细节， 先分析下这个多层闭包， 以下为精简后的结构：

```
function setupModuleLoader(window) {
    return ensure(ensure(window, 'angular', Object), 'module', function() { // 第一层clousure
        var modules = {};
        ...
        return function module(name, requires, configFn) { // 第二层clousure
            ...
            return ensure(modules, name, function() { // 第三层clousure
                var invokeQueue = [];
                ...
                var config = invokeLater('$injector', 'invoke');
                var moduleInstance = {...};
                if (configFn) {
                    config(configFn);
                }
                return moduleInstance;   // 第四层实例返回
            }
        }
    });
}
```
### ensure()函数
> 最外层直接返回ensure(ensure(window, 'angular', Object), 'module', function(){}), 那么我们首先应该搞懂ensure是干什么的, 下面是ensure()函数:

```
function ensure(obj, name, factory) {
    return obj[name] || (obj[name] = factory());
}
```

> ensure函数代码非常简单， 但里边包含的信息量确实很丰富的。 这个函数根本上的作用类似于javascript namespace的作用。

> 首先判断obj[name]是否设置， 设置的话直接返回， 如果没有设置， 先设置obj[name] = factory(), 然后返回。

> 结果返回obj上面的一个name属性相对应的对象， factory一般为function， 即obj[name]的构造函数。

> 结合上面的ensure(ensure(window, 'angular', Object), 'module', function(){}), 实际上返回的是window.angular.module。

> window.angular是一个普通的javascript Object, window.angular.module则是(function(){}); 

> 这种实现也是angular为了不污染全局空间， 所有的对象都在angular命名空间。


### angular的核心代码块

```
(function(window, document, undefined) {
  ...
  bindJQuery(); // 如果使用jQuery, 则绑定jQuery, 否则使用内置的JQLite. 对于大多数angular应用来说，JQLite足够了。

  publishExternalAPI(angular); // 发布外部可用的angular api

  jqLite(document).ready(function() {
    angularInit(document, bootstrap); // 初始化， 启动入口
  });

})(window, document);
```
> 这里我们不纠结每个核心函数的功能， 仅仅围绕本文重心来看， 调用setupModuleLoader的是第二个环节，即publishExternalAPI(angular).

```
function publishExternalAPI(angular){
    extend(angular, {
        ...
    });
    angularModule = setupModuleLoader(window);
    // 尝试获取ngLocale模块， 如果不存在则设置它
    try {
        angularModule('ngLocale');
    } catch (e) {
        angularModule('ngLocale', []).provider('$locale', $LocaleProvider);
    }
    ...
}
```

### angularModule分析
> 从上面的代码可知angularModule = setupModuleLoader(window); 返回的是第一层closure, 

> 就是window.angular.module, 这个是一个函数对象
(function() { return return function module(name, requires, configFn) { // 第二层closure ... } })()

> angularModule(name[, requires][, configFn])是原型方法

1. name: 模块名称
2. requires: 模块的依赖关系
3. configFn: 模块的配置函数

> 继续分析setupModuleLoader， 可以发现， angularModule可以接收一个参数name, 也可以接收两个或三个参数。

> 如果仅仅提供一个参数name, 则相当于getter, 获取给定名称的模块实例。 这个功能实现还是要靠前面简单而复杂的ensure函数做到的。 如果modules[name]存在， 直接返回， 否则执行必要检查， 最终返回moduleInstance. 

> 后者实际上是setter, 需要除了name参数之外， 还需要requires, 而configFn可选。

> 综上所述， angularModule实际上就是模块的getter, setter器。 

> setupModuleLoader()函数就实至名归了， 就是设置模块的加载器， 用于获取或者设置angular.module.{mod_name}的。

### angular的模块加载器实现原理
> 闭包，还是闭包，从setupModuleLoader的函数精简代码，我们可以看到， 最外层闭包里边定义了一个变量modules, 这个就是angular模块管理的"全局缓存"集中管理器， 不管后续的查找getter模块还是setter模块，都是通过这个modules来实现的。

> 利用的是闭包， 内部函数返回后， 这个内部函数依然对其定义的局部空间的变量具有访问权限。 实现了缓存的目的。

```
function setupModuleLoader(window) {
    return ensure(ensure(window, 'angular', Object), 'module', function() { // 第一层closure
        var modules = {};
        ...
        return function module(name, requires, configFn) { // 第二层closure
            if (requires && modules.hasOwnProperty(name)) { // 这里给定了requires, 是setter, 需要将原来的modules[name]重置，然后重新设置。
                modules[name] = null;
            }
            /**
             * modules[name]存在，直接返回 备注getter无需requires
             * modules[name]不存在， setter, 继续下面的逻辑
             */
            return ensure(modules, name, function() { // 第三层closure            
                if (!requires) { // setter必须提供requires, 可以直接是空数组[].
                    throw Error('No module: ' + name);
                }
                var invokeQueue = [];
                var runBlocks = []; 

                var config = invokeLater('$injector', 'invoke');
                
                var moduleInstance = {
                    _invokeQueue: invokeQueue,
                    _runBlocks: runBlocks,
                    requires: requires,
                    name: name,
                    provider: invokeLater('$provide', 'provider'),
                    factory: invokeLater('$provide', 'factory'),
                    service: invokeLater('$provide', 'service'),
                    value: invokeLater('$provide', 'value'),
                    constant: invokeLater('$provide', 'constant', 'unshift'),
                    filter: invokeLater('$filterProvider', 'register'),
                    controller: invokeLater('$controllerProvider', 'register'),
                    directive: invokeLater('$compileProvider', 'directive'),
                    config: config,
                    run: function(block) {
                        runBlocks.push(block);
                        return this;
                    }
                }

                if (configFn) {
                    config(configFn);
                }
                return moduleInstance;   // 第四层实例返回
            }
        }
    });
}
```

### invokeLater()函数分析
```
function invokeLater(provider, method, insertMethod) {
    return function() {
        invokeQueue[insertMethod || 'push']([provider, method, arguments]);
        return moduleInstance;
    }
}
```
> invokeLater函数见名知意， 就是先注册起来，后面调用。
1. provider: 注册函数的名称标识
2. method: 方法
3. insertMethod: 插入方法， 默认push, 猜测可能有unshift, 就是javascript向数组插入新元素的顺序，push向后面附加新元素。

> 该函数返回一个闭包函数，该闭包函数，最终将(provider, method, [arguments]) push|unshift到invokeQueue数组中。这里并不马上执行。

> invokeQueue大概类似[[provider, method, [params]], [...], [...]], 后续调用方法大概类似invokeItem[0]invokeItem[1](invokeItem[2]).

### setupModuleLoader第三层clousure分析
> 该层clousure也是使用了局部变量invokeQueue作为调用队列， 统一管理调用队列， 这个队列保存了调用对象标识符， 方法名，以及调用参数的详细信息， 通过invokeLater返回闭包函数， 便于invoke队列的注册，其中参数由具体属性返回的闭包传入。

1. config: $injector , invoke
2. provider: $provider , provider
3. factory: $provider , factory
4. service: $provider , service
5. value: $provider , value
6. constant: $provider , constant | unshift插入模式
7. filter: $filterProvider , register
8. controller: $controllerProvider , register
9. directive: $compileProvider , register

> 以上均为moduleInstance的属性， 都是闭包函数。 后续可以调用moduleInstance.config(params), 实际完成在invokeQueue中的注册。

### angularModule
```
var /** holds major version number for IE or NaN for real browsers */
    msie              = int((/msie (\d+)/.exec(lowercase(navigator.userAgent)) || [])[1]),
    jqLite,           // delay binding since jQuery could be loaded after us.
    jQuery,           // delay binding
    slice             = [].slice,
    push              = [].push,
    toString          = Object.prototype.toString,

    /** @name angular */
    angular           = window.angular || (window.angular = {}),
    angularModule,
    nodeName_,
    uid               = ['0', '0', '0'];
```

> 我们注意到在angular.js最上面有这么些定义， 这些变量对于angular来说是全局变量(当然只是在闭包中定义的)。

> publishExternalAPI其实仅仅是为angular注册了一些外部可用的API, 以及赋予了angularModule这个模块加载器。真正有意思的还在后面， angularInit()才是重点。

### 总结下setupModule做了些什么
1. 主要是返回一个模块管理器angularModule
2. angularModule是一个闭包， 在这个闭包中定义了一个模块管理的变量modules, 可以通过angularModule进行getter, setter, 实际上就是对modules[mod_name] = moduleInstance;
3. 而moduleInstance包含有很多属性，比如invokeQueue, runBlocks, name, 以及其他一些闭包属性: provider, factory, service等等。

### publishExternalAPI实现如下功能
1. 注册一些api到angular命名空间， 比如extend, copy, uppercase, lowercase等等一些API.
2. 获取angularModule, 即angular.module
3. 注册ngLocal, ng到模块管理器中
