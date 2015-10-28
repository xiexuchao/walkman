彻底搞懂injector -- annotate
============================

> 为了让Injector知道要向给定函数注入什么， injector是需要知道函数的参数列表的。 这就是annotate函数的任务。annotate从字面含义解就是注解，注释。

> Angular有三种不同的方式注解你的方法：

1. 使用数组。 数组的最后一个元素为函数， 其他元素为函数的参数名称。 代码`['$scope', '$q', function($scope, $q){}]`
2. 为函数添加一个$inject属性， 这个属性包含参数的列表。`function myFunction($scope, $q){}; myFunction.$inject = ['$scope', '$q']`
3. 什么也没有。 Angular会自动通过函数来解析参数。 当然这种情况在代码压缩后不一定能工作，因为js代码压缩后， 函数的参数名称可能被替换为简短的参数名。 `function myFunction($scope, $q){}`

> 下面看看代码

```
var ARROW_ARG = /^([^\(]+?)=>/;
var FN_ARGS = /^[^\(]*\(\s*([^\)]*)\)/m;
var FN_ARG_SPLIT = /,/;
var FN_ARG = /^\s*(_?)(\S+?)\1\s*$/;
var STRIP_COMMENTS = /((\/\/.*$)|(\/\*[\s\S]*?\*\/))/mg;
var $injectorMinErr = minErr('$injector');

function extractArgs(fn) {
  var fnText = fn.toString().replace(STRIP_COMMENTS, ''),
      args = fnText.match(ARROW_ARG) || fnText.match(FN_ARGS);
  return args;
}

function anonFn(fn) {
  // For anonymous functions, showing at the very least the function signature can help in
  // debugging.
  var args = extractArgs(fn);
  if (args) {
    return 'function(' + (args[1] || '').replace(/[\s\r\n]+/, ' ') + ')';
  }
  return 'fn';
}

function annotate(fn, strictDi, name) {
  var $inject,
      argDecl,
      last;

  if (typeof fn === 'function') {
    if (!($inject = fn.$inject)) {
      $inject = [];
      if (fn.length) {
        if (strictDi) {
          if (!isString(name) || !name) {
            name = fn.name || anonFn(fn);
          }
          throw $injectorMinErr('strictdi',
            '{0} is not using explicit annotation and cannot be invoked in strict mode', name);
        }
        argDecl = extractArgs(fn);
        forEach(argDecl[1].split(FN_ARG_SPLIT), function(arg) {
          arg.replace(FN_ARG, function(all, underscore, name) {
            $inject.push(name);
          });
        });
      }
      fn.$inject = $inject;
    }
  } else if (isArray(fn)) {
      last = fn.length - 1;
    assertArgFn(fn[last], 'fn');
    $inject = fn.slice(0, last);
  } else {
    assertArgFn(fn, 'fn', true);
  }
  return $inject;
}
```

> annotate()可以接收数组参数，也可以接收函数参数。

> 如果传入数组参数， annotate假设落入第一种形式的情况：`['$scope', function($scope){}]`. 这种情况下， 我们断言数组的最后一个元素为函数， 然后删除最后一个元素， 返回数组。

> 如果仅仅传入一个函数， 首先检查函数是否已经通过设置属性$inject注解过。`if (!($inject = fn.$inject)) {`

> 如果没有，我们需要通过函数自身定义来提取参数。 代码的这部分让我们通过正则表达式进行一次有意义的旅行。

> angular利用调用函数toString()的优势事实， 来返回函数定义的完整文本。 angular也支持[arrow functions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions)。

> function.toString()返回脚本中输入的完整文本内容，包括注释。 因此先删除注释， 这里使用正则`/((\/\/.*$)|(\/\*[\s\S]*?\*\/))/mg`来匹配注释。 其中`\/\/.*$)`匹配'//'单行注释。 而`(\/\*[\s\S]*?\*\/)`匹配'/* */'类型的注释， 注意这里要使用修饰符'm', 匹配多行。

```
var fn = function(c/* a parameter*/, a, // More parameters
.... /* Interesting */ r){ };
fn.toString().replace(/((\/\/.*$)|(\/\*[\s\S]*?\*\/))/mg, '');

/* function (c, a,
r){ };*/
```

> 然后使用正则匹配出参数列表。 正则`/^[^\(]*\(\s*([^\)]*)\)/m`匹配参数列表。下面对该正则进行拆解：

1. ｀^`: 字符串开始
2. `\s*`: 匹配任意数量的空格
3. `[^\(]*`: 匹配非(的任意字符， 这将是空格和函数名称， 因此在新版本的angular中不再需要function前置了。
4. `\(`: 匹配(字符。 
5. `[^\)]*`: 非)的任意字符。
6. `)`: 匹配)字符。

> 获取了参数列表后， 然后通过参数分隔符将参数分割成不同部分，通过replace,将每个参数名放入$inject属性中。

```
arg.replace(FN_ARG, function(all, underscore, name){
  $inject.push(name);
})
```

> replace关于第二个参数为function的使用方法，参见[replace](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace#Specifying_a_function_as_a_parameter).

> 下面分析下匹配参数名的正则表达式`/^\s*(_?)(\S+?)\1\s*$/`：

1. `^`: 参数文本开始
2. `\s*`: 任意数量的空格字符
3. `(_?)`: 0个或多个_字符
4. `(\S+?)`: 任意数量的非空格字符。?表示惰性匹配， 尽量少的匹配字符。 因此它会尽量在后面定义的下划线出现前结束匹配。
5. `\1`: 匹配前面的第一组子字符串。 这里就是下划线。这个正则有效确保， 如果参数是以`_`(任意数量)开头的，那么也将以`_`(和前面的字符对称)结束的.
6. `\s*`: 任意数量的空格

> 这个模式非常有趣的一点是， angular模块中可以将你的参数使用任意数量的_包围起来(只要两边的数量相同)并且这些不会包含在注释和后续的服务位置中。

> Phew, that's a lot of regex.

> 那么现在我们有了参数的列表， injector能使用它们来定位注入到我们函数的值了。

> 下一篇 [invoke](https://github.com/walkerqiao/walkman/blob/master/docs/angular/angular-invoke.md)
