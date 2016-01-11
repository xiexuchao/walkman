### 第三章 内存管理

  托管语言例如PHP和非托管语言比如C的最大不和谐的区别在于内存指针的管控。
  
-------------------------

#### 内存
  在PHP中，产生一个字符串变量非常简单`<?php $str = "hello world";?>`, 并且字符串可以随意修改，赋值，移动。 另一方面，在C语言中，虽然你能使用诸如此类的`char *str = "hello world";`, 这个字符串不能修改，因为它位于程序空间。要创建一个可操作字符串，你将不得不分配一块内存以及通过诸如strdup()这样的函数拷贝这些内容。
```
{
  char *str;
  str = strdup("hello world");
  if(!str) {
    fprintf(stderr, "Unable to allocate memory!");
  }
}
```
  本章你将探索的几个原因在于，传统内存管理函数(malloc(), free(), strdup(), realloc(), calloc()等等)基本上都不能通过PHP源代码直接使用。

##### 释放malloc分配的内存
  几乎所有平台的内存管理都是基于请求内存、释放内存的处理过程。 应用程序与其上面的层进行通话(通常是操作系统)"我需要用一些字节内存来使用"。 如果还有空间可用，操作系统提供内存给程序，并标记这个内存块不再给其他应用程序使用。
  
  当应用程序使用内存操作完成，它希望将内存返还给操作系统， 以便能分配给其他有需要的应用。如果程序不归还内存，操作系统没有办法知道它不再使用那些内存， 也就无法分配给其他进程使用。 如果内存块没有释放， 拥有内存的应用又丢失那块内存的踪迹，这就出现了所谓的泄漏， 因为这块内存不能再被其他进程使用了。
  
  在典型的客户端应用程序中，较小的罕见内存泄漏有时候可以容忍，因为知道进程将不久后会停止，泄漏的内存将会隐含的返还给操作系统。 这也不是什么伟大壮举，因为操作系统知道内存给了什么程序，并且当程序终止的时候，它可以确定这块内存不再会使用。
  
  对于长时间运行的守护进程来说， 包括web服务器比如Apache以及mod_php扩展来说，进程被设计成长时间运行的， 通常是无限时间。 因为操作系统不能清理使用的内存，任何程度的泄漏不管多小，时间长了都会变大，终究会耗尽系统资源。
  
  考虑下用户空间的stristr()函数；使用大小写不敏感的匹配查找一个字符串， 它实际上创建了要查找的字符串和被查找字符串的小写拷贝， 然后执行一个更加传统的大小写敏感匹配来查找相应的偏移位。 在字符串的偏移位定位到之后，然而，不再使用小写版本的查找字符串和被查找字符串。如果不释放这些拷贝， 那么每个脚本都使用一下stristr(), 随着时间推移将泄漏一些内存。最终web服务器进程将拥有所有的系统内存， 但是没有使用它们。
  
  理想的解决方式是，你会喊叫，写良好的、干净的、一致的代码， 确实这是绝对正确的。 在PHP解释器这样的环境里，然而，这仅仅是解决了一半。
  
##### 错误处理
  为了给到用户空间脚本和它依赖的扩展的活动请求提供保释的能力，需要存在一种突然从活动请求实体跳出的方式。 这种方式是在Zend引擎里处理的，在请求开始的时候设置一个保释地址， 然后在任何die()或exit()被调用或遇到任何致命错误(E_ERROR)的时候执行一个longjmp()到保释地址。
  
  虽然这种保释地址简化了程序流， 它几乎总是意味着资源清理代码(比如free())将被跳过，内存将会泄漏。考虑下面简单的引擎处理函数调用的代码版本。
```
void call_function(const char *fname, int fname_len TSRMLS_DC)
{
  zend_function *fe;
  char *lcase_fname;
  
  lcase_fname = estrndup(fname, fname_len);
  zend_str_tolower(lcase_fname, fname_len);
  
  if(zend_hash_find(EG(function_table),
    lcase_fname, fname_len + 1, (void **)&fe) == FAILURE) {
    zend_execute(fe->op_array TSRMLS_CC);
  } else {
    php_error_docref(NULL TSRMLS_CC, E_ERROR,
      "Call to undefined function: %s()", fname);
  }
  efree(lcase_fname);
}
```
  当遇到php_error_docref()行，内部错误处理器看到这个错误级别是致命的并调用longjmp()打断当前程序流，并离开call_function()甚至没有到达efree(lcase_fname)行。你可能认为efree行可能被移到上面的zend_error()行里边，但是关于首次调用call_function实例的代码地方在哪呢？特别是类似fname自身分配的字符串以及你不能释放它，在没有使用错误信息的时候。
  
