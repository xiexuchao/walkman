# 第一章 Unix System Overview(Unix系统概述)
## 1.1 简介
  所有的操作系统都向它运行的程序提供服务。典型的服务包括执行新程序、打开文件、读取文件、分配内存区域、获取当前时间等等。 本书着重描述各种Unix操作系统所提供的服务。
  
  以严格的步进方式、不超前引用尚未说明过的术语的方式来说明UNIX几乎是不可能的(可能是令人厌恶的)。 本章从程序设计人员的角度快速浏览UNIX, 并对书中引用的一些术语和概念进行简要的说明并给出实例。在以后各章种，将对这些概念做更详细的说明。本章也对不熟悉Unix的程序设计人员简要介绍了Unix提供的各种服务。
  
## 1.2 Unix架构
  严格意义上说，操作系统可以被定义为控制计算机硬件资源的软件，并能为程序运行提供环境。一般而言，我们称这种软件叫做内核(kernel)， 因为它相对比较小、而且位于环境的核心位置。 下图展示了Unix系统架构:
  ![Unix架构图](https://github.com/walkerqiao/walkman/blob/master/images/APUE/unix_architecture.png)
  
  内核接口层就是所谓的系统调用软件层(system calls)(上图中的阴影部分). 公共功能库构建于系统调用接口层之上， 但是应用程序可以使用自由使用系统调用和公共类库。(我们会在后面的1.11节介绍更多这方面的。) Shell一种特殊的应用，提供了运行其他应用程序的接口。
  
  从广义上来讲，操作系统是由内核以及所有其他使得计算机有用并有个性的软件组成。这些其他应用程序包括系统工具、应用程序、shells、公共函数等等。
  
  例如，Linux是GNU操作系统使用的内核。 有些人称它为GNU/Linux操作系统， 但是更多普遍称为简单的Linux。 虽然这种叫法在严格意义上来说是不对的， 不标准的， 给术语操作系统双重意义。(但是这样优势是更加简洁。)

## 1.3 登录
  登录Unix系统时，先键入登录名，然后键入口令。系统在其口令文件，通常是/etc/passwd文件中查看登录名。口令文件中的登录项由7个以冒号分割的字段组成: 登录名、加密口令、数字用户ID、数字组ID、注视字段、起始目录、以及shell程序。
  
  很多比较新的系统已将加密口令移到另外一个文件中。第六章讲说明这种文件以及存取它们的函数。
  
  一旦登入，系统先显示一些典型的系统信息，然后就可以向shell程序键入命令。shell是一个命令行解释器，它读取用户输入，然后执行命令，用户通常用终端，优势则通过文件(shell脚本)向shell进行输入。 常用的shell有:
  * Bourne Shell, /bin/sh
  * C Shell, /bin/csh
  * KornShell, /bin/ksh
  
  系统从口令文件中登录项的最后一个字段中了解到应该执行哪一个shell.

  自V7以来，Bourne Shell得到广泛应用，几乎每一个现有的Unix系统都提供Bourne Shell. C Shell是在伯克利开发的，所有BSD版本都提供这种shell. 另外AT&T系统V/386 R3.2和SVR4也提供C Shell. KornShell是Bourne Shell的后继者，它由SVR4提供。 KornShell在大多数Unix系统上运行，但在SVR4之前，通常它需要另行购买，所以没有其他两种shell流行。

## 1.4 文件和目录

### 文件系统
  Unix文件系统是目录和文件的一种层次安排，目录的起点称为根(root)，其名字是一个字符'/'.
  
  目录(directory)是一个包含目录项的文件，在逻辑上，可以认为每个目录项都包含一个文件名，同时还包含说明该文件属性的信息。文件属性是：文件类型、文件长度、文件所有者、文件的许可权(例如，其他用户能否访问该文件)、文件的最后修改时间等。 stat和fstat函数返回一个包含所有文件属性的信息结构。
### 文件名
  目录中的各个名字称为文件名(filename).不能出现在文件名中的字符只有两个'/', 和空操作符(null).
### 路径名
  0个或多个以斜线分割的文件名序列(可以任选地以斜线开头)构成路径名(pathname), 以斜线开头的路径名称为绝对路径名(absolute pathname), 否则称为相对路径(relative pathname).
  
  实例，列出一个目录中所有文件的名字，下面程序是ls(1)命令的主要实现部分:
```
#include "apue.h"

int main(int argc, char **argv)
{
    DIR *dp;
    struct dirent *dirp;

    if(argc != 2)
        err_quit("a single argument (the directory name) is required");

    if((dp = opendir(argv[1])) == NULL)
        err_sys("can't open %s", argv[1]);

    while((dirp = readdir(dp)) != NULL)
        printf("%s\n", dirp->d_name);

    closedir(dp);
    exit(0);
}
```
  其中apue.h为公共头，包含了公共的库函数定义，以及必要的头文件引入。 这里的DIR, dirent都是需要引入`include <dirent.h>`. 
  
  ls(1)这种表示方法是Unix的惯用方法，用以引用Unix手册集中的一个特定项。它引用第一部分中的ls项。各部分通常用1-8表示，在每个部分中的各项则按字母顺序排列。假定你有一份所使用的Unix系统的手册。
  
```
  早期的Unix系统把8个部分都集中在一本手册中，现在的趋势是把这些部分分别安排在不同的手册中:有用户专用手册、程序员专用手册、
  系统管理员专用手册等。
  
  某些Unix系统把一个给定部分中的手册页又用一个大写字母进行分成若干小部分，例如， 
  AT&T(1990e)中的所有标准I/O函数都被指明在3S部分中，例如fopen(3S).
  
  某些Unix系统，例如以Xenix为基础的系统，不是采用数字将手册分成若干部分，而是用C表示命令(第一部分)，S表示服务(通常是第2、3部分)等等。
```

  如果你有联机手册，则可用下面的命令查看ls命令手册页: `man 1 ls`
  
  上面的程序只打印目录中各个文件的名字，不显示其他信息。
  
  

### 工作目录
  每个进程都有一个工作目录(working directory,有时称为当前工作目录). 所有相对路径名都从工作目录开始解释。进程可以用chdir函数更改其工作目录。
### 起始目录
  登录时，工作目录设置为起始目录(home directory),该起始目录从口令文件中的登录项中取得。

## 1.5 输入输出
### 文件描述符
  文件描述符是一个小的非负整数，内核用以标识一个特定进程正在存访的文件。当内核打开一个现存文件或创建一个新文件时，他就返回一个文件描述符。 当读、写文件时，就可使用它。
  
### 标准输入、标准输出和标准出错
  按惯例，每当运行一个新程序时，所有的shell都为其打开三个文件描述符:标准输入、标准输出以及标准出错。 如果像简单命令ls那样没有做什么特殊处理，则这三个描述副都连向终端。 大多数shell都提供一种方法，使任何一个或所有这三个描述符都能重新定向到某一个文件: `ls > file.list`
  
  执行ls命令，其标准输出重新定向到名为file.list的文件上。
  
### 不用缓存的I/O
  函数open, read, write, lseek以及close提供了不用缓存的I/O.这些函数都用文件描述符进行工作。
```
#include "apue.h"

#define BUFFSIZE 8129

int main(void)
{
    int n;
    char buf[BUFFSIZE];

    while((n = read(STDIN_FILENO, buf, BUFFSIZE)) > 0)
        if(write(STDOUT_FILENO, buf, n) != n)
            err_sys("write error");

    if(n < 0)
        err_sys("read error");

    exit(0);
}
```
  编译`gcc copy.c -o copy...`, 执行命令`ls -la . | ./copy > data`则ls-la的输出变为./copy的标准输入，继而被作为标准输出到data文件中。
  
  上面代码中read, write以及STDIN_FILENO, STDOUT_FILENO都是在unistd.h中定义。 这里赞不过多介绍。
```
这里穿插介绍下apue项目源代码结构：
apue +---------- lib
_    |            |------- error.c
_    |            |------- xx.c             ==> libapue.a
_    |            |------- xxlib.c
_    |            |------- makefile
_    |---------- include
_    |             |------ apue.h
_    |             |------ error.h
_    |---------- subproject
_    |             |----------  program1.c         => program1
_    |             |----------  makefile
_    |---------- subproject
_    |---------- makefile
_    |---------- make.defines.{osname}    // 包含特定os定义的make变量定义
_    |---------- make.libapue.inc         // 编译libapue的包含文件
_    |---------- systype.sh               // 获取os名称的脚本
```
  在apue根目录有一个makefile， 直接调用make可对所有项目进行编译，包括类库libapue.a。 而每个目录下面都有单独的makefile,用于编译每个子项目下面的所有程序。 这里每个子项目的程序列表都定义在makefile中。
  
  需要注意一点， 如果子项目修改后需要编译，同时需要确保libapue.a是最新的，从子项目里边无法确认libapue.a是否为最新。 makefile书写的问题吧。 暂时不做修改。
  
### 标准I/O
  标准I/O函数提供了一种对不用缓存的

## 1.6 程序和进程
## 1.7 错误处理
## 1.8 用户识别
## 1.9 信号
## 1.10 时间值
## 1.11 系统调用和类库
## 1.12 总结
