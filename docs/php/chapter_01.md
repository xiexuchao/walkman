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
  
  RINIT方法可能期望像下面的代码:
```
/**
 * 每个页面请求的开始时候运行
 */
PHP_RINIT_FUNCTION(myextension)
{
  zval *myext_autoglobal;
  
  /**
   * 初始化在MINIT中声明的自动全局变量为数组。
   * 等价于执行$_MYEXTENSION = array();
   */
  ALLOC_INIT_ZVAL(myext_autoglobal);
  array_init(myext_autoglobal);
  zend_hash_add(&EG(symbol_table), "_MYEXTENSION", sizeof("_MYEXTENSION") - 1,
    (void**)&myext_autoglobal, sizeof(zval*), NULL);
  return SUCCESS;
}
```
  在请求处理完成，或者到达脚本文件的最后一行，或者通过die(), exit()退出，PHP通过调用每个扩展的RSHUTDOWN(Request Shutdown)方法启动清理进程。RSHUTDOWN相应于auto_append_file，就像RINIT相应于auto_prepend_file指令一样。RSHUTDOWN和auto_append_file指令最大的不同在于，RSHUTDOWN总是被执行，而在用户空间调用die()或exit()，会跳过auto_append_file指令的执行。
  
  最后一分钟需要在符号表和其他资源销毁之前执行的可以放在RSHUTDOWN中。在所有RSHUTDOWN方法执行完成，符号表中的每个变量都被明确的unset(), 在这之间所有非持久资源和对象析构被调用为了优雅的释放资源。
  
```
/**
 * 在每个页面请求结束的时候运行
 */
PHP_RSHUTDOWN_FUNCTION(myextension)
{
  zval **myext_autoglobal;
  
  if(zend_hash_find(&EG(symbol_table), "_MYEXTENSION", sizeof("_MYEXTENSION"), 
    (void **)&myext_autoglobal) == SUCCESS) {
    /**
     * 使用$_MYEXTENSION数组做一些有意义的事情
     */
    php_myextension_handle_values(myext_autoglobal TSRMLS_CC);
  }
  return SUCCESS;
}
```

  最后，当所有请求都完成，web服务器或者其他SAPI准备关闭，PHP循环每个扩展的MSHUTDOWN方法。这是扩展在MINIT周期最后一次机会注销处理器以及释放持久内存非配。
```
/*
 * 该模块准备卸载，常量及函数将会被自动清楚，持久资源，类实例，以及流处理器必须手工注销掉。
 */
PHP_MSHUTDOWN_FUNCTION(myextension)
{
  UNREGISTER_INI_ENTRIES();
  php_unregister_url_stream_wrapper("myproto" TSRMLS_CC);
  php_stream_filter_unregister_factory("myfilter" TSRMLS_CC);
  return SUCCESS;
}
```

==================================

#### 生命周期
  每个PHP实例，无论是从init脚本开始，还是从命令行开始，都遵循一些列前面介绍的Request/Module, Init/Shutdown的事件串，以及脚本自身的实际执行。 每个启动和关闭阶段执行的次数、频率都依赖于使用的SAPI. 四个最常见的SAPI配置是CLI/CGI, 多进程模块、多线程模块以及嵌入式。
  
##### CLI生命周期
  CLI(和CGI)在其单个请求生命周期相对独特的；然而，模块与请求步骤依然遵循离散循环。下图展示了在命令行调用脚本test.php的进展情况。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/php/cli_php_life_cycle.png)

##### 多进程生命周期
  最常见的PHP嵌入web服务器的配置是使用PHP作为apache 1的APXS模块或apache 2中使用Pre-fork MPM。 很多其他web服务器配置也是这种情况，就是本书中所谓的多进程模型。
  
  称之为多进程模型是因为apache启动的时候，立即fork几个子进程，每个子进程都有自己的进程空间，并且功能互相独立。 在给定的子进程中， PHP实例的生命周期看起来就像下图所示。唯一变化的就是多个请求位于单个MINIT/MSHUTDOWN对之间。
  
  ![](https://github.com/walkerqiao/walkman/blob/master/images/php/single_apache_php_life_cycle.png)
  
  这个模型不允许任何一个子进程意识到其他子进程所拥有的数据， 虽然它确实允许子进程die以及按需替换而不影响其他子进程的稳定。下图显示了单个apache调用以及调用它们每个自己的MINIT, MSHUTDOWN，RINIT, RSHUTDOWN的多个子进程。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/php/mpm_php_life_cycle.png)
  
