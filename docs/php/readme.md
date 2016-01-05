### PHP扩展开发
  PHP为了扩展方便扩展开发，提供了一个ext_skel工具，可以自动生成扩展代码骨架。
  
  首先我们假设要创建一个venus扩展，其中提供一个函数venus_skel函数， 这个原型文件为venus.skel, 代码如下:
```
string venus_string(string str)
```
  那么我们可以使用ext_skel工具来生成这个扩展的骨架。`./ext_skel --extname=venus --proto=venus.skel`. 生成模块venus, 其目录结构如下:
```
$ ls -la venus

-rw-r--r--.  1 xxx users 2044 1月   4 14:23 config.m4
-rw-r--r--.  1 xxx users  348 1月   4 14:22 config.w32
-rw-r--r--.  1 xxx users    5 1月   4 14:22 CREDITS
-rw-r--r--.  1 xxx users    0 1月   4 14:22 EXPERIMENTAL
-rw-r--r--.  1 xxx users  398 1月   4 14:22 .gitignore
-rw-r--r--.  1 xxx users 2301 1月   4 14:22 php_venus.h
drwxr-xr-x.  2 xxx users 4096 1月   4 14:22 tests
-rw-r--r--.  1 xxx users 5673 1月   4 14:22 venus.c
-rw-r--r--.  1 xxx users  499 1月   4 14:22 venus.php
```

  然后修改config.m4: 第10~12行, 去掉前面的注释，代码如下:
```
 PHP_ARG_WITH(venus, for venus support,
 Make sure that the comment is aligned:
 [  --with-venus             Include venus support])
```
  修改venus.c文件中的venus_string函数:
```
PHP_FUNCTION(venus_string)
{
    char *str = NULL;
    int argc = ZEND_NUM_ARGS();
    int str_len;
    char *result;

    if (zend_parse_parameters(argc TSRMLS_CC, "s", &str, &str_len) == FAILURE)
        return;

    str_len = spprintf(&result, 0, "<a href=\"%.78s\">Link</a>", str);
    RETURN_STRINGL(result, str_len);
}
```
  最后编译安装扩展phpize, ./configure, make, make install. 那么扩展就被安装到/pathtophp/lib/php/extensions/no-debug-non-zts-20151012/ 下面， 然后修改php.ini, 添加extension=venus.so。
  重启php-fpm或httpd, 这个要看你的php安装模式了，不详述。
  
  最后测试扩展。 将venus.php拷贝到web根目录，测试ok.
  
### PHP扩展开发总结
  PHP扩展开发非常简单，总结如下:
  1. 使用ext_skel工具产生模块骨架目录结构
  2. 修改config.m4文件， 该文件是后面编译的重点，后面介绍。
  3. 修改模块源文件
  4. 编译安装模块
  5. 测试模块
  

### config.m4简介
  * dnl: 以该符号开头的行为注释行
  * PHP_ARG_WITH或者PHP_ARG_ENABLE: 指定了PHP扩展模块的工作方式
  * PHP_REQUIRE_CXX: 用于指定这个扩展用到了C++
  * PHP_ADD_INCLUDE: 指定PHP扩展模块用到的头文件目录
  * PHP_CHECK_LIBRARY: 指定PHP扩展模块PHP_ADD_LIBRARY_WITH_PATH定义以及库连接错误信息等
  * PHP_SUBST: 用于说明这个扩展编译成动态链接库的形式
  * PHP_NEW_EXTENSION: 用于指定有哪些源文件应该被编译,文件和文件之间用空格隔开

  PHP_ARG_**: 这些给configure提供了可选项，在运行./configure --help时显示的帮助文本。就像名称暗示的，其两者不同之处在于--with-xxx还是--enable-xxx。每个扩展应提供至少一个以上的选项以及扩展名称，以便用户可选择是否将扩展构建至PHP中。按惯例，PHP_ARG_WITH()用于取得参数的选项，例如扩展所需库或程序的位置；而PHP_ARG_ENABLE()用于代表简单标志的选项。
