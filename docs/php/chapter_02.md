## 第二章 变量的里里外外
  每种编程语言共同点是信息的存储和检索；PHP也不例外。虽然很多语言需要先声明所有的变量，那么它们要保存的信息类型就固定了，PHP坚持程序员实时创建变量并且可以存储这个语言所能表达的任何类型的信息。 当存储信息需要时，在那一刻自动转换为所需类型。
  
  因为你已经在用户空间使用PHP了，这个概念，就是所谓的松散类型，应该对你不陌生。 在这一章中，你会看到这些信息是如何被PHP的父语言C所编码的， C语言是需要严格类型的。
  
  当然，编码数据仅仅是公式的一半。 跟踪信息的所有片段，每一个需要一个标签和容器。从用户空间的境界，你会发现这些就是变量名和作用域的概念。

----------------------------
### 数据类型
  PHP中数据存储单元基础就是所谓的zval, 或者称为Zend Value. 它非常简单，定义在Zend/zend.h头文件中， 有四个字段，格式如下:
```
typedef struct _zval_struct {
  zvalue_value value;
  zend_uint refcount;
  zend_uchar type;
  zend_uchar is_ref;
} zval;
```
  应该从直觉上可以知道这些成员中的大部分是基本存储类型: refcount无符号整型，type无符号字符类型，is_ref无符号字符类型。而value成员实际上是一个联合体定义，PHP5中定义如下:
```
typedef union _zvalue_value {
  long lval;
  double dval;
  struct {
    char *val;
    int len;
  } str;
  HashTable *ht;
  zend_object_value obj;
} zvalue_value;
```
  这个联合体允许Zend存储PHP很多不同类型的变量，有能力存储单个，统一结构的变量。
  Zend当前定义了8种类型的数据，如下所示:
  * IS_NULL: 该类型在变量首次使用时自动赋值给未初始化的变量， 也可以明确的在用户空间使用内置NULL常量赋值给变量。该变量类型提供了一个特殊的"non-value", 该值和布尔值FALSE和整数0相区别开来。
  * IS_BOOL: 布尔变量可以具有两种可能性，要么为TRUE, 要么为FALSE. 用户空间的控制结构if, while, 三元操作符(ternary), for中的条件表达式隐含的在计算时将条件转换为布尔类型。
  * IS_LONG: PHP中的整型类型，使用主机系统的有符号整型(signed long)数据类型。 在大多数32位平台这个量存储范围为-2147483648到+2147483647. 有一点例外，当用户空间脚本尝试存储整型值超过这个范围，自动被转换为双精度浮点数类型(IS_DOUBLE)
  * IS_DOUBLE: 浮点数据类型使用主机系统的有符号双精度数据类型(signed double). 浮点数存储没有确切的精度；值范围，最小2.225*10^(-308)到最大限制大约在1.798*10^308， 8字节表示。
  * IS_STRING: PHP的最通用数据类型是字符串，存储于方式和老练的C程序员期望的一样。一块内存区域，足够大保存字符串的所有字节/字符，为字符串分配一个指针存储在宿主zval中。
  * IS_ARRAY: 
  * IS_OBJECT:
  * IS_RESOURCE:
  
  上面的IS_*存储于元素的zval结构的type域下面，确定zval结构中元素value的那部分是它的值。

  最明显的检查值类型的方式可能是从给定的zval对其解除引用，用下面的代码片段:
```
void describe_zval(zval *foo)
{
  if(foo->type == IS_NULL) {
    php_printf("The variable is NULL");
  } else {
    php_printf("The variable is of type %d", foo->type);
  }
}
```
  很明显，但是错了。
  那么好，不是错误，但是至少不是最优的方式。 Zend头文件包含了大量的zval访问宏，扩展作者希望使用它们来检测zval数据。 基本原因在于避免不兼容，当如果引擎的API发生了改变，但是但是代码的负面优势通常变得容易理解。 这里是那个相同的代码片，使用了Z_TYPE_P()宏:
```
void describe_zval(zval *foo)
{
  if(Z_TYPE_P(foo) == IS_NULL) {
    php_printf("The variable is NULL");
  } else {
    php_printf("The variable is of type %d", Z_TYPE_P(foo));
  }
}
```
  _P后缀表明传入的参数包含了单个间接层。该集合还有两个宏: Z_TYPE()和Z_TYPE_PP(), 前者期望zval的类型参数没有间接层， 后者希望有两个间接层(zval **).
  
  不过PHP7对这个结构做了些颠覆。后面研究PHP7的数据类型及存储原理。
  
> 注意， 这个例子中的特殊输出函数php_printf(), 用于显示一片数据。 这个函数语法上等价于stdio的printf()函数；然而，它为web服务器SAPI处理特殊进程，并且采用了PHP输出缓冲机制。 在后面第5章的你会了解更多关于此函数的情况。

============================

### 数据值
  使用数据类型type, zval的值可以使用三种宏来检测。 这些宏也是以Z_开头，选择性的以_P或_PP，依赖于间接引用的度。
  
  对于简单标量类型，布尔值，长整型和浮点数， 这些宏简短且一致: BVAL, LVAL, DVAL.
