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
字段            值                                描述
-----------------------------------------------------------------------------------
size            sizeof(zend_module_entry)         结构的字节长度
zend_api        ZEND_MODULE_API_NO                该模块被编译的Zend API版本
zend_debug      ZEND_DEBUG                        标志，表明模块是否需要使用调试开始的方式编译
zts             USING_ZTS                         标识表明是否该模块需要使用线程安全开启编译
ini_entry       NULL                              这个指针用于zend内部来保持非局部引用到该模块声明的任意INI实体
deps            NULL                              指向模块依赖列表的指针
name            "mymodule"                        模块的名称。这个是简短名称，例如"spl"或"standard".
functions       mymodule_functions                指向模块函数表的指针， zend用于向用户空间暴露功能。
```
