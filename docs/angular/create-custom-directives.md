#创建自定义指令
  这篇文档介绍的是在AngularJS项目中什么时候需要创建自定义指令，以及如何实现自定义指令的。

## 什么是指令(Directives)?

  从高层次来说，指令就是对DOM元素的标记(比如说属性、元素名称、注释或者CSS类)，指令告诉AngularJS的HTML编译器($compile)为那些DOM元素绑定特定的行为(例如，通过事件监听器)，或者甚至转换DOM元素和其子元素。

  Angular伴随了这些内置指令集合，就像ngBind, ngModel, ngClass。 非常类似你创建的控制器和服务， 你也可以为你的angular应用创建自己的指令集使用。当angular引导应用后， HTML编译器穿越DOM, 通过DOM元素匹配指令。

> 编译HTML模版到底是什么意思? 对AngularJS来说，编译意思是附加指令到HTML上，使得HTML具有交互性。 我们这里使用术语"compile"，原因是递归处理附加指令借鉴于编译编程语言的源代码编译。

## 指令匹配
  在我们写指令之前，需要了解Angular HTML编译器如何确定什么时候使用特定指令。
  
  类似元素匹配选择器时候使用的术语， 我们说元素在指令为元素声明的一部分时该元素匹配这个指令。

  下面的这个例子， 我们可以说<input>元素匹配ngModel指令。
`<input ng-model="foo">`

  下面的例子，<input>元素也匹配ngModel指令：
`<input data-ng-model="foo">`
  
  而下面的元素匹配person指令：
`<person>{{name}}</person>`

## 规范


  
## 参考链接
1. [Creating Custom Directives](https://docs.angularjs.org/guide/directive)
