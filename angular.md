AngularJS源码分析
=================


彻底搞懂setupModuleLoader(window)函数
----------------------------------------
{
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
}