```
$ ./configure --help
...
  --with-example[=FILE]       Include example support. FILE is the optional path to example-config
  --enable-example-debug        example: Enable debugging support in example
  --with-example-extra=DIR      example: Location of extra libraries for example
...

$ ./configure --with-example=/some/library/path/example-config --disable-example-debug --with-example-extra=/another/library/path
...
checking for example support... yes
checking whether to enable debugging support in example... no
checking for extra libraries for example... /another/library/path
...
```
>在调用 configure 时，不管选项在命令行中的顺序如何，都会按在 config.m4 中指定的顺序进行检测。

  参考连接：http://www.php.net/manual/zh/internals2.buildsys.configunix.php
  
### 扩展的结构
  不管是通过手工，还是通过ext_skel，还是通过另外的扩展生成器，例如CodeGen, 所有的扩展至少有四个文件:
  * config.m4: Unix构建系统配置
  * config.w32: Windows构建系统配置
  * php_xxx.h: 当将扩展作为静态模块构建并放入php二进制包时，构建系统要求用php_加扩展名称命名的头文件包含一个对扩展模块结构的指针定义。就像其他头文件，此文件经常包含附加的宏、原型和全局量。
  * xxx.c: 主要的扩展源文件。按惯例，此文件名就是扩展的名称，但不是必须的。此文件包含模块结构定义、INI条目、管理函数、用户空间函数和其他扩展所需的内容。
  
  构建系统的文件在别处讨论，本节关注于其余的部分。扩展应包含任意数量的头文件、源文件、单元测试和其他支持文件， 此四个文件仅够组成最小的扩展。 下面列举counter扩展的文件列表:
```
ext/
 counter/
  .cvsignore
  config.m4
  config.w32
  counter_util.h
  counter_util.c
  php_counter.h
  counter.c
  package.xml
  CREDITS
  tests/
   critical_function_001.phpt
   critical_function_002.phpt
   optional_function_001.phpt
   optional_function_002.phpt
```
  非源文件
  .cvsignore文件用于已检入一个php cvs资源库(通常是PECL)的扩展，由ext_skel生成这个文件内容如下:
```
.deps
*.lo
*.la
```
  这几行告诉CVS要忽略由PHP构建系统生成的临时文件。这只是惯例，可以被完全省略掉且无不良影响。
  
  CREDITS文件用纯文本格式列出了扩展的贡献者和(或)维护者。此文件的主要用途是为了为已捆绑的扩展生成被phpcredits()所使用的荣誉信息。按惯例，文件的第一行应保存扩展的名称，第二行使用逗号分割的贡献者名单。贡献者通常是按其贡献物的时间顺序。例如，在PECL包中，此信息已经维护在package.xml中了。 这就是可以被省略且无不良影响的其他文件。
  
  package.xml文件是基于PECL的扩展特有的，是一个元信息文件，记录了扩展的依赖关系、作者、安装需求和其他有价值的信息。对于不是基于PECL的扩展来说，此文件就无关紧要了。
  
  
### 基本组成
  C是现代定义的非常低级的语言。这意味着没有很多内置的支持可让PHP过渡的(granted), 例如反射机制，动态模块加载，边界检查，线程安全数据管理以及各种有用的数据结构包括链表和hash表。 同时，C也是语言支持和功能的共同标准。 给予足够努力，这些概念没有什么不可能实现的； Zend引擎就使用了所有这些概念。
  
  大量努力已经付出，使得Zend api既具扩展型又易于理解，但是C强制特定必要声明于各种扩展之上， 对于不太有经验的视角来看似乎有些冗余和不必要。 所有本节中这些组成，通常都写过一次，而在zend 2, 3中忘记掉了。 这里有一些例外， 由php5.3的ext_skel预先生成的php_counter.h和counter.c文件， 显示了预先生成声明:
> 注意， 精明的读者可能会注意到在文件中有几个声明展示。 这些声明特定于各种Zend子系统，以及必要的时候会讨论的。

