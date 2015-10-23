彻底搞懂setupModuleLoader(window)函数
----------------------------------------
> angularjs源码中setupModuleLoader函数从名称上理解就是设置模块的加载器。 源代码相对比较复杂， 使用了多层级的闭包实现。 下面仅仅抽取重要代码， 忽略掉细节， 先分析下这个多层闭包， 以下为精简后的结构：

```
function setupModuleLoader(window) {
    return ensure(ensure(window, 'angular', Object), 'module', function() { // 第一层closure
        var modules = {};
        ...
        return function module(name, requires, configFn) { // 第二层closure
            ...
            return ensure(modules, name, function() { // 第三层closure
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