> 注意 php_error_docref()函数是一个内部等价与TRigger_error(). 第一个参数是可选的文档引用位置将被附加到docref.root, 如果在php.ini中启用的话。第三个参数可以是任意的家族E_*,常数表明显示度。 第四个和后面的参数跟printf()样式一样， 是可变参数列表。

##### Zend内存管理
  对请求应急救援(bailout)的解决是在Zend内存管理层(ZendMM). 引擎的这部分扮演的角色类似于操作系统正常扮演的， 为调用应用程序分配内存。 不同之处在于，它是足够低层的，在进程空间中是请求感知的，因此当请求die, 它能执行类似操作系统在进程die的时候执行的行为。 也就是说，它明确的释放了该请求的所拥有的内存。 下图展示了ZendMM和操作系统以及PHP进程的关系。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/php/zendmm_os_php_relation.png)
  
  总结: ZendMM在请求die掉的时候，释放该请求所使用的内存。 操作系统在进程die掉的时候，释放该进程所使用的内存。ZendMM粒度更细腻些。
  
  除了提供隐式的内存清理，ZendMM也根据php.ini中的设置控制每个请求内存使用: memory_limit。 如果脚本尝试请求多于系统整体可用的内存，或者超过它的每个请求持有限制，ZendMM会自动引发一个E_ERROR信息，然后开始紧急救助(bailout)进程。增加这个的好处是大多数内存分配调用的返回值无需检查，因为失败导致立即longjmp()到引擎的shutdown部分。
  
  在PHP内核和操作系统实际的内存管理层挂钩本身是通过不比引用所有内部内存分配所需要使用可替代的函数集合复杂实现的。例如，相对于分配16位的内存块使用malloc(16), PHP代码使用emolloc(16). 另外执行实际的内存分配任务，ZendMM将使用绑定的请求信息标记那个块，以便当请求紧急救助，ZendMM可以隐式的清理它。
  
  通常，内存需要分配分配的要比单个请求持久要长一点。 这些类型的分配，叫做持久分配，因为它们在请求结束还是持久的，可以使用传统的内存分配器因为它们不需要ZendMM为它添加额外的单个请求信息。有时候，它不知道知道运行时是否特定分配需要持久或非持久， 因此ZendMM暴露了一系列帮助宏扮演诸如其他内存分配的函数集合， 但是有一个额外参数表明持久性。
  
  如果你真正的希望持久分配，这个参数被设置为1， 这种情况下，请求将传入传统malloc()家族的分配器。 如果运行时逻辑确定这个块不需要持久化，这个参数将被设置为0， 调用将被沟道到每个请求内存分配器函数。
  例如， pemalloc(buffer_len, 1)映射到malloc(buffer_len), 而pemalloc(buffer_len, 0)映射到emalloc(buffer_len)使用下面的宏定义:
```
#define in Zend/zend_alloc.h:

#define pemalloc(size, persistent) \
  ((persistent) ? malloc(size) : emalloc(size))
```
  在ZendMM中可以找到的每一个分配器函数都可以在传统同行中找到相应的分配器。
  
  下表列出ZendMM支持的分配器函数以及它们的e/pe同行关系：
```
分配函数                          e/pe同行分配函数
--------------------------------------------------------------------------------
void *malloc(size_t count)        void *emalloc(size_t count);
                                  void *pemalloc(size_t count, char persistent);
--------------------------------------------------------------------------------
void *calloc(size_t count)        void *ecalloc(size_t count);
                                  void *pecalloc(size_t count, char persistent);
--------------------------------------------------------------------------------
void *realloc(void *ptr, size_t   void *erealloc(void *ptr, size_t count);
  count);                         void *perealloc(void *ptr, size_t count, char persistent);
--------------------------------------------------------------------------------
void *strdup(void *ptr)           void *estrdup(void *ptr);
                                  void *pestrdup(void *ptr, char persistent);
--------------------------------------------------------------------------------
void free(void *ptr);             void *efree(void *ptr);
                                  void *pefree(void *ptr, char persistent);
--------------------------------------------------------------------------------
```
  你会注意到即使pefree()都需要持久标志。 这是因为在pefree()被调用时，它实际上不知道ptr是否为持久分配的。调用free()在非持久分配可能导致少量的双重free, 而调用efree()在持久分配上，将最大可能导致段错误，因为内存管理尝试查找的管理信息不存在。 你的代码需要记住数据结构是分配为持久还是非持久的。
  
  另外核心的分配函数集合，少量增加相对方便的ZendMM函数存在:
  `void *estrndup(void *ptr, int len);`
  
  分配len+1字节内存，并拷贝ptr的len长度到新分配的块中。 estrndup的行为大致如下:
