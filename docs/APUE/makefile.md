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
  通常情况下，会使用下面的命令来编译:`gcc -o hellomake hellomake.c hellofunc.c -I.`.
  编译两个.c文件为可执行文件hellomake. -I参数告诉gcc在当前目录查找包含文件hellofunc.h。 没有Makefile， 那么我们一般都是经典的做法: 测试/修改/调试 循环。那么我们需要在命令行使用上下键来选择前面执行的编译命令。如果有更多的C文件加入进来， 简直就是噩梦开始。
  
  下面我们使用简单的Makefile来帮助你实现上面的编译。
```
// Makefile
hellomake: hellomake.c hellofunc.c
  gcc -o hellomake hellomake.c hellofunc.c -I.
```
  上面创建了makefile文件，那么我们就可以在命令行输入make hellomake命令， 然后回车就搞定了。当然也可以简单的输入make就搞定了，因为我们的makefile只有一个指令， 那么make不带参数，默认使用第一个指令。
  
  有了这个makefile, 就不用再使用上下键来找之前使用的编译命令了.
  
  下面对makefile稍作修改，如下：
```
CC=gcc
CFLAGS=-I.
hellomake: hellomake.o hellofunc.o
  $(CC) -o hellomake hellomake.o hellofunc.o $(CFLAGS)
```
  上面使用了makefile的宏定义CC和CFLAGS. 
  
```
CC=gcc
CFLAGS=-I.
DEPS = hellomake.h

%.o: %.c $(DEPS)
  $(CC) -c -o $@ $< $(CFLAGS)
  
hellomake: hellomake.o hellofunc.o
  $(CC) -o hellomake hellomake.o hellofunc.o $(FLAGS)
```
  增加宏DEPS, 这里是.c文件所依赖的.h文件列表。 然后定义一个规则，即生成.o后缀的对象文件。 这个规则是将.c的文件对应编译为.o的文件，并且包含进来.h的依赖头文件。-c参数标识产生.o文件。 -o $@表示将冒号前面的命名文件放在编译对象文件名的地方。 $<表示依赖文件列表的第一个项。
  
  下面例子再使用$@,$^分别代表冒号左右的项来实现makefile.
```
CC=gcc
CFLAGS=-I.
DEPS = hellomake.h
OBJ = hellomake.o hellofunc.o

%.o: %.c $(DEPS)
  $(CC) -c -o $@ $< $(CFLAGS)
  
hellomake: $(OBJ)
  $(CC) -o $@ $^ $(FLAGS)
```

  最后再考虑另外一种场景， 如果我们希望头文件.h都放在include目录下面， 我们的源代码都放在src目录， 其他本地类库放在lib目录下面。 
```
IDIR=../include
CC=gcc
CFLAGS=-I$(IDIR)

ODIR=obj
LDIR=../lib

LIBS=-lm

_DEPS = hellofunc.h
DEPS = $(PATSUBST %, $(IDIR)/%,$(_DEPS))

_OBJ = hellomake.o hellofunc.o
OBJ = $(patsubst %, $(ODIR)/%, $(_OBJ))

$(ODIR)/%.o: %.c $(DEPS)
  $(CC) -c -o $@ $< $(CFLAGS)
  
hellomake: $(OBJ)
  gcc -o $@ $^ $(CFLAGS) $(LIBS)
  
.PHONY: clean

clean:
  rm -f $(ODIR)/*.O *~ CORE $(INCDIR)/*~
```

## 自动变量(automatic variables)
  假设你需要写将.c文件编译为.o文件的模式规则: 如何写cc命令来保证在正确的源文件名上进行操作呢? 你不可能以食谱的形式列出这些名字，因为每次隐含的命令的名字都不同的。
  
  这时就需要借助make的特殊特性，自动变量来完成。 这种变量针对每次执行的规则进行计算而来， 基于规则的target和先决条件。上面的例子我们可以使用$@和$^分别代表要编译的.c文件和编译为的.o对象文件名。
  下面是自动变量的列表:
  * $@: 规则的target文件名。如果target为打包成员，那么$@是打包文件的名字。 
  * $%: 当target是打包成员的时候，该变量表示target成员名。如果target不是打包成员，那么$%为空。
  * $<: 第一个先决条件的名字
  * $?: 所有比target更新的先决条件的名字列表， 以空格分割。
  * $^: 所有先决条件的名字列表。
  * $+: 类似$^.
