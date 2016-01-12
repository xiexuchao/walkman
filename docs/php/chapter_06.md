### 第六章 返回值
  用户空间函数使用return关键词传回信息给调用域， 和C里边你可能使用相同的模式。
  应用程序例如:
```
function sample_long() {
  return 42;
}
$bar = sample_long();
```
  当sample_long()被调用，数字42被返回， 产生到$bar变量中。在C中，这可能使用几乎等价的代码基准:
```
int sample_long(void) {
  return 42;
}
void main(void) {
  int bar = sample_long();
}
```
  当然，在C中，你已经知道什么函数被调用并准备返回基于它的函数原因，因此你可以声明变量，并相应的将结果存储在里边。当PHP用户空间处理时，然而，变量类型是动态的，你必须回到zval类型。

------------------------------

#### return_value变量
  你可能尝试去相信你的内部函数应该返回一个即刻zval, 或者更多可能分配内存为zval，然后返回一个zval*， 就像下面的代码块一样:
```
PHP_FUNCTION(sample_long_wrong)
{
  zval *retval;
  MAKE_STD_ZVAL(retval);
  ZVAL_LONG(retval, 42);
  return retval;
}
```
  不幸的是，你将被结束，最终错粗。而不是强迫每个函数实现分配一个zval并返回它，Zend引擎预先分配了这个空间，在方法被调用之前。 然后初始化zval的类型为IS_NULL, 并以参数的形式传入那个值，命名为return_value. 这时同样的函数， 实现正确的:
```
PHP_FUNCTION(sample_long)
{
  ZVAL_LONG(return_value, 42);
  return;
}
```
  注意PHP_FUNCTION()实现没有返回空。 相反，return_value参数直接被产生，使用恰当的数据， Zend引擎将处理这个值，在内部函数完成执行后。
  
  作为提醒，ZVAL_LONG()宏是对一系列赋值操作的包装，这种情况下是:
```
Z_TYPE_P(return_value) = IS_LONG;
Z_LVAL_P(return_value) = 42;
```
  或者更加原始的:
```
return_value->type = IS_LONG;
return_value->value.lval = 42;
```
> 注意，return_value的is_ref和refcount属性几乎根本不被内部函数直接修改。这些值被初始化并被Zend引擎处理，当调用你的函数的时候。

  让我们看看实战中这个特殊函数，通过添加到sample扩展中。 并在php_sample_functions结构中包含这个sample_long()的入口:
```
static function_entry php_sample_functions[] = {
  PHP_FE(sample_hello_world, NULL)
  PHP_FE(sample_long, NULL)
  {NULL, NULL, NULL}
};
```
  在这点上，扩展可以使用make重新构建。如果一切ok, 你可以运行PHP并检测你的新函数: `php -r 'var_dump(sample_long());'`

##### 紧紧的包装你的宏
  在可读、可维护的兴趣上，ZVAL_*()宏已经被复制的同行，被指定到return_value变量。 在每个例子中，宏的ZVAL部分都被替换为术语RETVAL, 然后初始化参数将被标注变量所属被修改忽略。
  
  在之前的例子中，sample_long实现可以精简如下:
```
PHP_FUNCTION(sample_long)
{
  RETVAL_LONG(42);
  return;
}
```
  下表列出了Zend引擎定义的RETVAL家族宏。 在所有情况除了两个，RETVAL宏被表示为ZVAL同行使用初始化return_value参数被删除。