```
void *estrndup(void *ptr, int len)
{
  char *dst = emalloc(len + 1);
  memcpy(dst, ptr, len);
  dst[len] = 0;
  return dst;
}
```
  种植符号\0显式的添加到缓冲后面，确保任何使用estrndup()的函数对字符串赋值不需要担心传入的结果缓冲到函数期望终止字符串，例如printf(). 当使用estrndup()拷贝非字符串数据，最后一个字节必要浪费，但是更多情况下还是需要， 便利大于效率的减少。
  
```
void *safe_emalloc(size_t size, size_t count, size_t addtl);
void *safe_pemalloc(size_t size, size_t count, size_t addtl, char persistent);
```
  通过这两个函数分配的实际内存长度是((size * count) + addtl). 你可能会问， 为什么需要额外的函数? 为什么不用emalloc/pemalloc， 为什么我们自己不去匹配呢? 原因在于相同的词:安全。 虽然导致这种情况的基本不可能， 可能结尾导致这样等价会导致整型限制溢出主机平台。 这样可能导致分配负字节数字，或者更坏的情况， 负数明显限于调用程序希望得到的。 safe_emalloc()避免这种类型的错略，通过检查整型溢出，并明确防止诸如此类的溢出发生。
  
> 注意， 不是所有的内存分配运行时具有一个p*同行方法。 例如，PHP5.1之前perstrndup(), safe_pemalloc()不存在。 偶尔有时候你需要在Zend API中解决这些差异。

===========================
#### 引用计数
  注意内存分配和释放是像PHP这样的语言中多请求处理长期至关重要的术语，但是这只是蓝图的一半。 对服务器而言为了处理每秒成千上万的点击保证功能有效性，每个请求需要使用尽可能少的内存，以及执行最低限度的不必要的数据复制。考虑下面的PHP代码片段:
```
<?php
  $a = "Hello World";
  $b = $a;
  unset($a);
```
  第一条调用，单个变量被创建，12字节的内存块被赋予给它，并保存"Hello World"字符串，结尾以NULL结束。第二行代码，$b被设置成和$a同样的值，然后$a被释放掉.
  
  如果PHP把每个变量赋值都视为拷贝变量内容的原因，那么又需要额外的12字节来拷贝赋值字符串，额外的处理器负载将消耗在数据拷贝过程。当第三行出现，原来变量被释放，那么做的内容赋值变得完全没有必要，看起来很可笑。 那么现在继续深入，想象一下，如果有10MB的文件被加载到两个变量中。那样将需要20MB的空间，而实际上只需要10MB空间就足够了。 引擎有必要浪费这么多时间和内存在这样无用功上吗? 
  要知道PHP比上面的要明智多了。
  
  记住一点，变量名称和它们的值在引擎里边实际上是两个不同概念。变量值实际上是一个无名的zval*保存的，在刚的情况下，是字符串值。它被赋值给变量$a, 使用Zend的_hash_add(). 那么如果两个变量指向相同的值呢?
  
```
{
  zval *helloval;
  MAKE_STD_ZVAL(helloval);
  ZVAL_STRING(helloval, "Hello World", 1);
  zend_hash_add(EG(active_symbol_table), "a", sizeof("a"),
    &helloval, sizeof(zval *), NULL);
  
  zend_hash_add(EG(active_symbol_table), "b", sizeof("b"),
    &helloval, sizeof(zval *), NULL);
}
```
  在这一点上，你可能实际上观察到$a, $b都包含了相同的字符串"Hello World". 不幸的是，你遇到第三行unset($a);. 这种情况下，unset()不知道$a指向的数据$b也在使用，因此不能盲目的释放掉那个内存。任何后续对$b的访问查找那个已经被释放的内存空间将导致引擎崩溃。提示: 你的目的不是让引擎奔溃。 
  
  这是通过zval的第三个字段成员解决的: refcount. 当变量第一次被创建并设置，它的refcount初始化为1，因为它假设该变量仅仅被创建者使用。 当你的代码片段将helloval赋值给$b，需要增加refcount为2， 因为该值现在被两个变量引用:
```
{
  zval *helloval;
  MAKE_STD_ZVAL(helloval);
  ZVAL_STRING(helloval, "Hello World", 1);
  zend_hash_add(EG(active_symbol_table, "a", sizeof("a"),
    &helloval, sizeof(zval*), NULL);
  
  ZVAL_ADDREF(helloval);
  zend_hash_add(EG(active_symbol_table, "b", sizeof("b"),
    &helloval, sizeof(zval*), NULL);
}
```
  当使用unset()删除$a变量的拷贝版本，可以看到refcount参数，某些关心数据的应该实际上仅仅是减少refcount, 而不是让它孤立。
  
