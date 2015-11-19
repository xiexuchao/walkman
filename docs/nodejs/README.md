# NodeJS API
  本文主要是个人学习和使用Node过程的总结，主要关注API层面的相关技术。
  
## Process 进程
  进程process对象是一个全局对象，可以在任何地方访问到它。它是EventEmitter的一个实例。
  
### 进程退出码 Exit codes
  Node执行程序正常情况下会返回0，这点和其他应用程序一致。返回0就意味着包括所有异步在内的操作都已结束。(linux下面可以通过echo $?查看，win cmd下面可以通过echo %ERRORLEVEL%查看)，除了0外，还有如下返回状态码:
  * 1: 未捕获的致命异常(Uncaught Fatal Exception), 发生了未捕获异常，这个异常没有被某个domain或异常处理函数处理。
  * 2: 未使用 由bash保留用于内置误用码
  * 3: 解析错误, Node引导进程导致的javascript源代码解析错误。这个非常少见，这个基本上只发生在node本身的开发过程中。
  * 4: 评估失败 node开发过程中出现的错误码
  * 5: 致命错误 在V8中发生的致命的未发现错误。一般会使用FATAL ERROR前缀在stderr输出一个信息
  * 6: 不正确的异常处理
  * 7: 异常处理函数运行时失败
  * 8: 未使用
  * 9: 无效参数
  * 10: 运行时失败
  * 12: 无效的调试参数
  * >128: 信号退出

### Process相关事件

#### exit事件
  当进程将要退出时触发。这是一个在固定时间检查模块状态(如单元测试)的好时机。需要注意的是exit的回调结束后，主事件循环将不再运行，所以计时器也会实效。
  
  这个事件发生的时候，就没有办法在这个点上阻止主循环的退出了，一旦所有的exit监听器运行完成进程就退出了。因此这里只能执行一些同步操作。 该事件的回调函数接受一个code参数，进程退出的退出码。
  
  这个事件只有在node明确使用process.exit()退出进程或者在主循环耗干的时候隐式触发。
  
  监听exit事件的例子
```
  process.on('exit', function(code){
    // 不要指望这里会帮你做什么
    setTimeout(function(){
      console.log('主事件循环已停止，所以不会执行');
    }, 0);
    
    // 这里是最后机会做点事情了
    console.log('退出前执行');
  });
```
#### message事件
  通过ChildProccess.send()发送的信息可以在子进程中使用message事件获取。
  * message: 解析的JSON或基本类型对象
  * sendHandle: 处理对象，net.Socket或net.Server对象或者undefined.
```
// main.js
var fork = require('child_process').fork;                                              
var example1 = fork(__dirname + '/example1.js');                                       

example1.on('message', function(response) {                                            
    console.log(response);                                                             
});

example1.send({func: 'input'});

// example1.js
function func(input) {
    console.log(input);
    process.send('Hello ' + input);
}

process.on('message', function(m) {
    func(m);
});

// node main.js ==> Hello [object Object]
```

#### rejectionHandled事件
#### uncaughtException事件
#### unhandledRejection事件


# 参考文章
  * [英文node API](https://nodejs.org/api/process.html)
  * [中文node API](http://nodeapi.ucdok.com/#/api/process.html)