```
返回值宏
------------------------------------------------------------------------------
一般化的ZVAL宏                                return_value特定的同行
ZVAL_NULL(return_value)                       RETVAL_NULL()
ZVAL_BOOL(return_value, bval)                 RETVAL_BOOL(bval)
ZVAL_TRUE(return_value)                       RETVAL_TRUE
ZVAL_FALSE(return_value)                      RETVAL_FALSE
ZVAL_LONG(return_value, lval)                 RETVAL_LONG(lval)
ZVAL_DOUBLE(return_value, dval)               RETVAL_DOUBLE(dval)
ZVAL_STRING(return_value, str, dup)           RETVAL_STRING(str, dup)
ZVAL_STRINGL(return_value, str, len, dup)     RETVAL_STRING(str, len, dup)
ZVAL_RESOURCE(return_value, rval)             RETVAL_RESOURCE(rval)
-------------------------------------------------------------------------------
```
> 注意TRUE, FALSE宏没有括号。这些视为Zend/PHP编码标准的畸变,但是保留主要是为了向后兼容。如果你构建的扩展受到一个undefined macro RETVAL_TRUE()错误， 确保不要包含括号。

  相当常见的是，在你函数已经使用了一个返回值，它将准备退出，并返回控制给调用域。 对于这种原因，这些退出的一个或多个为内部函数特殊设计的宏集合: RETURN_*()家族:
```
PHP_FUNCTION(sample_long)
{
  RETURN_LONG(42);
}
```

  虽然不是实际可见，但是这个函数仍然在RETURN_LONG()宏被抵用结束明确返回。这个可以通过在RETURN_LONG()后面添加一个行php_printf()调用来测试。
```
PHP_FUNCTION(sample_long)
{
  RETURN_LONG(42);
  php_printf("I will never be reached.\n");
}
```
  php_printf(), 就像它内容所暗示的，根本不会执行，因为RETURN_LONG()调用明确的离开了函数。
  
  不像RETVAL系列，RETURN同行为每种上面列出的类型都存在一种对应的。 也像RETVAL系列，RETURN_TRUE, RETURN_FALSE宏没有使用括号。
  
  更加复杂的类型，比如对象和数组，也是通过return_value参数返回的。然而，它们的自然排除简单宏基于创建方法。即使资源类型，当它有一个RETVAL宏，需要额外工作来产生。在第8到11章你会看到如何返回这些类型的值。
  
##### 是否值得麻烦呢?
  Zend内部函数的一个未充分利用的特性是return_value_used参数。考虑下面的用户空间代码:
```
function sample_array_range() {
  $ret = array();
  for($i = 0; $i < 1000; $i++) {
    $ret[] = $i;
  }
  
  return $ret;
}
sample_array_range();
```
  因为sample_array_range()被调用，而没有存储结果到变量中， 工作和内存使用创建了1000个元素的数组完全浪费了。 当然，以这种方式调用sample_array_range()相当愚蠢， 但是如果提前知道这些会不会好呢?
  
  虽然对用户空间函数不可访问，但内部函数可以条件性跳过，否则无指针行为就像这个，依赖于return_value_used参数的设置， 对所有内部函数常见。
  
```
PHP_FUNCTION(sample_array_range)
{
  if(return_value_used) {
    int i;
    array_init(return_value);
    for(i = 0; i < 1000; i++) {
      add_next_index_long(return_value, i);
    }
    
    return;
  } else {
    php_error_docref(NULL TSRMLS_CC, E_NOTICE,
      "Static return-only function called without processing output");
    RETURN_NULL();
  }
}
```
  要看这个函数的操作，在前面的sample.c中添加下面的行到php_sample_functions定义里边即可:`PHP_FE(sample_array_range, NULL)`
  
##### 返回引用值
  从用户空间的工作你已经知道， PHP函数可以通过引用返回值。因为实现的问题，从内部函数返回引用应该避免在PHP5.1之前使用，因为不工作。考虑下面的用户空间代码片段:
```
function &sample_reference_a() [
  if(!isset($GLOBALS['a'])) {
    $GLOBALS['a'] = NULL;
  }
  
  return $GLOBALS['a'];
}

$a = 'Foo';
$b = sample_reference_a();
$b = 'Bar';
```
  在这段代码片段中，$b被创建为$a的引用，如果它已经被设置： $b = &$GLOBALS['a']; 或者因为在全局的其他位置使用$b = &$a;
  
  当最后一行到达，$a和$b都查找实际的值，包含值"Bar". 让我们看看先通的函数，使用内部实现:
