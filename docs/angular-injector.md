彻底搞懂Angular Injector
======================

> AngularJS的实现是相当优雅，本篇主要从代码层面分析下injector, 让我们彻底搞懂它。Injector是angular中相对比较核心的一个模块， 框架其他部分基本都是围绕它的。 源代码位于src/auto/injector.js, 里边有一个createInjector的函数。 这个方法揭开整个过程的序幕。

> createInjector在设置了一些属性后， 然后返回一个instanceInjector对象。

### instanceInjector
> instanceInjector是一个对象， 有下面的函数属性:
```
    return {
      invoke: invoke,
      instantiate: instantiate,
      get: getService,
      annotate: createInjector.$$annotate,
      has: function(name) {
        return providerCache.hasOwnProperty(name + providerSuffix) || cache.hasOwnProperty(name);
      }   
    };
```

> 首先我们转向下面这些方法。

1. [annotate](https://github.com/walkerqiao/walkman/blob/master/docs/angular/angular-annotate.md)
2. [invoke](https://github.com/walkerqiao/walkman/blob/master/docs/angular/angular-invoke.md)
3. [get](https://github.com/walkerqiao/walkman/blob/master/docs/angular/angular-injector-get.md)
4. [instantiate](https://github.com/walkerqiao/walkman/blob/master/docs/angular/angular-injector-instantiate.md)

> 然后我们再将所有的串起来， 深入研究下createInjector方法，看看injector如何设置及使用的。[the twin injectors](https://github.com/walkerqiao/walkman/blob/master/docs/angular/angular-create-injector.md)

> 最后我们深入看看使用injector模块是如何加载，以及服务是如何注册的。[loadModules](https://github.com/walkerqiao/walkman/blob/master/docs/angular/angular-load-modules.md)

> 本来想自己写的， 发现这位哥写的超有条例， 就边读代码边翻译了。 [参考链接](http://taoofcode.net/studying-the-angular-injector/)