```
extern zend_module_entry counter_module_entry;
#define phpext_counter_ptr &counter_module_entry;

#ifdef PHP_WIN32
#define PHP_COUNTER_API __declspec(dllexport)
#elif defined(__GNUC__) && __GNUC__ >= 4
#define PHP_COUNTER_API __attribute__ ((visibility("default")))
#else
#define PHP_COUNTER_API
#endif

#ifdef ZTS
#include "TSRM.h"
#enif

#include "php.h"
#include "php_ini.h"
#include "ext/standard/info.h"
#include "php_counter.h"

/** ... **/
#ifdef COMPILE_DL_COUNTER
ZEND_GET_MODULE(counter)
#endif
```
  * counter_module_entry相关的行声明了全局变量，并且一个宏指针指向它， 这个包含了扩展的zend_module_entry。 尽管后面讨论真正全局的弊端，这种使用依然是愿意的，Zend使用这种预防措施是为了避免变量的误用。
  * PHP_COUNTER_API声明用于模块想要暴露给其他模块使用的非PHP函数。counter扩展没有声明这些，在最后版本的header文件中，这个宏会被删除掉。PHPAIP宏声明一样可以被标准扩展使用使得phpinfo()工具函数对其他模块可用。
  * TSRM.h头文件的包含，如果php或者扩展不是以线程安全模式编译的话，被跳过，这里使用ZTS来检测。
  * 标准的头文件包含列表，特别是扩展自己的php_counter.h文件被给定。 config.h给定了扩展访问来确定configure所产生。 php.h是进入PHP和ZEND API的网关。php_ini.h添加API运行时配置入口。 并不是所有扩展都需要它。 最后,ext/standard/info.h导入前面说的phpinfo()工具API的东西。
  * COMPILE_DL_COUNTER仅由configure来定义，如果counter扩展既是启用的，而且是作为动态加载模块而非直接连接到PHP的。 ZEND_GET_MODULE定义了小函数，Zend可以用来在运行时获取扩展的zend_module_entry。

> 注意：细心的读者可能深入到main/php_config.h， 在尝试构建counter模块静态启用可能注意到那里会有一个HAV_COUNTER常量定义，有些源代码不检查。 这是简单理由，这个检查没有做:它没有必要。 如果扩展没有启用，源代码文件根本不会被编译。

### zend_module结构
  PHP扩展的主文件包含几个C程序员新的结构。最重要的三个是: 首先接触的是在启动新扩展的时候，就是zend_module结构。 这个结构包含一个告诉Zend引擎关于扩展依赖、版本、回调、以及其他重要数据的宝藏。 随着时间的推移，这个结构发生了突变；这一节我们关注这个结构，基于php5.2之后， 并且在php5.3有非常小的变动。
  
  counter.c中的zend_module声明看起来类似下面的代码。 这个例子是由ext_skel工具生成的， 删除了一些过时的结构。
```
zend_module_entry counter_module_entry {
  STANDARD_MODULE_HEADER,
  "counter",
  counter_functions,
  PHP_MINIT(counter),
  PHP_MSHUTDOWN(counter),
  PHP_RINIT(counter),
  PHP_RSHUTDOWN(counter),
  PHP_MINFO(counter),
  "0.1", /** replace with version number for your extensions **/
  STANDARD_MODULE_PROPERTIES
};
```
  
  这卡一看令人却步， 但是该结构的很多成员都非常容易理解。 这里是zend_module的声明(php5.3 zend_modules.h)
```
struct _zend_module_entry{
  unsigned short size;
  unsigned int zend_api;
  unsigned char zend_debug;
  unsigned char zts;
  const struct _zend_ini_entry *ini_entry;
  const struct _zend_module_dep *deps;
  const char *name;
  const struct _zend_function_entry *functions;
  int (*module_startup_func)(INIT_FUNC_ARGS);
  int (*module_shutdown_func)(SHUTDOWN_FUNC_ARGS);
  int (*request_startup_func)(INIT_FUNC_ARGS);
  int (*request_shutdown_func)(SHUTDOWN_FUNC_ARGS);
  void (*info_func)(ZEND_MODULE_INFO_FUNC_ARGS);
  const char *version;
  size_t globals_size;
#ifdef ZTS
  ts_rsrc_id* globals_id_ptr;
#else
  void* globals_ptr;
#endif
  void (*globals_ctor)(void *global TSRMLS_DS);
  void (*globals_dtor)(void *global TSRMLS_DS);
  int module_started;
  unsigned char type;
  void *handle;
  int module_number;
};
```
  很多对于扩展开发者来说，根本不会碰触到。有一些标准宏会自动为它们设置恰当的值。宏STANDARD_MODULE_HEADER文件包含了直到deps字段的所有。相应的, STANDARD_MODULE_HEADER_EX会将deps留空，以便开发者自己使用。 开发者通常负责name到version之间的字段。在version之后，STANDARD_MODULE_PROPERTIES宏会填充剩下的结构，或者STANDARD_MODULE_PROPERTIES_EX宏可用于让globals和post-deactivation功能字段不填充。很多现代扩展会使用模块全局变量。
  