##### 多线程生命周期
  越来越多的，PHP在多线程web服务器配置中出现，例如IIS的ISAPI接口，以及apache2的Worker MPM. 在多线程web服务器中，同一时间只有一个进程在运行。但是在那个进程中有多线程同时执行。这允许多位开销，包括重复调用MINIT/MSHUTDOWN得以避免，真正的全局数据被分配以及仅仅初始化一次，以及潜在打开的多个请求确定性共享信息的大门。下图展示了当运行在多线程web服务器，例如apache2中使用PHP发生的并行处理流。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/php/multi_thread_php_life_cycle.png)
  
##### 嵌入式生命周期
  回忆一下，嵌入SAPI就是另外一个SAPI实现，和CLI, APXS或ISAPI接口遵循相同的规则， 很容易想象，请求的生命周期遵循相同的基本路径: 模块初始化=>请求初始化=>请求=>请求关闭=>模块关闭。 确实，嵌入SAPI完全遵循它兄弟姐妹同样的这些步骤。
  
  让嵌入式SAPI单独出现的原因是请求可以被分为多个脚本片段，这些功能作为单个请求的一部分。控制也将在PHP和调用应用程序之间基于最多的配置来回传递多次。
  
  虽然嵌入请求可能由单个或多个元素组成，嵌入应用程序都是面向同一个请求独立的必要条件作为web服务器。为了处理两个或多个同时嵌入的环境， 应用程序将需要像apache1或apache2的线程一样。尝试处理两个单独请求环境在单个非线程进程空间会导致意想不到的结果，并且当然不需要的。
  
==================================

#### Zend线程安全
  当PHP刚起步的时候，它作为单个CGI进程运行，那么不关心线程安全，因为没有进程空间可以比单个请求更加长寿。内部变量可以被生命在全局空间并且可以任意访问和改变而不会有任何后果，只要它们的内容被正确的初始化了。任何没有清理正确的资源也将在进程终止的时候被释放。

  后来，PHP嵌入到多进程web服务器，比如apache。给定的内部变量仍然可以定义在全局并且可以被活动请求安全访问，只要它在每个请求开始的时候被正确的初始化，以及在结束的时候清理掉，因为每个进程空间一次只能有一个请求为激活状态。这个点上来说，每个请求的内存管理被添加防止资源泄漏而失去控制。
  
  随着单进程多线程web服务器开始出现，那么新的处理全局数据的方法就必要了。最终，这个将出现一个新的层叫做TSRM(线程安全资源管理)。
  
##### 线程安全与非线程安全定义
  在简单的非线程安全应用程序中，你可能喜欢通过将变量放在源代码文件的顶部来声明全局变量。 编译器将为你的应用程序在数据段分配一个内存块来保存那个信息单元。
  
  在多线程应用中，么么线程都需要它自己的数据元素版本，必要分配一个单独的内存块给每个线程。 给定线程然后在需要数据的时候从正确的内存块取用数据，从那个指针来引用数据。
  
##### 线程安全数据池
  在扩展的MINIT阶段，TSRM层使用一次或多次ts_allocate_id()函数调用来被通知有多少数据将需要被存储。TSRM添加那个字节计数到运行中的必要数据空间总数，然后返回线程数据池的那个段部分的一个新的，唯一的标识符。
```
typedef struct {
  int sampleint;
  char *samplestring;
} php_sample_globals;

int smaple_globals_id;

PHP_MINIT_FUNCTION(sample)
{
  ts_allocate_id(&sample_globals_id,
    sizeof(php_sample_globals),
    (ts_allocate_ctor) php_sample_globals_ctor,
    (ts_allocate_dtor) php_sample_globals_dtor);
    
  return SUCCESS;
}
```
  当是时候在请求中访问数据段，扩展为当前线程资源池从TSRM层请求一个指针，通过ts_allocate_id()返回的资源ID暗示获取恰当的索引建议。
  
  用另外一种方式，以代码流的术语，下面的语句SAMPLE_G(sampleint) = 5;就是你可以看到的，在之前的MINIT语句中联系的模块，在线程安全版本下面， 这个语句通过一系列宏扩展为下面的情况:
  `(((php_sample_globals*)(*((void ***)tsrm_ls))[sample_global_id - 1])->sampleint = 5`
  
  不用担心如果你对解析上面的语句有问题。它和PHPSAPI集成的相当好，因此有些开发者根本不需要了解它如何工作。
##### 什么情况下不要用线程


==================================

#### 总结
