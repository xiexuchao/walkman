### 第5章 你的第一个PHP扩展
  每个PHP扩展至少由两个文件组成: 一个配置文件，告诉编译器要编译什么扩展库需要的文件，另外一个是源代码文件，就是做实际事情的文件。
  
---------------------------------

#### 扩展剖析(Anatomy of an Extension)
  实际情况下，一般由第二个或第三个配置文件，以及多个头文件。你的第一个扩展，使用每个这种类型的文件一个文件，并添加到那里。

##### 配置文件(config.m4)
  这里仅仅介绍*nix下面的。首先在你的源代码目录的ext/下面创建一个扩展目录sample. 现实中这个新目录可以放在任何位置，但是为了演示Win32,以及本章后面的构建选项， 我将要求你一次将这些放在一起。
  
  下一步，进入这个新扩展目录，创建一个config.m4文件，内容如下:
```
PHP_ARG_ENABLE(sample,
  [Whether to enable the "sample" extension],
  [ enable-sample Enable "sample" extension support])

if test $PHP_SAMPLE != "no"; then
  PHP_SUBST(SAMPLE_SHARED_LIBADD);
  PHP_NEW_EXTENSION(sample, sample.c, $ext_shared)
fi
```
  最小的配置./configure设置选项是"enable-sample". PHP_ARG_ENABLE第二个参数将被显示，在./configure处理到达这个扩展的配置文件的时候。第三个参数，是用户使用./configure --help的时候，显示给用户的可用选项。
  
> 注意，曾经想要知道为什么一些扩展配置使用enable-extname, 而另外的却使用with-extname？功能上的，两者没有什么区别。 然而实际上，enable意味着这个特性可以被开启而且无需第三方的类库， with，相应的，意思是这个扩展的特性想要一些这样的依赖。
> 到目前为止， sample扩展不需要链接其他类库， 因此你使用enable版本。 第17章，扩展类库将介绍使用with， 并介绍编译器来使用额外的CFLAGS和LDFLAGS设置。

  如果中断用户调用./configure使用--enable-sample选项， 然后本地环境变量， $PHP_SAMPLE将被设置为yes, PHP_SUBST()是一个PHP修改版本的标准autoconf的AC_SUBST()宏，必要启用构建扩展为共享模块。
  
  最后但并非最不重要的(Last but not least), PHP_NEW_EXTENSION()声明该模块并列举(enumerate)所有源代码文件必须编译为扩展的部分。 如果需要多个文件，那么在第二个参数中以空格分割列出，例如:`PHP_NEW_EXTENSION(sample, sample.c sample2.c sample3.c, $ext_shared)`
  
  最后一个参数是对应于PHP_SUBST(SAMPLE_SHARED_LIBADD)命令，同样的也是构建共享模块。
  
##### 头文件
  当用C开发时，通常都搞清楚特定类型的数据分离到外部头文件中， 然后被源代码文件包好起来。虽然PHP不需要这样，但是当模块变大，这样的分法比单个文件要简单得多。
  
  你可以在你的头文件中使用下面的代码开头，这里头文件叫做php_sample.h:
```
#ifndef PHP_SAMPLE_H
/** prevent double inclusion **/
#define PHP_SAMPLE_H

/** define extension properties **/
#define PHP_SAMPLE_EXTNAME "sample"
#define PHP_SAMPLE_EXTVER "1.0"

/** import configure options when building outside of the php source tree **/
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

/** Include php standard header **/
#include "php.h"

/** define the entry point symbol 
 * Zend will use when loading this module
 */
extern zend_module_entry sample_module_entry;
#define phpext_sample_ptr &sample_module_entry;

#endif /** PHP_SAMPLE_H **/
```
  这个头文件完成个基本任务: 如果扩展是使用phpize工具(本书大部分都通过这个工具构建)构建，那么HAVE_CONFIG_H就会得到已经定义，然后会也会包含config.h。不管扩展时如何被编译的，它都要从PHP源代码树中包含php.h。 这个头文件随后包含了数个散布在PHP源代码中的其他头，提供各种PHP API访问的集合。
  
  下一步，zend_module_entry结构用于扩展，声明为external, 这样可以被Zend使用dlopen()和dlsym()捡起，当模块使用extension=xxx行加载的时候。
  
  这个头文件也包含了一些预处理定义，源文件中会使用它们。
  