> 注意： 下面表给出了每个字段可以允许开发者填充的值， 每个字段一个，没有使用所写宏的方式。 这不是推荐的方式。正确的方式是只填充那些可能变化的字段。 尽可能的使用宏。

```
字段                  值                                描述
-----------------------------------------------------------------------------------
size                  sizeof(zend_module_entry)         结构的字节长度
zend_api              ZEND_MODULE_API_NO                该模块被编译的Zend API版本
zend_debug            ZEND_DEBUG                        标志，表明模块是否需要使用调试开始的方式编译
zts                   USING_ZTS                         标识表明是否该模块需要使用线程安全开启编译
ini_entry             NULL                              这个指针用于zend内部来保持非局部引用到该模块声明的任意INI实体
deps                  NULL                              指向模块依赖列表的指针
name                  "mymodule"                        模块的名称。这个是简短名称，例如"spl"或"standard".
functions             mymodule_functions                指向模块函数表的指针， zend用于向用户空间暴露功能。
module_startup_func   PHP_MINIT(mymodule)               回调函数，Zend会在模块载入php特定实例时首先调用。
module_shutdown_func  PHP_MSHUTDOWN(mymodule)           回调函数，Zend会在特定PHP实例卸载模块的时候调用
request_startup_func  PHP_RINIT(mymodule)               回调函数，Zend会在每隔请求开始的时候调用。它应尽可能的短或者NULL， 因为每个请求都会有消耗
request_shutdown_func PHP_RSHUTDOWN(mymodule)           回调函数, Zend会在每个请求结束的时候调用. 它也应该尽可能的简短,或者为NULL, 因为每个请求调用它还是有代价的。
info_func             PHP_MINFO(mymodule)               回调函数, 当phpinfo()函数被调用的时候，Zend会调用它
version               NO_VERSION_YET                    模块版本号的字符串， 由模块开发者指定。 推荐版本数字符合version_compare()期望的。(例如: "1.0.5-dev"), 或CVS或SVN版本号(比如: $Rev: 322138 $)
global_size           sizeof(zend_mymodule_globals)     包含模块全局变量的数据结构的尺寸，如果有全局变量的话
globals_id_ptr        &mymodule_globals_id              |只有这两个字段中的一个存在， 依赖于是否使用USING_ZTS常量为TRUE. 
globals_ptr           &mymodule_globals                 |前一个是模块全局变量在TSRM的分配表中的索引,后面的是直接指向全局变量的指针。
globals_ctor          PHP_GINIT(mymodule)               这个函数在任意module_startup_func之前初始化模块的全局变量
globals_dtor          PHP_GSHUTDOWN(mymodule)           这个函数在任意module_shutdown_func之后取消模块全局变量的分配
post_deactivate_func  ZEND_MODULE_POST_ZEND_DEACTIVATE_N(mymodule) 这个函数在请求shutdown之后被调用。 它很少使用
module_started        0                                 
type                  0                                 这些字段用于Zend内部追踪信息
handle                NULL
module_number         0 
```

  特定情况下的结构填充
  使用所有的这些字段，可能比较困惑用于什么目的。 下面是zend_module对counter的定义例子， 更新的最后形式:
