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

  
  
