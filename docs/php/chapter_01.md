### 第一章 PHP生命周期

  在通常的Web服务器环境，通常不明确启动PHP解释器；一般启动Apache或者其他一些web server，它们来加载PHP以及当有.php文件被请求的时候处理脚本所需要的进程。
  
------------------------

#### 诸事始于SAPI
  虽然可能看起来完全不一样，CLI二进制实际行为也是类似的方式。php命令，输入系统提示符后，启动命令行sapi, 这个sapi扮演一种小型的web服务器，设计来服务单个请求。当脚本执行完成，这个小型的PHP web服务器关闭并将控制权返回给shell.

==================================

#### 启动及停止
  启动及停止都发生两个独立的启动过程和停止过程。一个周期是PHP解释器作为一个整体执行结构及值的初始化设置，对整个SAPI生命周期都是持久的。另外一个是瞬态设置，仅仅针对单个页面请求。
  
  在启动初始化的过程中，还没有任何请求到来的时候，PHP调用每个扩展的MINIT(模块初始化)方法。 这个时候，扩展希望声明常量、定义类、以及注册资源、流以及过滤处理器， 将来所有请求的脚本都能使用。 这样的特性，设计用来在所有请求都存在，就是所谓的持久化的东西。
  通常MINIT方法看起来类似下面的情况:
```
/**
 * 初始化myextension模块
 * 这个在SAPI启动的时候立即发生
 */
PHP_MINIT_FUNCTION(myextension)
{
  /** Globals **/
#ifdef ZTS
  ts_allocate_id(&myextension_globals_id,
    sizeof(php_myextension_globals),
    (ts_allocate_ctor) myextension_globals_ctor,
    (ts_allocate_dtor) myextension_globals_dotr);
#else
  myextension_globals_ctor(&myextension_globals TSRMLS_CC)
#endif

  /** REGISTER_INI_ENTRIES(), 引用全局结构，在后面INI设置中详细介绍 **/
  REGISTER_INI_ENTRIES();
  
  /** Constants **/
  REGISTER_LONG_CONSTANT("MYEXT_MEANING", 42, CONST_CS | CONST_PERSISTENT); /** define('MYEXT_MEANING', 42);
  
  REGISTER_STRING_CONSTANT("MYEXT_FOO", "bar", CONST_CS | CONST_PERSISTENT); /** define("MYEXT_FOO", "bar");
  
  /** Resources **/
  le_myresource = zend_register_list_destructors_ex(
    php_myext_myresource_dtor, NULL,
    "My Resource Type", module_number);
  le_myresource_persist = zend_register_list_destructors_ex(
    NULL, php_myext_myresource_dtor,
    "My Resource Type", module_number);
    
  /** 流过滤 **/
  if(FAILURE == php_stream_filter_register_factory("myfilter",
    &php_myextension_filter_factory TSRMLS_CC)) {
    return FAILURE;
  }
  
  /** 流包装 **/
  if(FAILURE == php_register_url_stream_wrapper("myproto",
    &php_myextension_stream_wrapper TSRMLS_CC)) {
    return FAILURE;
  }
  
  /** 自动全局 **/
#ifdef ZEND_ENGINE_2
  if(zend_register_auto_global("_MYEXTENSION", sizeof("_MYEXTENSION") - 1,
    NULL TSRMLS_CC) == FAILURE) {
    return FAILURE;
  }
  
  zend_auto_global_disable_jit("_MYEXTENSION", sizeof("_MYEXTENSION") - 1
    TSRMLS_CC);
#else
  if(zend_register_auto_global("_MYEXTENSION", sizeof("_MYEXTENSION") - 1,
    NULL TSRMLS_CC) == FAILURE) {
    return FAILURE;
  }
#endif
  return SUCCESS;
}
```
  MINIT里边做的事情有:
  * 全局变量初始化
  * INI初始化
  * 常量注册
  * 资源注册
  * 流过滤
  * 流包装
  * 自动全局变量

  在请求产生，PHP设置操作环境包括符号表(变量存储的位置)以及同步每个目录的配置值。PHP然后再次循环扩展，这个时候调用每个扩展的RINIT方法。这里，扩展可能重置全局变量为默认值，预先产生的变量到脚本的符号表中，或者执行其他任务比如记录页面请求日志到文件中。RINIT可以被认为是一种对所有请求脚本的auto_prepend_file指令。
  RINIT做的事情:
  * 重置全局变量为默认值
  * 预先产生的变量放到符号表中
  * 记录页面请求日志到文件
  




==================================

#### 生命周期


==================================

#### Zend线程安全

==================================

#### 总结
