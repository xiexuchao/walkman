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
  
### 总结
  PHP扩展开发非常简单，总结如下:
  1. 使用ext_skel工具产生模块骨架目录结构
  2. 修改config.m4文件， 该文件是后面编译的重点，后面介绍。
  3. 修改模块源文件
  4. 编译安装模块
  5. 测试模块
  

  