```
zend_module_entry counter_module_entry = {
  STANDARD_MODULE_HEADER,
  "counter",
  counter_functions,
  PHP_MINIT(counter),
  PHP_MSHUTDOWN(counter),
  PHP_RINIT(counter),
  PHP_RSHUTDOWN(counter),
  PHP_MINFO(counter),
  NO_VERSION_YET,
  PHP_MODULE_GLOBALS(counter),
  PHP_GINIT(counter),
  PHP_GSHUTDOWN(counter),
  NULL,
  STANDARD_MODULE_PROPERTIES_EX
};
```
  * STANDARD_MODULE_HEADER在模块没有定义任何依赖关系的时候使用
  * counter是扩展的名称， 用于定义各种传给Zend的回调函数。 "counter"在启动和关闭时候使用模块， 全局变量，以及请求函数， 以及为phpinfo()提供信息，因此所有的7个回调函数被定义了。
  * 假设有一种类型为zend_function_entry *的变量命名为counter_functions. 包含在包含模块定义的文件开头位置，包含了模块暴露给用户空间的函数列表
  * NO_VERSION_YET是一种特别方便的方式告诉Zend该模块尚未有版本号。 实际模块正确的方式可能是放1.0到这个地方。
  * counter使用单个模块的全局变量，因此使用PHP_MODULE_GLOBALS.
  * 这个模块没有post-deactivate函数，所以使用NULL
  * 既然这个模块使用全局变量，STANDARD_MODULE_PROPERTIES_EX被使用来填充结构
  
### 扩展全局变量
  PHP扩展全局变量介绍
  在诸如C语言中，全局变量是可以在任意函数中访问，而无需额外声明的。 这些传统全局变量有一些缺点:
  * 除非有特殊的选项传给编译器，否则全局变量可以在程序中的任意片段访问和改变，而不管那些代码是否应该如此。
  * 经典传统变量不是线程安全的。
  * 全局变量名称尽可能作为变量自身是全局的
  
> 例1 存储基本的counter接口值的错误方式
```
static long basic_counter_value;
PHP_FUNCTION(counter_get)
{
  RETURN_LONG(basic_counter_value);
}
```
  表面上看这是一个可行的解决方案，并且确实测试其功能也正确。 然而，如果一些情况，php在相同的线程中有多个拷贝，那就意味者有多个counter模块。 这些多线程突然共享同一个counter值， 很明显不是所希望的。 另外一个问题就是当考虑另一个扩展可能有一天碰巧具有相同名字的全局变量，因为C的作用域规则，这就会导致潜在的编译错误，或者更加坏的情况， 运行时错误。 需要更加复杂的解决方案，因此存在Zend的线程安全单个模块的全局变量支持。
  
#### 声明模块全局变量
  不管模块是使用一个全局变量还是一打全局变量，他们都必须定义到一个结构中， 并且那个结构需要声明。有一些宏帮助做这些， 因此可以避免模块间名字冲突: ZEND_BEGIN_MODULE_GLOBALS(), ZEND_END_MODULE_GLOBALS()和ZEND_DECLARE_MODULE_GLOBALS(). 这三个宏都带一个模块名的简短名称，这在counter模块中仅仅是简单的counter. 这里是在php_counter.h中声明的全局结构:
```
ZEND_BEGIN_MODULE_GLOBALS(counter)
  long baisc_counter_value;
ZEND_END_MODULE_GLOBALS(counter)

>>>
  typedef struct _zend_##module_name##_globals { /** begin **/
    long basic_counter_value;
  } zend_##module_name##_globals;       /** end **/
```
  下面是在counter模块中的全局结构声明:
```
ZEND_DECLARE_MODULE_GLOBALS(counter)

>>>
#ifdef ZTS

#define ZEND_DECLARE_MODULE_GLOBALS(module_name)                     \                                                                                                         
   ts_rsrc_id module_name##_globals_id;
#define ZEND_EXTERN_MODULE_GLOBALS(module_name)                        \
   extern ts_rsrc_id module_name##_globals_id;
#define ZEND_INIT_MODULE_GLOBALS(module_name, globals_ctor, globals_dtor)   \
   ts_allocate_id(&module_name##_globals_id, sizeof(zend_##module_name##_globals), (ts_allocate_ctor) globals_ctor, (ts_allocate_dtor) globals_dtor);

#else

#define ZEND_DECLARE_MODULE_GLOBALS(module_name)                     \
   zend_##module_name##_globals module_name##_globals;
#define ZEND_EXTERN_MODULE_GLOBALS(module_name)                        \
   extern zend_##module_name##_globals module_name##_globals;
#define ZEND_INIT_MODULE_GLOBALS(module_name, globals_ctor, globals_dtor)   \
   globals_ctor(&module_name##_globals);

#endif
```