##### 写时复制
  通过引用计数节省内存确实是个好主意，但是如果当你希望仅仅修改这样变量中的其中一个时会发生什么呢? 考虑下面的代码片段:
```
<?php
  $a = 1;
  $b = $a;
  $b += 5;
```
  看看逻辑流，你将当然期望$a仍然是1，而$b现在应该是6. 在这个点上，你知道Zend尽量节省内存，在第二行代码后，通过$a, $b引用相同的zval， 那么当第三行出现的时候，$b必须被改变?
  
  答案是，Zend查看refcount, 看它是否大于1，并将其分离。 Zend引擎中的分离处理是通过销毁引用对，你只看到了相反的过程:
```
zval *get_val_and_separate(char *varname, int varname_len TSRMLS_DC)
{
  zval **varval, *varcopy;
  if(zend_hash_find(EG(active_symbol_table), varname, varname_len + 1, (void **)&varval) == FAILURE) {
    return NULL;
  }
  
  if((*varval)->refcount < 2) {
    /* varname is the only actual reference, no separating to do */
    return *varval;
  }
  
  /** otherwise, make a copy of the zval* value **/
  MAKE_STD_ZVAL(varcopy);
  varcopy = *varval;
  /** Duplicate any allocated structures within the zval* **/
  zval_copy_ctor(varcopy);
  
  /** Remove the old version of varname
   * this will decrease the refcount of varval in the process
   */
  zend_hash_del(EG(active_symbol_table), varname, varname_len + 1);
  
  /**
   * Initialize the reference count of the newly created value and attach it to the varname variable
   */
  varcopy->refcount = 1;
  varcopy->is_ref = 0;
  zend_hash_add(active_symbol_table), varname, varname_len + 1, &varcopy, sizeof(zval*), NULL);
  
  /** Return the new zval* **/
  return varcopy;
}
```

  现在引擎有了zval*, 它知道仅仅被$b所拥有，它可以转换它为长整型，并根据代码请求将其增加5.
##### 写时变化
  引用计数的概念也创建了一个新的数据操作可能性，以用户空间脚本实际认为的形式，用术语来说就是引用。 考虑下面的用户空间代码片段:
```
<?php
  $a = 1;
  $b = &$a;
  $b += 5;
```
  对PHP代码由经验，你本能的知道$a的值将改变为6.虽然最初为1，并且没有直接改变它。 这发生的变化在于，引擎通过增加5到$b的值， 注意$b是对$a的引用，也就是说:"对我来说改变这个值而无需分离它们， 因为我想让所有对它引用的变量都看到这种改变。"
  
  但是，引擎又是如何知道的呢? 简单的很，看zval的最后一个字段元素is_ref。 这个仅仅是一个on/off位，定义值是否用户空间样式的引用集合。 在之前的代码片段， 当第一行被执行， $a获得一个refcount为1， is_ref为0的zval, 因为它只属于$a变量， 没有其他变量有机会对其写引用。在第二行代码， refcount增加为2， 这次除了这点，脚本包括了一个&符号，表示完全引用， is_ref元素也被设置为1.
  
  最后，在第三行代码的时候，引擎一旦获得$b所关联的值，检查是否需要分离。 这次，变量值不分离，因为包含了一个前面没有包含的检查。 这就是refcount检查点get_var_and_separate(), 使用了另外的一个条件:
```
if((*varval)->is_ref || (*varval)->refcount < 2) {
  /** varname仅仅是实际引用，或者他是对其他变量的完全引用， 两种形式都不要执行分离操作 **/
  return *varval;
}
```
  这次，即使refcount为2， 分离处理也被短路，因为这个值是完全引用。引擎能自由修改它不用关心这个值的其他变量变量似乎神奇的自己改变它一样。
  
##### 分离焦虑(Separation Anxiety)
  使用这种拷贝和引用，有大量这种事件的组合不能通过is_ref和refcount明智的操作。 考虑下面的PHP代码情况:
```
<?php
  $a = 1;
  $b = $a;
  $c = &$a;
```
  这里有一个简单的值，需要和三个不同变量联系起来， 两个需要写时改变的完全引用对，第三个是写时复制分离上下文。 仅仅使用is_ref和refcount来描述关系， 那这些值怎么工作呢?
  
  答案是:什么也不是。这种情况下， 值必须复制为两个区别的zval*, 即使都包含了完全相同的数据。如下图:
  
  
===========================

#### 总结
