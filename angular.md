AngularJS源码分析
=================


彻底搞懂setupModuleLoader(window)函数
----------------------------------------
> angularjs源码中setupModuleLoader函数从名称上理解就是设置模块的加载器。 源代码相对比较复杂， 使用了多层级的闭包实现。 下面仅仅抽取重要代码， 忽略掉细节， 先分析下这个多层闭包， 以下为精简后的结构：

```
function setupModuleLoader(window) {
    return ensure(ensure(window, 'angular', Object), 'module', function() {
        var modules = {};
        ...
        return function module(name, requires, configFn) {
            ...
            return ensure(modules, name, function() {
                var invokeQueue = [];
                ...
                var config = invokeLater('$injector', 'invoke');
                var moduleInstance = {...};
                if (configFn) {
                    config(configFn);
                }
                return moduleInstance;
            }
        }
    });
}
```

> 最外层直接返回ensure(ensure(window, 'angular', Object), 'module', function(){}), 那么我们首先应该搞懂ensure是干什么的, 下面是ensure()函数:

```
function ensure(obj, name, factory) {
    return obj[name] || (obj[name] = factory());
}
```

> ensure函数代码非常简单， 但里边包含的信息量确实很丰富的。 这个函数根本上的作用类似于javascript namespace的作用。

> 首先判断obj[name]是否设置， 设置的话直接返回， 如果没有设置， 先设置obj[name] = factory(), 然后返回。

> 结果返回obj上面的一个name属性相对应的对象， factory一般为function， 即obj[name]的构造函数。

> 结合上面的ensure(ensure(window, 'angular', Object), 'module', function(){}), 实际上返回的是window.angular.module, 而，window.angular是一个普通的javascript Object, window.angular.module则是(function(){}); 这种实现也是angular为了不污染全局空间， 所有的对象都在angular命名空间。