#### 访问模块全局变量
  正如上面所讨论的，每个模块的全局变量都是定义在C结构体中，这个结构体的名字使用Zend宏遮蔽起来。 那么， 实际访问结构的成员也是用宏来实现。 当然了，大多数而非全部扩展都在头文件中使用上述声明方式。
  
```
#ifdef ZTS
#define COUNTER_G(v) TSRMG(counter_globals_id, zend_counter_globals *, v)
#else
#define COUNTER_G(v) (counter_globals.v)
#endif
```

  下面是新版本的counter_get()代码：
```
/** php_counter.h **/
ZEND_BEGIN_MODULE_GLOBALS(counter)
  long basic_counter_value;
ZEND_END_MODULE_GLOBALS(counter)

#ifdef ZTS
#define COUNTER_G(v) TSRMG(counter_globals_id, zend_counter_globals *, v)
#else
#define COUNTER_G(v) (counter_globals.v)
#endif

/** counter.c **/
ZEND_DECLARE_MODULE_GLOBALS(counter)

/** ... **/
PHP_FUNCTION(counter_get)
{
  RETURN_LONG(COUNTER_G(basic_counter_value));
}
```

  这是一个正确的实现。
  
### 扩展的生命周期
  PHP扩展在其生命周期中经历数个步骤。 所有这些步骤都给开发者执行各种初始化、终止或者信息功能的机会。Zend API允许对扩展存在的五个单独步骤进行挂钩， 而不用调用PHP函数。
  
#### 加载、卸载及请求
  随着Zend引擎的运行，它处理来自客户端的一个或多个请求。 在传统的CLI实现中，相应于进程的一个执行。然而，很多其他实现，最值得注意的是Apache模块，可以映射很多请求到单个PHP进程上。 PHP扩展可以因此在其生命周期中看到很多请求。
  
#### 概览
  * 在Zend API中，模块仅仅在相关的PHP进程启动的时候一次加载到内存中。 每个模块接受到一个模块初始化的函数调用，这个函数在zend_module中指定， 在加载的时候执行。
  * 不管什么时间相关的PHP进程启动处理来自客户端的请求-- 例如，当php解释器被告知开始工作--每个模块接收到一个请求初始化的函数调用，这个函数也是在zend_module中指定的。
  * 当相关的PHP进程完成一个请求的处理，每个模块接受到请求终止的函数调用。同样也是在zend_module中指定的。
  * 当相关PHP进程关闭给定模块从内存中按一定顺序卸载， 模块会接收到模块终止函数调用，同样也是在zend_module中指定的。

#### 做什么，什么时候做??
  在这四个阶段可以有很多任务可执行。 下面的表陈述了一些通用的初始化和终止任务的所属关系:
```
模块初始化/终止                   请求初始化/终止
分配/释放以及初始化模块全局变量   分配/释放以及初始化特定请求的变量
注册/注销类实体
注册/注销ini实体
注册常量
```

#### phpinfo()回调
  除了全局初始化和特定很少使用的回调，有一个模块生命周期的一部分来检查:调用phpinfo(). 从这个调用输出用户可见，是否文本或HTML或者其他一些格式，这些是在调用时候由每个加载到php解释器的独立扩展产生的。
  
  为了提供格式化自然输出， 头ext/standard/info.h提供了一个函数数组来产生标准显示元素。特别的， 几个创建熟悉的既存表格的函数如下:
  * php_info_print_table_start(): 在phpinfo()输出中打开一个table. 不带参数
  * php_info_print_table_header(): 在phpinfo()输出中打印一个table头。 接受一个参数，列数目，加上同样数目的char *参数，这些作为每个列的头。
  * php_info_print_table_row(): 在phpinfo()输出中打印表行。 接受一个参数，列数目，加上相同数目的char *参数，即每个列的文本内容。
  * php_info_print_table_end(): 关闭php_info_print_table_start()开启的表开始。不带参数。
  
  使用这四个函数，可以产生扩展特性组合的状态信息。这些信息就是counter扩展回调信息。

```
PHP_MINFO_FUNCTION(counter)
{
  char buf[10];
  
  php_info_print_table_start();
  php_info_print_table_row(2, "counter support", "enabled");
  snprintf(buf, sizeof(buf), "%ld", COUNTER_G(basic_counter_value));
  php_info_print_table_row(2, "Basic counter value", buf);
  php_info_print_table_end();
}
```
