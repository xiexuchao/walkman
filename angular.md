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
>> 最外层直接返回ensure(ensure(window, 'angular', Object), 'module', function(){}), 那么我们首先应该搞懂ensure是干什么的。
>> ensure()函数分析
