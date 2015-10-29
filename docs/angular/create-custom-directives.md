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
  首先让我们讨论下注册指令的API. 非常类似控制器， 指令也是在模块上注册的。 要注册一个指令， 可以使用module.directive API. module.directive接收标准化指令名称， 后面跟着工厂方法。 这个工厂方法必须返回一个对象， 使用不同的选项告诉$compile当匹配到指令的时候行为如何。
  
> ### 最佳实践:
> 优先使用定义对象， 其次是返回一个函数。

  我们将一些指令的通用例子， 然后深入到不同到选项和编译处理过程中。
  
> ### 最佳实践:
> 为了避免和将来标准冲突， 最好为你自己的指令定义个前缀。 例如，如果你创建一个<carousel>指令， 那么在html7中可能会有问题， 因为那里有这个元素。 两三个字符前缀即可。 同时，你不要使用ng或者可能和angular将来版本的指令相冲突的前缀字符串。

  下面的例子中， 我们使用前缀my(eg: myCustomer)
  
## 模版扩展指令
  假设你有一片展示客户信息的模版。 这个模块在代码中会多次重复。 修改一个地方， 其他多个地方都会修改。 这就是使用指令扩展模版的好机会。
  
  让我们创建一个指令简单使用静态模版替换它的内容。
```
// script.js
angular.module('docsSimpleDirective', [])
.controller('Controller', ['$scope', function($scope){
  $scope.customer = {
    name: 'Naomi',
    address: '1600 Ampitheatre'
  };
}])
.directive('myCustomer', function(){
  return {
    template: 'Name: {{customer.name}} Address: {{customer.address}}'
  };
});
```

```
/** index.html **/
<div ng-controller="Controller">
  <div my-customer></div>
</div>
```

  注意我们已经绑定的这个指令。 在$compile编译和链接<div my-customer></div>后， 将尝试匹配元素子元素指令。 这就是说可能和其他指令组合起来。 后面的例子我们将会看到如何实现的。
  
> ### 最佳实践:
> 除非你的模版非常的小，否则一般更推荐将模版分离到单个的html文件中， 然后使用templateUrl选项来定位模版。

  如果你熟悉ngInclude指令，templateUrl和它工作原理类似。 下面使用templateUrl来替代实现的:

```
angular.module('docsSimpleDirective', [])
.controller('Controller', ['$scope', function($scope){
  $scope.customer = {
    name: 'Naomi',
    address: '1600 Ampitheatre'
  };
}])
.directive('myCustomer', function(){
  return {
    templateUrl: 'path/to/my-customer.html'
  };
});
```

  templateUrl也可以是一个函数， 返回要加载使用到指令的html模版的URL. angular调用templateUrl带有两个参数: 指令调用的元素element和元素相关的属性attr.

> ### 注意:
> 当前你还没有能力从templateUrl访问作用域scope变量, 因为模版请求是在作用域初始化之前发生的。

```
angular.module('docsSimpleDirective', [])
.controller('Controller', ['$scope', function($scope){
  $scope.customer = {
    name: 'Naomi',
    address: '1600 Ampitheatre'
  };
}])
.directive('myCustomer', function(){
  return {
    templateUrl: function(elem, attr) {
      return 'customer-' + attr.type + '.html';
    }
  };
});
```

```
<div ng-controller="Controller">
  <div my-customer type="name"></div>
  <div my-customer type="address"></div>
</div>
```

> ### 注意:
> 当你创建指令的时候，默认是限制在属性和元素上的。 为了创建在类名和注释上面的指令，需要使用restrict选项。

  restrict选项一般设置值有如下几种:
  * 'A': 仅仅匹配属性名称
  * 'E': 仅仅匹配元素名
  * 'C': 仅仅匹配类名
  * 'M': 仅仅匹配注释指令

  这些指令可以按需要组合: 'AEC': 匹配元素、类名或属性名。
  
