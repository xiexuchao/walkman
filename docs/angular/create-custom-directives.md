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

  Angular规范元素的标签和属性名来确定哪些元素匹配哪些指令。我们一般通过指令的大小写敏感驼峰规范名来引用指令。(eg: ngModel). 然而， html是大小写不敏感的， 因此我们在dom中引用指令通过小写形式， 一般使用dom元素中划线属性(eg: ng-model).
  
  规范化处理如下:
  1. 去掉元素/属性前面的'x-', 'data-x'。
  2. 将:, -或_分割的名称转换为驼峰状。


  例如， 下面的各种形式都是等价的，都是匹配ngBind指令的。
```
// index.html
<div ng-controller="Controller">
  Hello <input ng-model="name"> <hr />
  <span ng-bind="name"></span> <br />
  <span ng:bind="name"></span> <br />
  <span ng_bind="name"></span> <br />
  <span data-ng-bind="name"></span> <br />
  <span x-ng-bind="name"></span> <br />
</div>
```

```
// script.js
angular.module('docsBindExample', [])
.controller('Controller', ['$scope', function($scope) {
  $scope.name = 'Max Karl Ernst Ludwig Planck (...)';
}]);
```

> #### 最佳实践:
> 优先使用中划线分割格式(eg: ng-bind -> ngBind). 如果你要使用HTML验证工具， 你可以使用data-前置版本替代。(eg: data-ng-bind -> ngBind). 接收上面展示的其他形式主要是历史遗留问题， 因此建议避免使用它们。

## 指令类型
  $compile能基于元素名称、属性、类名、以及注释匹配指令。
  
  angular提供的所有指令都匹配属性名、标签名、注释或者类名。 下面演示各种可以在模版中引用的不同形式的指令(这个例子为myDir指令)。

```
<my-dir></my-dir>
<span my-dir="exp"></span>
<!-- directive: my-dir exp -->
<span class="my-dir: exp;"></span>
```
  
> ### 最佳实践：
> 优先通过标签名和属性来使用指令， 其次考虑注释和类名。这样做容易确定哪个元素匹配哪些指令。

> ### 最佳实践:
> 注释指令通常用于那些DOM API限制了创建指令的能力，因为需要跨越多个元素。(eg: 在`<table>`元素中)。 AngularJS 1.2介绍了ng-repeat-start和ng-repeat-end作为较好的解决这类问题的方式。鼓励开发者在必要的时候使用注释指令。

## 文本和属性绑定
  在编译器匹配文本和属性的编译过程中，使用$interpolate服务来查看是否嵌套表达式。这些表达式被注册为$watch, 并且作为正常$digest循环的一部分更新。插值(interpolation)类似：`<a ng-href="img/{{username}}.jpg">Hello {{username}}</a>`
  
## ngAttr属性绑定
  Web浏览器有时候会挑剔什么属性值是有效的。
  例如， 考虑下面的模版:
```
<svg>
  <circle cx="{{cx}}"></circle>
</svg>
```
  我们期望angular能绑定这个， 但是当我们检查控制面板的时候会发现如下错误`Error: Invalid value for attribute cx="{{cx}}"`, 因为svg dom api的限制， 不能简单写cx="{{cx}}", 使用ng-attr-cx, 可以解决这个问题。
  
  如果属性绑定是使用ngAttr前缀前置的(示例ng-attr-), 然后在绑定它的过程中，会应用相应无前缀属性。 这样就允许你绑定你想要的属性， 急切应付浏览器处理(比如svg元素的circle[cx]属性)。 当使用ngAttr的时候，$interpolate使用所有标签或者什么标签都不适用， 因此如果差值字符串的任何表达式结果为undefined, 属性被删除并不添加到元素上。
  
  例如， 我们可以通过下面方式修复上面的问题:
```
<svg>
  <circle ng-attr-cx="{{cx}}"></circle>
</svg>
```

  如果希望修改驼峰状属性(svg元素有合法驼峰状属性), 比如svg的viewBox属性， 那么他可以使用下划线来表示这个属性来绑定它的自然驼峰状属性。
  
  例如， 要绑定viewBox, 我们可以写: `<svg ng-attr-view_box="{{viewBox}}"></svg>`

## 创建指令

## 参考链接
1. [Creating Custom Directives](https://docs.angularjs.org/guide/directive)
