#Makefile学习及C项目编译
  Makefile是组织代码编译的简单方式。下面我们来学习下这个小工具。
  
## 先上一个简单的例子
  假设我们的项目有三个文件, hellomake.c, hellofunc.c, hellofunc.h。 入口程序为hellemake.c，使用的函数定义在hellofunc.c中， 而hellofunc.h是函数头文件。代码分别如下：

```
// hellomake.c
#include "hellomake.h"

int main(void)
{
  hello_world();
  
  return (0);
}

// hellofunc.c
#include <stdio.h>
void hello_world()
{
  printf("Hello World!");
  return ;
}

// hellofunc.h
void hello_world(void);
```
  通常情况下，会使用下面的命令来编译:`gcc -o hellomake hellomake.c hellofunc.c -I`.
  编译两个.c文件为可执行文件hellomake. -I参数告诉gcc在当前目录查找包含文件hellofunc.h。 没有Makefile， 那么我们一般都是经典的做法: 测试/修改/调试 循环。那么我们需要在命令行使用上下键来选择前面执行的编译命令。如果有更多的C文件加入进来， 简直就是噩梦开始。
  
  下面我们使用简单的Makefile来帮助你实现上面的编译。
```
// Makefile
hellomake: hellomake.c hellofunc.c
  gcc -o hellomake hellomake.c hellofunc.c -I
```
  上面创建了makefile文件，那么我们就可以在命令行输入make hellomake命令， 然后回车就搞定了。当然也可以简单的输入make就搞定了，因为我们的makefile只有一个指令， 那么make不带参数，默认使用第一个指令。
  
  有了这个makefile, 就不用再使用上下键来找之前使用的编译命令了.
  
  下面对makefile稍作修改，如下：
```
CC=gcc
CFLAGS=-I.
hellomake: hellomake.c hellofunc.c
  $(CC) -o hellomake hellomake.c hellofunc.c $(CFLAGS)
```
  上面使用了makefile的宏定义。
  