> 那么什么时候我们使用属性指令，什么时候使用元素指令呢? 
> 当创建控制模版的组件指令的时候使用元素指令。 通常场景是当你为模版创建特定领域语言的时候。 
> 当对既有元素做下修饰，让它实现新的功能的时候，可以使用属性指令。

  前面使用元素指令作为myCustomer指令的选择是正确的， 因为你不是在修饰元素，让它具有一些customer属性， 而是定义元素作为customer组件的核心行为。

## 孤立指令作用域(isolating the scope of a directive)
  我们的myCustomer指令很好，但是有致命缺陷。 我们在给定作用域下面只能使用一次。
  当前的实现我们需要，为了能复用这个指令，需要为每个指令创建一个类似的控制器。
  
```
angular.module('docsSimpleDirective', [])
.controller('NaomiController', ['$scope', function($scope){
  $scope.customer = {
    name: 'Naomi',
    address: '1600 Ampitheatre'
  };
}])
.controller('IgorController', ['$scope', function($scope){
  $scope.customer = {
    name: 'Igor',
    address: '123 somewhere'
  };
}])
.directive('myCustomer', function(){
  return {
    templateUrl: 'path/to/my-customer.html'
  };
});

/** index.html **/
<div ng-controller="NaomiController">
  <my-customer></my-customer>
</div>
<div ng-controller="IgorController">
  <my-customer></my-customer>
</div>
```

  明显这不是一个很好的解决方案。
  我们希望有能力在指令内部和外部作用域做区别， 然后将外部作用域映射到指令的内部作用域。 我们可以通过所谓的孤立作用域来实现它。 使用指令的scope选项即可。
  
```
angular.module('docsSimpleDirective', [])
.controller('Controller', ['$scope', function($scope){
  $scope.naomi = {
    name: 'Naomi',
    address: '1600 Ampitheatre'
  };
  $scope.naomi = {
    name: 'Igor',
    address: '123 somewhere'
  };
}])
.directive('myCustomer', function(){
  return {
    restrict: 'E',
    scope: {
      customerInfo: '=info'
    },
    templateUrl: 'path/to/my-customer.html'
  };
});

<div ng-controller="Controller">
  <my-customer info="naomi"></my-customer><hr />
  <my-customer info="igor"></my-customer>
</div>
```

  看看上面的html可知道，第一个<my-customer>使用info='naomi', 第二个使用info='igor', 暴露的是控制器的作用域。 
  再看看scope选项： `scope: { customerInfo: '=info'}`.
  
  scope选项是一个对象， 包含了每个隔离作用域绑定的属性。 上面例子只有一个隔离属性：
  * 名字为customerInfo, 对应于指令的隔离作用域属性customerInfo
  * 它的值为=info, 告诉$compile绑定到info属性。

> ### 注意：
> scope选项中的=attr属性规范就像指令名称一样。 要绑定到<div bind-to-this="thing">的属性，你需要指定绑定值为=bindToThis.

对于要绑定的属性名和你想要绑定到的指令内作用域的名称相同， 可以使用这样的简短语法`scope:{customer: '='}` <=> `scope:{customer: '=customer'}`

除了使得不同数据绑定到指令内部作用域下面， 使用孤立作用域还有另外一个效果。
```
angular.module('docsIsolationExample', [])
.controller('Controller', ['$scope', function($scope) {
  $scope.naomi = { name: 'Naomi', address: '1600 Amphitheatre' };
  $scope.vojta = { name: 'Vojta', address: '3456 Somewhere Else' };
}])
.directive('myCustomer', function() {
  return {
    restrict: 'E',
    scope: {
      customerInfo: '=info'
    },
    templateUrl: 'my-customer-plus-vojta.html'
  };
});

// index.html
<div ng-controller="Controller">
  <my-customer info="naomi"></my-customer>
</div>

// my-customer.html
Name: {{customerInfo.name}} Address: {{customerInfo.address}}
<hr>
Name: {{vojta.name}} Address: {{vojta.address}}
```

  vojta.name, vojta.address都是undefined. 虽然我们在控制器中定义了它们， 但是在指令中是不可使用的。
  
  正如孤立作用域的名字所示， 指令孤立了所有除了scope中指定的之外的所有模型。 这点对于实现可复用组件非常有用的， 因为它阻止模型状态改变，除非模型明确通过scope传入的。
  
