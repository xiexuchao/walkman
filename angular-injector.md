彻底搞懂Angular Injector
======================

> AngularJS的实现是相当优雅，本篇主要从代码层面分析下injector, 让我们彻底搞懂它。Injector是angular中相对比较核心的一个模块， 框架其他部分基本都是围绕它的。 源代码位于src/auto/injector.js, 里边有一个createInjector的函数。 这个方法揭开整个过程的序幕。

> createInjector在设置了一些属性后， 然后返回一个instanceInjector对象。

### instanceInjector
