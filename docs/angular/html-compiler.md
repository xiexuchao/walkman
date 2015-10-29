HTML Compiler
=====================

## 概述
  Angular的HTML编译器(compiler)允许开发者教浏览器识别新的HTML语法。编译器允许为任何HTML元素或属性附加行为，甚至创建新的HTML元素或带有新行为的元素属性。Angular称这些行为扩展为指令(Directives).

  HTML有大量结构为静态文档以声明的方式格式化HTML。例如，如果某些东西需要居中， 没有必要提供指令给浏览器，找到window的尺寸，除以2找到中间为止， 然后中间还需要使用文本中间位置。而是简单的使用align="center"属性来让任意元素达到希望的行为。 这就是声明语言的强大之处。
  
  然而， 声明语言也有限制， 因为它不能让你教浏览器认识新语法。 例如，没有方便的方法让浏览器让文本处于1/3位置，而非居中。 那么我们希望有一种办法告诉浏览器来认识新的HTML语法。
  
  Angular预先绑定通用指令， 这样有利于构建任何应用。 我们也希望你能创建自定义指令， 对你的应用特定的指令。 这些扩展就成为构建你自己应用的特定领域语言了。
  
  所有的这些编译都是发生在浏览器端；不是在服务器端，也没有预先编译阶段。

## Compiler
  Compiler是一个angular服务， 它遍历DOM查找属性。 编译过程有两个步骤:
  * 编译(compile): 遍历DOM并收集所有指令。 结果是一个链接函数。
  * 链接(link): 组合指令和作用域，生成鲜活的视图。 作用域模型的任何变动反射到视图中， 任何用户和视图的交互都反射到作用域模型中。 这样使得作用域模型是唯一的事实源。

  有些指令比如ng-repeat会为集合中的每个元素克隆DOM元素。有编译和链接两个步骤就可以提升性能， 因为克隆的模版只需要编译一次， 后续只需要为每个克隆实例链接一次即可。
  
## 指令
  指令是在编译过程中当遇到特定HTML结构的时候应该被触发的行为。指令可以放在元素名、属性名、类名、以及注释中。 这里不详细列举了。
  
## 理解视图
  大部分其他模版系统都是消费静态字符串模版并使用数据合并， 返回另外一个新字符串。 结果文本然后使用innerHTML放入元素中。
  
