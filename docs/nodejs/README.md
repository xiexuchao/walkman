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
  监听exit事件的例子
```
  process.on('exit', function(){
  
  });
```
#### message事件
#### rejectionHandled事件
#### uncaughtException事件
#### unhandledRejection事件


# 参考文章
  * [英文node API](https://nodejs.org/api/process.html)
  * [中文node API](http://nodeapi.ucdok.com/#/api/process.html)