```
void display_values(zval boolzv, zval *longpzv, zval **doubleppzv)
{
  if(Z_TYPE(boolzv) == IS_BOOL) {
    php_printf("The value of the boolean is: %s\n", Z_BVAL(boolzv) ? "true" : "false");
  }
  
  if(Z_TYPE_P(longpzv) == IS_LONG) {
    php_print("The value of the long is: %ld\n", Z_LVAL_P(longpzv));
  }
  
  if(Z_TYPE_PP(doubleppzv) == IS_DOUBLE) {
    php_printf("The value of the double is: %f\n", Z_DVAL_PP(doubleppzv));
  }
}
```
  字符串变量，因为它包含两个属性，有一对三胞胎的宏代表char *(STRVAL), int(STRLEN)元素:
```
void display_string(zval *zstr)
{
  if(Z_TYPE_P(zstr) != IS_STRING) {
    php_printf("The wrong datatype was passed!\n");
  }
  
  PHPWRITE(Z_STRVAL_P(zstr), Z_STRLEN_P(zstr));
}
```

  数组数据类型存储在内部结构HashTable *， 可以使用ARRVAL三胞胎:Z_ARRVAL(zv), Z_ARRVAL_P(pzv), Z_ARRVAL_PP(ppzv). 当通过PHP核心和PECL模块的旧代码查看，你可能会遇到HASH_OF()宏， 它参数为zval *, 该宏一般等价于Z_ARRVAL_P()宏；然而，这种用法被废弃了，新代码中尽量不要使用。
  
  对象代表复杂内部结构， 有多个访问宏: OBJ_HANDLE，返回处理标识符， OBJ_HT处理表，OBJCE类定义， OBJPROP属性HashTable, OBJ_HANDLER操作OBJ_HT表中的特定处理器方法。 不要担心这些变化多样的对象宏， 后面在第10章，11章会详述它们。
  
  在zval中， 资源数据类型存储为简单整型，可以使用RESVAL三胞胎来访问。 这个整数传入zend_fetch_resource()函数，从数个标识符中查找注册的资源。资源数据类型会在后面第9章中介绍。
  
  
===========================


### 数据创建
  到目前为止，你知道如何从zval中取出所需数据，是时候创建一些你自己的数据了。 虽然zval能在函数顶部简单声明为直接变量， 它将使得变量数据存储局部性，为了离开函数到达用户空间， 需要对其进行拷贝。
  
  `因为你几乎总是希望你创建的zval以某种形式到达用户空间，你将要分配一块内存并赋值这个块给zval*指针。再一次很明显，解决方案使用malloc(sizeof(zval))不是正确的答案。 相反，你将使用另外一个Zend宏: MAKE_STD_ZVAL(pzv). 这个宏将在其他zval旁边以优化的内存块分配空间， 自动处理内存不足的问题(下一章会介绍这块), 并初始化新zval的refcount和is_ref属性。`
> 注意: 除了MAKE_STD_ZVAL(), 你会经常看到PHP源代码中另外一个zval*创建宏:ALLOC_INIT_ZVAL(). 这个宏和MAKE_STD_ZVAL()的区别仅仅在于，初始化zval *的数据类型为IS_NULL;

  一旦数据存储空间可用， 是时候为你全新的zval产生一些信息。 在前面数据存储上读取部分，你可能都准备使用Z_TYPE_P()和Z_SOMEVAL_P()宏来设置你的新变量。看起来这个明显的解决方案正确的?
  
  然而再次失败!
  
  Zend另外暴露了一系列宏来设置zval *的值。 下面是这些宏以及如何扩展这个到你已经熟悉的那些:
```
ZVAL_NULL(pzv)  Z_TYPE_P(pzv) = IS_NULL;
```
  虽然这个宏和直接版本没有节省什么， 它包括下列完整的宏:
```
ZVAL_BOOL(pzv, b) Z_TYPE_P(pzv) = IS_NULL;
                  Z_BVAL_P(pzv) = b ? 1 : 0;
                  
ZVAL_TRUE(pzv);   ZVAL_BOOL(pzv, 1);
ZVAL_FALSE(pzv);  ZVAL_BOOL(pzv, 0);
```

  `注意，任何非零值提供给ZVAL_BOOL()都会导致返回true事实。这很明显，因为任何非零值类型在用户空间都被强制转换为布尔类型。当硬编码到内部代码中， 最好明确使用1作为true事实。 宏ZVAL_TRUE()和ZVAL_FALSE()提供了这种便利，有时候让代码更加易于理解。`
  
```
ZVAL_LONG(pzv, l);    Z_TYPE_P(pzv) = IS_LONG;
                      Z_LVAL_P(pzv) = l;

ZVAL_DOUBLE(pzv, d);  Z_TYPE_P(pzv) = IS_DOUBLE;
                      Z_DVAL_P(pzv) = d;
```
  基本标量宏就像它们一样简单。 设置zval的类型，赋予一个数值给它们。
  