![单向绑定视图](https://github.com/walkerqiao/walkman/blob/master/images/One_Way_Data_Binding.png)
  这意味着任何数据的变化需要重新使用模版合并，然后再用innerHTML赋予给DOM元素。使用这种方法引发的问题有:
  1. 读取用户输入并将其合并到数据中。
  2. 通过覆盖的方式清除用户输入。
  3. 管理整个更新过程。
  4. 缺乏行为表现力。

  Angular就不同了。 angular编译器消费DOM, 而非模版字符串。 结果是链接函数， 链接函数在鲜活视图中合并作用域模型结果。 视图和作用域模型绑定是透明的。 开发人员不需要做任何特殊调用来更新视图。 并且因为没有使用innerHTML, 因此不用意外担心用户输入。此外，angular指令可以包含不仅仅文本绑定， 也支持行为构造。

![双向绑定视图](https://github.com/walkerqiao/walkman/blob/master/images/Two_Way_Data_Binding.png)

  Angular的方式产生一个稳健的DOM。 DOM元素实例绑定到不改变绑定生命周期的模型项目实例 。这就意味着代码能维持元素和注册事件处理器， 并知道引用不会被模版数据合并所销毁。
  
## 指令如何被编译的呢?
  需要注意的非常重要的一点是angular操作的是DOM节点，而非字符串。 通常来说，你没有注意这个限制，因为当页面加载后， web浏览器自动解析HTML到DOM。
  
  HTML编译发生有以下三个阶段:
  1. $compile遍历DOM，并匹配指令集。 如果编译器发现元素匹配指令， 这个指令被添加到匹配DOM元素的指令列表。 单个元素可能匹配多个指令。
  2. 一旦所有指令匹配DOM元素被确定， 编译器根据优先级排序指令。 每个指令的编译函数被执行。 每个编译函数都有机会修改DOM元素。 每个编译函数都返回一个链接函数。 这些函数被组合到一个合并的链接函数里边， 在那里调用每个指令返回的链接函数。
  3. $compile使用作用域链接模版， 通过前面一步返回的调用合并的链接函数。这样实际上就是调用每个单独指令的链接函数， 在元素上面注册监听器， 以及如果每个指令配置$watch的话，使用scope设置$watch.

  结果是实时绑定作用域和DOM的。 因此在这个视点上看， 编译的作用域上的模型改变会反应到DOM上。
  
  下面是使用$compile服务的具体代码。 这个可能会给你个概念angular内部到底做了些什么。
```
var $compile = ...; // injected into your code;
var scope = ...;
var parent = ...; // DOM element where the compiled template can be appended

var html = '<div ng-bind="exp"></div>';

// step 1: parse html into dom element
var template = angular.element(html);

// step 2: compile the template
var linkFn = $compile(scope);

// step 3: link the compiled template with the scope
var element = linkFn(scope);

// step 4: append to dom (optional)
parent.appendChild(element);
```

## 编译和链接间的区别
  此时你可能疑惑，编译过程为什么被分为编译和链接过程。 简短的回答是编译和链接分离在模型改变导致dom结构变化是需要时间的。
  
  具有编译函数的指令是非常罕见的， 因为大部分指令都关心特定DOM元素实例， 而不是改变整个结构。
  
  指令通常都有link函数。 link函数允许指令注册监听器到特定克隆的dom元素实例上，以及从作用域拷贝内容到dom中。
  
> ### 最佳实践:
> 由于性能的原因，不同指令实例间可共享的操作都应该移动到compile函数中去。
  
## An Example of Compile Versus Link
  为了理解方便， 我们看一个使用ngRepeat的现实的例子：
```
Hello {{user.name}}, you have these actions:
<ul>
  <li ng-repeat="action in user.actions">
    {{action.description}}
  </li>
</ul>
```
  上面的例子在编译的时候， 编译器访问每个节点并查找指令集。{{user.name}}匹配插值指令(interpolation directive), ng-repeat匹配ngRepeat指令.
  
  但是ngRepeat有一个困境(dilemma). 它需要有能力为user.actions的每个action克隆<li>元素. 乍一看貌似平凡无奇， 但是当你考虑到user.actions后续会有元素添加进来的时候就会变得复杂了。这就意味着需要保存一个干净版本到<li>元素用在克隆上面。
  随着新的action被插入， 模版<li>元素需要被克隆并插入到<ul>中。 但是克隆<li>元素还不够， 而且还需要编译<li>，那么它到指令，比如{{action.description}}, 根据正确的作用域计算。
  解决这个问题的天真方法是简单插入<li>元素，然后编译它。 这个方法的问题是需要为每个克隆的<li>编译一次，这样就有大量重复工作。具体的说， 我们将每次在克隆前都要遍历<li>查找指令。 这样将导致编译处理很慢， 这样使得应用在插入新节点的时候就会响应很慢。
  
  解决的方案是将编译处理分为两个过程：
  1. 编译阶段标识所有的指令并按优先级排序
  2. 链接阶段执行链接特定的作用域实例和特定的<li>实例。

> ### 注意
> Link意味着在DOM上设置监听器，以及在scope上设置$watch来保持两者同步。

  ngRepeat防止编译过程降级到<li>元素，以便能从原始的<li>克隆，以及自己处理插入和删除dom。
  
  ngRepeat不单独编译<li>。 <li>元素编译的结果是一个链接函数， 包含了所有指令包含<li>元素， 准备附加到特定克隆的<li>元素上。
  
  在运行时， ngRepeat监视表达式和添加的项目到克隆<li>的数组，为克隆的<li>元素创建一个新的scope， 并在克隆对象<li>上调用链接函数。
  
  未完待续....

## 参考链接:
* [HTML Compiler](https://docs.angularjs.org/guide/compiler)
