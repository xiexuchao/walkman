彻底搞懂injector -- invoke
===========================

> invoke方法使用注入的参数调用给定函数。

```
    function invoke(fn, self, locals, serviceName) {
      if (typeof locals === 'string') {
        serviceName = locals;
        locals = null;
      }   

      var args = [], 
          $inject = createInjector.$$annotate(fn, strictDi, serviceName),
          length, i,
          key;

      for (i = 0, length = $inject.length; i < length; i++) {
        key = $inject[i];
        if (typeof key !== 'string') {
          throw $injectorMinErr('itkn',
                  'Incorrect injection token! Expected service name as string, got {0}', key);
        }   
        args.push(
          locals && locals.hasOwnProperty(key)
          ? locals[key]
          : getService(key, serviceName)
        );  
      }   
      if (isArray(fn)) {
        fn = fn[length];
      }   

      // http://jsperf.com/angularjs-invoke-apply-vs-switch
      // #5388
      return fn.apply(self, args);
    }   
```
> 首先看看这个方法带有四个参数， angular文档只维护了第一个参数。invoke首先将给定函数进行参数备注。

> 然后每个参数需要验证，确保都是字符串。

> 定位值用于注入。使用如下代码：

```
        args.push(
          locals && locals.hasOwnProperty(key)
          ? locals[key]
          : getService(key, serviceName)
        );  
```

>> 首先看参数是否位于locals， 如果存在直接使用， 否则获取给定key对应的服务。

> 一旦所有参数对应的值都检索出来， 那么使用apply调用fn使用这些参数， 然后返回执行结果。

### getService()

> getService()是invoke()的工作引擎。 这个方法接收一个服务名， 尝试在注册服务中定位服务。

```
    function getService(serviceName, caller) {
      if (cache.hasOwnProperty(serviceName)) {
        if (cache[serviceName] === INSTANTIATING) {
          throw $injectorMinErr('cdep', 'Circular dependency found: {0}',
                    serviceName + ' <- ' + path.join(' <- '));
        }
        return cache[serviceName];
      } else {
        try {
          path.unshift(serviceName);
          cache[serviceName] = INSTANTIATING;
          return cache[serviceName] = factory(serviceName, caller);
        } catch (err) {
          if (cache[serviceName] === INSTANTIATING) {
            delete cache[serviceName];
          }
          throw err;
        } finally {
          path.shift();
        }   
      }   
    }
```
> 当injector被创建的时候， 传入两个参数， cache, factory. 这是非常重要的， 后面我们会分解。 现在我们认为cache就是已经初始化的服务列表， factory就是初始化给定名称服务的方法。

> 如果服务已经存在于缓存中，查看服务是否真正INSTANTIATING占位。 如果是， 这就意味着这个服务在调用另外一个服务的同时，另外一个服务也在同时调用这个服务， 就是循环依赖。 injector不能处理这种情况， 因此抛出一个错误。 否则直接返回服务。

> 如果服务没有在缓存中， 那么需要初始化。

> 首先将服务名字放入path数组中。 这个数组仅仅用于在注入器初始化它们的时候服务路由的。 主要用于错误发生的时候，报告一些有用的信息。

> 缓存中的服务入口设置，开始先占位为INSTANTIATING, 然后factory函数带哦用来创建服务， 结果存放在缓存中， 并返回结果值。

> [instantiate](https://github.com/walkerqiao/walkman/master/docs/angular/angular-injector-instantiate.md)