##### 源代码
  最后但不是代表不重要， 你将创建一个简单的源代码骨架文件sample.c
```
#include "php_sample.h"

zend_module_entry sample_module_entry = {
#if ZEND_MODULE_API_NO >= 20010901
  STANDARD_MODULE_HEADER,
#endif
  PHP_SAMPLE_EXTNAME,
  NULL, /* functions */
  NULL, /* MINIT */
  NULL, /* MSHUTDOWN */
  NULL, /* RINIT */
  NULL, /* RSHUTDOWN */
  NULL, /* MINFO */
#if ZEND_MODULE_API_NO >= 20010901
  PHP_SAMPLE_EXTVER,
#endif
  STANDARD_MODULE_PROPERTIES
};

#ifdef COMPILE_DL_SAMPLE
ZEND_GET_MODULE(sample)
#endif
```
  这就是全部， 这三个文件就是创建模块骨架需要的全部。 老实说，它不做什么有意义的事情，但是它却是开头， 你将能添加功能性通过本节的其他部分。首先，让我们看看发生了什么。
  
  开头一行非常简单，就是包含刚创建的头文件， 以及扩展需要的代码树中其他核心头文件。
  
  下一行，创建zend_module_entry结构，就是在头文件中声明的。 你将注意到模块入口的一个元素是条件列出的，基于ZEND_MODULE_API_NO定义。 这个API数字一般等于PHP的4.2.0, 因此如果你知道你的扩展根本不会构建在这些较旧的版本构建中，那么就不用使用这些条件判断， 只需要包含STANDARD_MODULE_HEADER元素即可。
  
  然而考虑到这个条件语句对编译时间的影响非常小，对结果要处理的二进制文件影响非常小，因此多数情况下，最好都带上这个条件检查。 这同样适用后面的模块版本的条件编译。
  
  其他的六个元素，开始都设置为NULL, 可以根据后面的关于具体做什么的注释看到一些提示。
  
  最后，代码最下面的你看到一个对每个PHP扩展都是通用的， 也就是能构建为共享模块的。 这个主要条件简单的添加一个引用，Zend使用它当扩展是动态加载的。不要担心它做了些什么，以及如何做这些，只要确保它被包围着，否则下面的节不会工作。
  
=========================================

#### 构建你的第一个扩展
  到目前为止，所有文件都各就各位了，是时候走起了。 在构建PHP二进制主要部分的时候，有不同的步骤采取，依赖于你是在*nix还是Windows上面编译。
  
##### 在*nix下面构建
  首先使用config.m4模版产生一个./configure脚本使用的信息。这个可以使用phpize工具，就是你安装的主PHP后安装的phpize.
```
Configuring for:
PHP Api Version:         20121113
Zend Module Api No:      20121212
Zend Extension Api No:   220121212
```
> 注意，Zend Extension Api No行前面有两个2，不是错别字； 它是针对于Zend Engine 2版本，意味着保持这个API数字比它的ZE1同伙要高些。

  如果这时候你再看当前目录，你会发现比原来多了大量的文件。 phpize程序将你扩展配置文件config.m4中的信息结合你的PHP构建以及所有必要片段产生一个编译，编译发生。 这意味着你不得不使用makefiles争斗并定位PHP的头文件，在你将再次编译的时候。PHP已经为你做了这些事情。
  
  下一步是简单的./configure， 你可能使用其他OSS包执行。你不是配置整个PHP一起，只是你自己的扩展， 因此所有你需要的输入如下:
```
$ ./configure enable-sample
```
  注意这里甚至没有使用enable-debug和enable-maintainer-zts。 这是因为phpize从PHP主体构建中已经带了这些， 并应用到你的扩展的./configure脚本。
  
  现在构建它!就像其他的包一样， make && make install, 然后产生脚本文件会处理其他的。
  
  当构建过程完成， 你会得到一个sample.so文件，该文件被安装到模块目录。 
  
##### 在Windows上面构建， 这里忽略掉。

##### 加载扩展构建为共享模块
  在请求的时候为了让PHP定位这个模块，需要在php.ini中使用extension=sample.so来动态加载这个模块。 具体的这里不介绍了。
  