```
ZVAL_STRINGL(pzv, str, len, dup); Z_TYPE_P(pzv) = IS_STRING;
                                  Z_STRLEN(pzv) = len;
                                  if(dup) {
                                    Z_STRVAL_P(pzv) = estrndup(str, len + 1);
                                  } else {
                                    Z_STRVAL_P(pzv) = str;
                                  }
                                  
ZVAL_STRING(pzv, str, dup);     ZVAL_STRINGL(pzv, str, strlen(str), dup);
```

  这里才是zval创建越来越有趣的地方。字符串，数组，对象以及其他资源，都需要分配额外内存来存储数据。 下一章会介绍内存管理的陷阱；到现在为止，仅仅注意到复制值1会分配新内存，并拷贝字符串的内容，而值0仅仅是简单的将zval指向已经存在的字符串数据。
  
```
ZVAL_RESOURCE(pzv, res);       Z_TYPE_P(pzv) = IS_RESOURCE;
                               Z_RESVAL_P(pzv) = res;
```

  回忆一下前面的资源在zval里边仅仅使用一个整型存储。因此ZVAL_RESOURCE()和ZVAL_LONG()宏扮演的行为一致，只是使用的类型不同而已。

===========================


### 数据存储
  你已经在用户空间使用过PHP的一些东西，那么你应该比较熟悉数组。数个PHP变量(zval)可以放入单个容器中(array), 并给每个元素一个数字或字符串作为名字。
  
  PHP中每个单独变量都能在数组中找到这种希望并不奇怪。当你创建变量的时候，通过赋一个值给它，Zend存储这个值到内部数组，即所谓的符号表(symbol table).
  
  在符号表(也就是定义全局作用域的表)是在请求启动的点，在扩展的RINIT被调用之前， 然后在脚本完成，后续的RSHUTDOWN方法被执行之后销毁。
  
  当用户空间函数或对象被调用，为那个函数或方法分配新的符号表， 被定义为活动符号表(active symbol table). 如果当前脚本执行不是函数或方法， 全局符号表被认为是活动的。
  
  看一看扩展执行globals结构(在Zend/zend_globals.h中定义)， 你将找到下面两个元素定义:
```
struct _zend_execution_globals {
  ...
  HashTable symbol_table;
  HashTable *active_symbol_table;
  ...
}
```
  `symbol_table，使用EG(symbol_table)访问， 总是全局变量作用域就像用户空间的$GLOBALS变量总是相应于PHP脚本的全局作用域。 实际上，从内部来看$GLOBALS变量仅仅是用户空间里边对EG(symbol_table)变量的包装。`
  
  `另外一个变量active_symbol_table， 类似通过EG(active_symbol_table)来访问， 代表那些在此刻活跃的那些变量作用域。`
  
  `这里需要注意的关键不同是EG(symbol_table), 不像你在使用PHP和Zend APIs使用和遇到的其他的HashTable，直接是一个变量。 几乎所有在HashTable上操作的函数，期望一个直接的HashTable *作为它们的参数。` 因此，使用EG(symbol_table)作为那些函数的参数时，需要使用&取地址。
  
  考虑下面的两个代码块， 功能相同:
```
<?php $foo = 'bar'; ?>
```

  用C实现:
```
{
  zval *fooval;
  MAKE_STD_ZVAL(fooval);
  ZVAL_STRING(fooval, "bar", 1);
  ZEND_SET_SYMBOL(EG(active_symbol_table), "foo", fooval);
}
```

  首先，使用MAKE_STD_ZVAL()分配一个新的zval, 然后初始化其值为"bar", 然后使用一个新宏，等价于(=)赋值操作， 使用标签foo组合那个值， 然后将它添加到活跃的符号表中。 因为当前没有用户空间函数是活跃的，EG(active_symbol_table) == &EG(symbol_table), 这最终意思就是改变量被存储在全局作用域中。

==========================


### 数据检索
  为了从用户空间检索数据， 需要查找它们存储在哪些符号表中。 下面代码展示了使用zend_hash_find()函数达到这个目的。
```
{
  zval **fooval;
  
  if(zend_hash_find(EG(active_symbol_table),
    "foo", sizeof("foo"),
    (void**)&fooval) == SUCCESS) {
    php_printf("Got the value of $foo!");
  } else {
    php_printf("$foo is not defined.");
  }
}
```

  这个例子的一些部分看起来有点滑稽。 为什么fooval定义为两级间接引用? 为什么使用sizeof()来确定foo的长度?`为什么计算zval***，强制转换为void**?` 如果问你自己这三个问题，拍拍你自己的背.
  
  首先， 值得了解的是HashTable不仅仅用于用户空间变量。 HashTable结构那么多才多艺，它用于整个引擎，在某些情况下，使得想要存储非指针值变得非常完美。 HashTable的桶是固定大小的，然而，为了存储任意尺寸的数据，HashTable将分配一块内存来包装待存数据。 在变量的情况下，`zval*被存储，因此HashTable存储机制分配了一块内存足够大去容纳指针。HashTable的桶使用这个带有zval*的新指针，那么你能有效在HashTable内部使用zval**结束。 下一章介绍HashTable明显有能力存储zval,而实际却存储zval*. `
  
  

==========================


### 数据转换

=========================


### 总结