> ### 注意:
> 通常来说， 作用于原型继承父类。 孤立作用域不是这样的。 

> ### 最佳实践
> 当你在整个应用中创建可复用组件的时候，使用scope选项创建孤立作用域。


## 创建操作DOM的指令
  下面我们创建的指令例子是显示当前时间。 一次一秒， 更新DOM反应当前时间。
  指令要修改dom一般使用link选项注册DOM监听器以及更新dom. 这是在模版被克隆之后执行， 就是指令逻辑所放的地方执行的。
  
  link接受一个函数，具有下面的签名: function link(scope, element, attrs, controller, transcludeFn) {...}.
  参数及含义如下:
  * scope: angular scope对象
  * element: 指令匹配的jqLite包装的元素对象
  * attrs: 哈希对象，带有key-value对。 规范化的属性名和相应属性值。
  * controller: 指令引入的控制器实例，或者它自己的控制器(如果有的话)。 具体的值依赖指令的require属性。
  * transcludeFn: 嵌入链接函数，预先绑定到确切的嵌入域的。

  在我们的link函数中，我们想要每秒更新显示时间， 或者用户改变我们绑定到指令的时间显示格式字符串。 我们使用$interval服务定期调用handler. 比$timeout使用更加容易些， 在端对端测试也工作良好。 我们需要确保所有的$timeout在完成测试之前完成。 我们也希望在指令被删除的时候，删除$interval, 那样我们不会引起内存泄漏。
  
```
angular.module('docsTimeDirective', [])
.controller('Controller', ['$scope', function($scope) {
  $scope.format = 'M/d/yy h:mm:ss a';
}])
.directive('myCurrentTime', ['$interval', 'dateFilter', function($interval, dateFilter) {
  function link(scope, element, attrs) {
      var format,
          timeoutId;

      function updateTime() {
          element.text(dateFilter(new Date(), format));
      }   

      scope.$watch(attrs.myCurrentTime, function(value){
          format = value;
          updateTime();
      }); 

      element.on('$destroy', function(){
          $interval.cancel(timeoutId);
      }); 

      timeoutId = $interval(function() {
          updateTime();
      }, 1000);
  }   
  return {
      link: link
  };  
}]);


/** index.html **/
<div ng-controller="Controller">
  Date Format: <input ng-model="format" /> <hr />
  Current Time is: <span my-current-time="format"></span>
</div>
```

  这里有一对事情需要注意。 和module.controller api一样， module.directive函数参数也是依赖注入的。 正是因为如此，我们可以在我们指令内部link函数中使用$interval, dateFilter。
  我们为元素注册一个事件: element.on('$destroy', ...), 那么什么会触发$destroy事件呢?
  AngularJS发射少量的特定事件。 当angular编译器编译的节点被销毁的时候， 它会发射一个$destroy事件。 类似的， 当angular 作用域被销毁的时候， 它会广播一个$destroy事件给它的所有监听器。
  
  通过监听这个事件， 你可以删除可能导致内存泄漏的事件监听器。 注册到作用域和元素的监听器当它们销毁的时候会自动清理掉， 但是如果在服务中注册了监听器， 或者在DOM节点上注册的监听器不会被删除掉的， 你需要自行清理， 否则可能会引起内存泄漏。
  
> ### 最佳实践:
> 指令在它们最后需要自行清理。 可以使用element.on('$destroy', ...) 或者scope.$on('$destroy', ...)在指令删除的时候来运行清理函数。

## 创建包含其他元素的指令
  前面的例子我们看到可以使用isolate scope传入模型到指令， 但是有时候，需要有能力传入整个模版而不是字符串或者对象。 让我们这样说， 我想创建一个dialog box组件。 对话框需要能包围任意内容。
  
  要实现这点， 需要使用transclude选项。


## 参考链接
1. [Creating Custom Directives](https://docs.angularjs.org/guide/directive)