=========================================

#### 静态构建
  在加载的模块列表中，你可能注意到，很多模块被列出，但是却没有出现在扩展指令extension=xxx中。这些模块是直接构建到PHP中，并编译为主构建过程的一部分。
  
##### 在*nix中构建静态模块
  此处略去。

=========================================

#### 功能函数集
  用户空间和扩展代码之间的快捷链接是PHP_FUNCTION(). 在你的sample.c顶部添加如下代码块，刚好在引入模块头文件的下面:
```
PHP_FUNCTION(sample_hello_world)
{
  php_printf("Hello World\n");
}
```
  PHP_FUNCTION宏就像正常的C函数声明，因为它就是完全如下扩展来的:
```
#define PHP_FUNCTION(name) \
  void zif_##name(INTERNAL_FUNCTION_PARAMETERS)
```
  这个例子情况计算为:
```
void zif_sample_hello_world(zval *return_value, char return_value_used, zval *this_ptr TSRMLS_DC)
```
  当然，简单声明函数还不够。引擎需要知道函数地址，以及用户空间期望的函数名字如何。这时下面的代码块完成的，你可能会立即在PHP_FUNCTION()块之后放置:
```
static function_entry php_sample_functions[] = {
  PHP_FE(sample_hello_world, NULL)
  {NULL, NULL, NULL}
};
```
  php_sample_functions向量是一个简单的NULL终止向量，当你继续添加函数到同一模块， 每个函数你暴露出去都出现为该向量的一个元素。 使用PHP_FE()宏，扩展后为:
```
{"sample_hello_world", zif_sample_hello_world, NULL},
```
  因此既为新函数提供了名字，也提供了一个实现函数的指针。 第三个参数用于提供参数提示信息，比如需要特定引用方式的参数传入。在第7章会看到这些特性。
  
  因此现在你已经有了一系列可暴露的函数李彪，但是仍然没有链接到引擎。 这时使用sample.c最后改变完成的，就是在前面sample_module_entry里边的functions部分替换为php_sample_functions,(确保后面要有逗号)
  那么现在可以使用前面的指令重新构建， 并使用-r测试它。
  
  都运行的不错，后面你会看到输出"Hello World!".
  
##### Zend内部函数
  内部函数名称的zif_前缀代表的是"Zend Internal Function", 用于避免可能的符号表冲突。例如，用户空间的strlen()函数可以实现为void strlen(INTERNAL_FUNCTION_PARAMTERS), 那么它就和C库的strlen实现冲突了。
  
  有时候，即使默认的前缀zif_也不能满足。通常这时因为函数名扩展为另外一个宏，被C编译器搞误解了。这种情况下，内部函数可以使用PHP_NAMED_FUNCTION()宏来给定一个随意名字，例如，PHP_NAMED_FUNCTION(zif_sample_hello_world)并表示为前面使用的PHP_FUNCTION(sample_hello_world).
  
  当使用PHP_NAMED_FUNCTION()声明的实现，PHP_NAMED_FE()宏用于链接到function_entry向量。 因此，如果你声明你的函数为PHP_NAMED_FUNCTION(purplefunc), 你将使用PHP_NAMED_FE(sample_hello_world, purplefunc, NULL)而不使用PHP_FE(sample_hello_world, NULL);
  
  这个实践可以见于ext/standard/file.c，fopen()函数实际上是使用PHP_NAMED_FUNCTION(php_if_fopen)声明的，用户空间关心的，通常没有关于函数别的，它仍然是通过fopen()调用的。在内部， 这个函数是被保护的， 防止预处理器宏和帮助编译器。
  
##### 函数别名
  有些函数可以通过多个名字引用。回忆一下内部声明的普通函数作为用户空间的函数名，使用zif_前缀，很容易看到PHP_NAMED_FE()宏将用于创建可选的映射:
```
PHP_FE(sample_hello_world, NULL)
PHP_NAMED_FE(sample_h1, zif_sample_hello_world, NULL)
```
  PHP_FE()宏将用户空间函数sample_hello_world和zif_sample_hello_world联系起来。PHP_NAMED_FE()宏然后将用户空间函数sample_hi使用相同的内部函数实现。
  
  

=========================================

#### 总结

=========================================
