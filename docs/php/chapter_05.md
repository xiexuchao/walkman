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
  
  

=========================================

#### 构建你的第一个扩展

=========================================

#### 静态构建

=========================================

#### 功能函数集

=========================================

#### 总结

=========================================
