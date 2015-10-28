#创建自定义指令
> 这篇文档介绍的是在AngularJS项目中什么时候需要创建自定义指令，以及如何实现自定义指令的。

## 什么是指令(Directives)?

> 从高层次来说，指令就是对DOM元素的标记(比如说属性、元素名称、注释或者CSS类)，指令告诉AngularJS的HTML编译器($compile)为那些DOM元素绑定特定的行为(例如，通过事件监听器)，或者甚至转换DOM元素和其子元素。

> Angular伴随了这些内置指令集合，就像ngBind, ngModel, ngClass。 非常类似你创建的控制器和服务， 你也可以为你的angular应用创建自己的指令集使用。当angular引导应用后， HTML编译器穿越DOM, 通过DOM元素匹配指令。




## 参考链接
1. [Creating Custom Directives](https://docs.angularjs.org/guide/directive)
