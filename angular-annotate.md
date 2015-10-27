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