```
#if (PHP_MAJOR_VERSION > 5) || (PHP_MAJOR_VERSION == 5 && \
  PHP_MINOR_VERSION > 0)
PHP_FUNCTION(sample_reference_a)
{
  zval **a_ptr, *a;
  
  if(zend_hash_find(&EG(symbol_table), "a", sizeof("a"),
    (void**)&a_ptr) == SUCCESS) {
    a = *a_ptr;
  } else {
    ALLOC_INIT_ZVAL(a);
    zend_hash_add(&EG(symbol_table), "a", sizeof("a"), &a, sizeof(zval*), NULL);
  }
  
  zval_ptr_dtor(return_value_ptr);
  if(!a->is_ref && a->refcount > 1) {
    /** $a是写时复制引用设置， 必须分离然后使用 **/
    zval *newa;
    MAKE_STD_ZVAL(newa);
    *newa = *a;
    zval_copy_ctor(newa);
    newa->is_ref = 0;
    newa->refcount = 1;
    zend_hash_update(&EG(symbol_table), "a", sizeof("a"), &newa,
      sizeof(zval*), NULL);
      
    a= newa;
  }
  
  a->is_ref = 1;
  a->refcount++;
  *return_value_ptr = a;
}
#endif
```
  return_value_ptr时另外一个传入所有内部函数的常用参数，zval**包含对return_value的指针。 通过调用zval_ptr_dtor()， 默认的retrun_value zval*被释放。 然后使用选择的新zval*替换它， 在这个例子中，变量$a，被促进到is_ref，并选择性的从任何可能具有的非完全引用对中分离出来。
  
  如果编译并运行这个代码，然而你可能得到一个段错误。为了让它能工作，需要添加一个结构到你的php_sample.h中:
```
#if (PHP_MAJOR_VERSION > 5) || (PHP_MAJOR_VERSION == 5 && PHP_MINOR_VERSION > 0)
static
  ZEND_BEGIN_ARG_INFO_EX(php_sample_retref_arginfo, 0, 1, 0)
  ZEND_END_ARG_INFO()
#endif
```
  然后使用结构，当你声明函数到php_sample_functions：
```
#if (PHP_MAJOR_VERSION > 5) || (PHP_MAJOR_VERSION == 5 && PHP_MINOR_VERSION > 0)
  PHP_FE(sample_reference_a, php_sample_retref_arginfo)
#endif
```
  这个结构在后续章节会了解到更多， 提供了重要提示给Zend引擎函数调用例程。 在这个例子中，它告诉ZE return_value将需要被重写， 并应该产生return_value_ptr使用正确的地址。 没有这个提示， ZE将简单的在return_value_ptr里边存放NULL, 那么将使这种特殊的函数奔溃， 当它到达zval_ptr_dtor()的时候。
  
> 注意 这些代码片段都用#if包围，告诉编译器，这个函数仅被PHP5.1以上支持。没有这个条件指令，扩展在PHP4中无法编译的(因为几个元素，包括return_value_ptr不存在)， 在PHP5.0中功能也不正确(那里有一个bug，导致引用返回实际返回的是值拷贝。)
  
  
  注意以上部分代码是可以通过测试的，部分在PHP7中由于内核的升级，不再支持， 比如zval结构体的修改，return_value_used变量废弃之类的。
  
============================

#### 通过引用返回值
  使用return构造来实现在函数内传回值和变量引用还是挺好的，但是有时候你希望函数返回多个值。当然你可以使用数组来实现，后面第8章会介绍。 或者你可以使用传入参数栈来实现回传值。
  
##### 通过引用传递参数调用
  一种简单的传递引用参数的方式是在调用作用域名中使用&符号表明按引用传参， 就像下面的代码一样:
```
function sample_byref_calltime($a)
{
  $a .= ' (modified by ref!)';
}

$foo = 'I am a string';
sample_byref_calltime(&$foo);
echo $foo;
```
  &符号放在参数中调用，导致$foo使用实际的zval, 而不是它内容的拷贝，被送到函数中。 这允许函数就地修改值， 然后有效的通过传入参数返回信息。如果sample_byref_calltime()没有使用&放置在$foo之前，函数内部的改变不会对原来的值有任何影响。
  
  用C来重复这种尝试，也没有什么特别的。 在源文件中创建下面的函数:
```
```

============================

#### 总结

============================
