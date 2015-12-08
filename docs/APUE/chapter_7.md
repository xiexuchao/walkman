## 第七章 进程环境

### 7.1 引言
  下一章将介绍进程控制原语，在此之前需要先了解进程的环境。本章中将学习:当执行程序时，其main函数是如何被调用的，命令行参数是如何传送给执行程序的；典型的存储器布局是什么样式；如何分配另外的存储空间；进程如何使用环境变量；进程终止的不同方式等等。
  另外还将说明longjmp和setjmp函数以及它们与栈的交互作用。本章结束之前，还将查看进程的资源限制。

### 7.2 main函数
  C程序总是从main函数开始执行。main函数的原型是: int main(int argc, char **argv);
  其中argc是命令行参数的数目，argv是指向参数的各个指针所构成的数组。
  
  当内核启动C程序时(使用一个exec函数。)在调用main函数前先调用一个特殊的启动例程。可执行程序文件将此启动例程指定为程序的起始地址--这是由连接编辑程序设置的，而连接编辑程序则由C编译程序(通常是cc)调用。启动例程从内核获得命令行参数和环境变量值，然后为调用main函数做好安排。

### 7.3 进程终止
  有五种方式使进程终止:
  1. 正常终止
  <ol>
    <li>从main返回</li>
    <li>调用exit</li>
    <li>调用_exit</li>
  </ol>
  2. 异常终止
  <ol>
    <li>调用abort</li>
    <li>由一个信号终止</li>
  </ol>

  上节提及的启动例程是这样编写的，使得从main返回后立即调用exit()函数。如果将启动例程以C代码形式表示(实际上该例程常常用汇编语言编写)，则它调用main函数的形式可能是:`exit( main(argc, argv) );

#### 7.3.1 exit和_exit函数
#### 7.3.2 atexit函数
  这两节参见前面的文章，这里待补充。

### 7.4 命令行参数
  当执行一个程序时， 调用exec的进程可以将命令行参数传递给该新程序。这是Unix SHELL的一部分常规操作。在前几章很多实例，我们已经看到了这一点。
```
#include "apue.h"
int main(int argc, char **argv)
{
  int i;
  for(i = 0; i < argc; i++) {
    printf("argv[%d]: %s\n", i, argv[i]);
  }

  for(i = 0; argv[i] != NULL; i++) {
    printf("argv[%d]: %s\n", i, argv[i]);
  }
}
```
  ANSI C和POSIX.1都要求argv[argc]是一个空指针。这就意味着，后面的循环处理`for(i = 0; argv[i] != NULL; i++)`是ok的。

### 7.5 环境表
  每个程序都接收到一张环境表。与参数表一样，环境表也是一个字符指针数组，其中每个指针包含一个以null结束的字符串地址。全局变量environ则包含了该指针数组的地址。
  `extern char **environ;`
  例如，如果该环境变量包含五个字符串，那么它看起来可能如图7-2所示。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/environ_demo.png)
  
  其中每个字符串的结束处都有一个null字符。我们称environ为环境指针，指针数组为环境表，其中各指针指向的字符串为环境字符串。
  
  按照惯例，环境由: name=value这样的字符串组成，这与图7-2中所示相同。大多数预定义名完全由大写字母组成，但这只是一个惯例。
  在历史上，大多数unix系统对main函数提供了第三个参数，他就是环境表地址:
  `int main(int argc, char **argv, char **envp);`
  因为ANSI C规定main只有两个参数，而且第三个参数与全局变量environ相比也没有带来更多益处，所以POSIX.1也规定应使用environ而不使用第三个参数。 通常用getenv和putenv函数来存取特定的环境变量，而不是用environ变量。但是，如果要查看整个环境，则必须使用environ指针。

### 7.6 C程序的内存空间布局

### 7.7 共享库

### 7.8 内存分配

### 7.9 环境变量

### 7.10 setjmp和longjmp函数

### 7.11 getrlimit和setrlimit函数

### 7.12 总结
