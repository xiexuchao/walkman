# 进程学习笔记

## 进程与程序
  程序是完成特定任务的一系列指令集合。
  
  什么是进程呢?
  * 从用户角度来看进程是程序的依次动态执行过程。
  * 从操作系统的核心来看，进程是操作系统分配的内存、CPU时间片等资源的基本单位。
  * 进程是资源分配的最小单位。
  * 每一个进程都有自己独立的地址空间与执行状态。
  * 像UNIX这样的多任务操作系统能够让许多程序同时运行，每一个运行着的程序就构成了一个进程。

  进程的数据结构
  
  进程的静态描述: 由三部分组成PCB(进程控制块)、有关程序段和该程序段对其进行操作的数据结构集。
  * 进程控制块(PCB): 用于描述进程情况及控制进程运行所需的全部信息，是操作系统用来感知进程存在的一个重要数据结构。
  * 代码段: 是进程中能被进程调度程序在CPU上执行的程序代码段。
  * 数据段: 一个进程的数据段，可以是进程对应的程序加工处理的原始数据，也可以是程序执行后产生的中间或最终数据。
  
  进程 = 代码段(编译后形成的一些指令) + 数据段(程序运行时需要的数据) + 堆栈段(程序运行时动态分配的一些内存) + PCB(进程信息，状态标识等)

  数据段包括:
  * 只读数据段: 常量
  * 已初始化数据段: 全局变量，静态变量
  * 未初始化数据段(bss)(0初始化段): 未初始化的全局变量和静态变量(实际上不分配内存，因为都为0，只有一些标记信息)

  进程与程序的区别和联系
  * 进程是动态的，程序是静态的。
  * 进程的生命周期是相对短暂的，而程序是永远的。
  * 进程数据结构PCB
  * 一个进程只能对应一个程序，一个程序可以对应多个进程

## 进程生命周期与PCB(进程控制块)

### 一. 进程状态变迁
  进程的三种基本状态:
  * Ready(就绪状态): 当进程已分配到除CPU以外的所有必要的资源，只要获得处理机便可立即执行，这时的进程状态称为就绪状态。
  * Running(执行状态): 当进程获得处理机，其程序正在处理机上执行，此时的进程状态称为执行状态。
  * Blocked(阻塞状态): 正在执行的进程，由于等待某个事件发生而无法执行时，便放弃处理机而处于阻塞状态。引起进程阻塞的事件可以由多种，例如，等待I/O的完成、申请缓冲区不能满足、等待信件(信号)等。

```
+------+        +----------+              +--------+              +--------+
|      |        |          |              |        |              |        |
| 新建 |------> |   就绪   | <--时间片到--|  运行  | -----------> |  结束  |
|      |        |          | ----调度---->|        |              |        |
+------+        +----------+              +--------+              +--------+
_                    ^                         |
_                    |                         |
_                I/O |                         | 等待某事件
_                结束|                         | 如I/O请求
_                    |     +--------+          |
_                    |     |        |          |
_                    +-----|  阻塞  |<---------+
_                          |        |
_                          +--------+
```
  一个进程在运行期间，不断从一个状态转换到另一种状态，它可以多次处于就绪和执行状态，也可以多次处于阻塞状态。
  * 就绪->执行: 处于就绪状态的进程，当进程调度程序为之分配了处理机后，该进程便由就绪状态转变为执行状态。
  * 执行->就绪: 处于执行状态的进程在执行过程中， 因为分配给它的一个时间片已经用完或更高优先级的进程抢占而不得不让出处理机，于是进程从执行状态转变成就绪状态。
  * 执行->阻塞: 正在执行的进程因等待某种事件发生而无法继续执行时，便从执行状态变成阻塞状态。
  * 阻塞->就绪: 处于阻塞状态的进程，若其等待的事件已经发生，于是进程由阻塞状态变为就绪状态。
  * 运行->终止: 程序执行完毕，撤销而终止。
  
  
  以上时最经典也是最基本的三种进程状态，但现在的操作系统都根据需要重新涉及了一些新的状态。
  如linux:
  * 运行状态(TASK_RUNNING): 是运行态和就绪态的合并，表示进程正在运行或准备运行，Linux中使用TASK_RUNNING宏表示此状态。
  * 可中断睡眠状态(TASK_INTERRUPTIBLE): 进程正在睡眠(被阻塞)，等待资源的到来时唤醒，也可以通过其他进程信号或时钟中断唤醒，进入运行队列。Linux中使用TASK_INTERRUPTIBLE来表示此状态。
  * 不可中断睡眠状态(深度睡眠状态)TASK_UNINTERRUBPTIBLE: 和浅度睡眠基本类似，但有一点就是不可被其他进程信号或时钟中断唤醒。Linux使用TASK_UNINTERRUPTIBLE宏来表示此状态。
  * 暂停状态(TASK_STOPPED): 进程暂停执行接受某种处理。如正在接受调试的进程处于这种状态。
  * 僵死状态(TASK_ZOMBIE): 进程已经结束但未释放PCB

## 进程控制块(PCB)
  进程控制块包括:
  * 进程描述信息:
  <ul><li>进程标识符用于唯一标识一个进程(pid, ppid)</li></ul>
  * 进程控制信息:
  <ul>
    <li>进程当前状态</li>
    <li>进程优先级</li>
    <li>程序开始地址</li>
    <li>各种计时信息</li>
    <li>通信信息</li>
  </ul>
  * 资源信息:
  <ul>
    <li>占用内存大小及管理用数据结构指针</li>
    <li>交换区相关信息</li>
    <li>I/O设备号、缓冲、设备相关的数据结构</li>
    <li>文件系统相关指针</li>
  </ul>
  * 现场保护信息(cpu进行进程切换时):
  <ul>
    <li>寄存器</li>
    <li>PC</li>
    <li>程序状态字PSW</li>
    <li>栈指针</li>
  </ul>

  进程标识:PID
  每个进程都会分配到一个独一无二的数字编号，我们称之为进程标识，或者就叫做PID.是正整数，取值范围从2到32768.
  可以通过cat /proc/sys/kernel/pid_max查看系统支持多少进程
  
  当一个进程被启动，它会顺序挑选下一个未使用的编号数字作为自己的PID.
  数字1一般未特殊进程init保留的。 init进程实际上是用户进程，它是一个程序，在/sbin/init, linux启动的第一个进程。
  实际上linux中还存在0号进程(内核进程), 它是一个空闲进程，它进行空闲资源的统计及交换空间的换入换出，1(init)进程是从0号进程创建的。
  
### 三. 进程创建
  不同操作系统所提供的进程创建原语的名称和格式不尽相同，但执行创建进程原语后，操作系统所做的工作却大致相同， 都包括以下几点:
  * 给新创建的进程分配一个内部标识(pcb), 在内核中建立进程结构。
  * 复制父进程的环境。
  * 为进程分配资源，包括进程映像所需要的所有元素(程序、数据、用户栈等)
  * 复制父进程地址空间的内容到该进程地址空间中。
  * 置该进程的状态为就绪，插入就绪队列。

### 四. 进程撤销
  进程终止时操作系统做以下工作:
  * 关闭软中断: 因为进程即将终止而不再处理任何软中断信号
  * 回收资源: 释放进程分配的所有资源，如关闭所有已打开文件，释放进程相应的数据结构等。
  * 写记账信息: 将进程在运行过程中所产生的记账数据(其中包括进程运行时的各种统计信息)记录到一个全局记账文件中
  * 置该进程为僵死状态: 向副进程发送子进程死的软中断信号，将终止信息status送到指定的存储单元中。
  * 转进度调度:因为此时CPU被释放，需要由进程调度进行CPU再分配。

### 五. 终止进程的五种方法
  从main函数返回: 从return返回，执行完毕退出
  调用exit: c函数库，实际上也是调用系统调用_exit完成的，在任何一个函数调用exit函数都可使得进程撤销
  调用_exit: 系统调用
  调用abort: 调用abort()函数使得进程终止，实际上该函数是产生一个SIGABRT信号
  由信号终止: 发送一些信号如SIGINT等信号。
  

## 进程复制fork, 孤儿进程，僵尸进程

### 进程的复制(或产生)
  使用fork函数得到的子进程从父进程的继承了整个进程的地址空间，包括:进程上下文、进程堆栈、内存信息、打开的文件描述符、信号控制设置、进程优先级、进程组号、当前工作目录、根目录、资源限制、控制终端等。

  子进程与父进程的区别在于:
  1. 父进程设置的锁，子进程不继承(因为如果是排它锁，被继承的话，矛盾了)
  2. 各自的进程ID和父进程ID不同
  3. 子进程的未决告警被清除
  4. 子进程的未决信号集设置为空集
  
### fork系统调用
  包含头文件<sys/types.h>和<unistd.h>
  函数功能: 创建一个子进程
  函数原型: pid_t fork(void); // 一次调用两次返回值，是在各自的地址空间返回，意味着现在有两个基本一样的进程在执行。
  参数: 无参数
  返回值: 
  如果成功创建一个子进程，对于父进程来说返回子进程ID
  如果成功创建一个子进程，对于子进程来说返回值为0
  如果为-1表示创建失败
```
+-----------------+
| initial process |
+-----------------+
_        |
_        |
_        V
_  +------------+
_  |  fork()    |----------------------------------+
_  +------------+                                  |
_        |                                         |
_        |                                     Returns 0
_   Returns a new PID                              |
_        |                                         |
_        |                                         |
_        V                                         V
+----------------------+                  +---------------------+
|  Original process    |                  |        New          |
|  continues           |                  |      Process        | 
+----------------------+                  +---------------------+
```

  父进程调用fork()系统调用，然后陷入内核，进行进程复制，如果成功:
  1. 则对调用进程即父进程来说返回值为刚产生的子进程pid, 因为进程PCB没有子进程信息，父进程只能通过这样获得。
  2. 对子进程(刚产生的新进程)， 则返回0
  
  这时就有两个进程在接着向下执行。
  如果失败，则返回-1, 调用进程继续向下执行。

  注: fork英文意思:分支，fork系统调用复制产生的子进程与父进程(调用进程)基本一样: 代码段+数据段+堆栈段+PCB, 当前的运行环境基本一样，所以子进程在fork之后开始向下执行，而不会从头开始执行。
  
  示例代码:
```
  #include <stdio.h>
  #include <unistd.h>
  #include <stdlib.h>
  
  #define ERR_EXIT(m) \
    do \
    {\
      perror(m);\
      exit(EXIT_FAILURE);\
    }\
    while (0)\
    
  int main(void)
  {
    pid_t pid;
    printf("before calling fork, calling process pid = %d\n", getpid());
    
    pid = fork();
    if(pid == -1)
      ERR_EXIT("fork error");
      
    if(pid == 0) {
      printf("this is child process and child's pid = %d, parent's pid = %d\n", getpid(), getppid());
    }
    
    if(pid > 0) {
      //sleep(1);
      printf("this is parent process and pid = %d, child's pid = %d\n", getpid(), pid);
    }
    
    return 0;
  }
```
  上面的代码gcc process_fork.c -o process_fork, 运行./process_fork， 运行结果如下:
```
before calling fork, calling process pid = 23721
this is parent process and pid = 23721, child's pid = 23722
this is child process and child's pid = 23722, parent's pid = 1
```
  当没有给父进程加sleep时，由于父进程先执行完，子进程成了孤儿进程，系统将其托孤给了1(init)进程，所以ppid = 1.

  当加上sleep(1);时，子进程先执行完:
```
before calling fork, calling process pid = 23733
this is child process and child's pid = 23734, parent's pid = 23733
this is parent process and pid = 23733, child's pid = 23734
```
  这次可以正确看到想要的结果。
  
### 孤儿进程、僵尸进程
  fork系统调用之后，父子进程将交替执行，执行顺序不定。
  如果父进程先退出，子进程还没想退出那么子进程的父进程将变为init进程(托孤给了init进程)。(任何一个进程都必须有父进程)
  
  如果子进程先退出，父进程还没有退出，那么子进程必须等到父进程捕获到了子进程的退出状态才真正结束，否则这个时候子进程就成为僵尸进程(僵尸进程:只保留一些退出信息公父进程查询)
  
  示例:
```
  #include <stdio.h>
  #include <unistd.h>
  #include <stdlib.h>
  
  #define ERR_EXIT(m) \
    do \
    {\
      perror(m);\
      exit(EXIT_FAILURE);\
    }\
    while (0)\
    
  int main(void)
  {
    pid_t pid;
    printf("before calling fork, calling process pid = %d\n", getpid());
    
    pid = fork();
    if(pid == -1)
      ERR_EXIT("fork error");
      
    if(pid == 0) {
      printf("this is child process and child's pid = %d, parent's pid = %d\n", getpid(), getppid());
    }
    
    if(pid > 0) {
      sleep(100);
      printf("this is parent process and pid = %d, child's pid = %d\n", getpid(), pid);
    }
    
    return 0;
  }
```
  从上可看到，子进程先退出，但进程列表中国年还可以查看到子进程(ps -ef | grep process_fork), 僵尸进程，如果系统中存在过多的僵尸进程，将会使得新的进程不能产生。
  
### 写时复制
  linux系统为了提高系统性能和资源利用率，在fork出一个新进程时，系统并没有真正复制一个副本。
  如果多个进程要读取它们自己那部分资源的副本，那么复制是不必要的。
  每个进程只要保存一个指向这个资源的指针就可以了。
  如果一个进程要修改自己的那部分资源的副本，那么就会复制那份资源。这就是写时复制的含义。
  
  fork和vfork
  在fork还没有实现copy on write之前，Unix设计者很关心fork之后立即执行exec所造成的地址空间浪费，所以引入了vfork系统调用。
  vfork有个限制，子进程必须立刻执行_exit()或exec函数。
  即使fork实现了copy on write, 效率也没有vfork高，但是我们不推荐使用vfork，因为几乎每一个vfork的实现，都或多或少存在一定的问题。
  
```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

#define ERR_EXIT(m) \
    do\
    {\
        perror(m);\
        exit(EXIT_FAILURE);\
    }\
    while (0)\

int main(void)
{
    pid_t pid;
    int val = 1;
    printf("before calling fork, val = %d\n", val);

    pid = fork();
    if(pid == -1) 
        ERR_EXIT("fork error");

    if(pid == 0) {
        printf("child process, before change val, val = %d\n", val);
        val++;

        printf("this is child process and val = %d\n", val);

        _exit(0);
    }   

    if(pid > 0) {
        sleep(1);

        printf("this is parent process and val = %d\n", val);
    }   

    return 0;
}
```
  编译并运行：
```
bogon:test apple$ gcc fork1.c -o fork1
bogon:test apple$ ./fork1 
before calling fork, val = 1
child process, before change val, val = 1
this is child process and val = 2
this is parent process and val = 1
```
  可见父进程和子进程共享资源写时复制copy on write.
  
```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>

#define ERR_EXIT(m) \
    do\
    {\
        perror(m);\
        exit(EXIT_FAILURE);\
    }\
    while (0)\

int main(void)
{
    pid_t pid;
    int fd; 
    fd = open("test.txt", O_WRONLY);

    if(fd == -1) 
        ERR_EXIT("Open Error");

    pid = fork();
    if(pid == -1) 
        ERR_EXIT("fork error");

    if(pid == 0) {
        write(fd, "child", 5); 
    }   

    if(pid > 0) {
        //sleep(1);
        write(fd, "parent", 6); 
    }   

    return 0;
}
```
  编译并运行
```
bogon:test apple$ gcc fork2.c -o fork2
bogon:test apple$ ./fork2
bogon:test apple$ cat test.txt
parentchildbogon:test apple$ 
```
  fork产生的子进程与父进程相同的文件描述符指向相同的文件表，引用计数器增加，共享文件偏移指针。
  
  从运行结果可以得知，父子进程共享文件偏移指针，父进程写完后文件偏移到parent后子进程开始接着写。
  
## 进程退出exit, _exit区别及atexit函数

### 一. 进程终止有5种方式:
  正常退出:
  * 从main函数返回
  * 调用exit
  * 调用_exit
  
  异常退出
  * 调用abort
  * 由信号终止
  
### 二. exit和_exit区别:
```
+-------------------------------------------+
|              进程运行                     |
+-------------------------------------------+
_         |                      |
_         |                      V
_         |       +------------------------------+
_         |       |      调用终止处理程序        |------+
_         |       +------------------------------+      |
_         |                      |                      |
_         |                      |                      |exit
_         |                      V                      |
_         _exit         +---------------------+         |
_         |             |    清除I/O缓冲      |---------+
_         |             +---------------------+
_         |                      |
_         V                      V
+-------------------------------------------+
|                 内核                      |
+-------------------------------------------+
_                     |
_                     |
_                     |
_                     V
+-------------------------------------------+
|             进程终止运行                  |
+-------------------------------------------+
```
  关于_exit():
  #include <unistd.h>
  void _exit(int status);
  #include <stdlib.h>
  void _Exit(int status);
  
  #include <stdlib.h>
  void exit(int status);
  
  exit()在stdlib.h中，而_exit()在unistd.h中。
  注: exit()就是退出，传入的参数时程序退出时的状态码，0表示正常退出，其他表示非正常退出，一般都用-1或1，标准C里边有EXIT_SUCCESS, EXIT_FAILURE两个宏，用exit(EXIT_SUCCESS);
  
  _exit()函数的作用最为简单: 直接使进程停止运行，清除其使用的内存空间，并销毁其在内核中的各种数据结构；exit()函数则在这些基础上做了一些包装，在执行退出之前加了若干道工序。
  exit()函数与_exit()函数最大的区别就在于exit()函数在调用exit系统调用之前要检查文件的打开情况，把文件缓冲区中的内容写回文件，就是清理"I/O缓冲".
  
  exit()在结束调用它的进程之前，要进行如下步骤:
  1. 调用atexit()注册的函数(出口函数), 按atexit注册时相反的顺序调用所有由它注册的函数，这使得我们可以指定在程序终止时执行自己的清理动作，例如，保存程序状态信息于某个文件，解开对共享数据库上的锁等。
  2. cleanup(): 关闭所有打开的流，这将导致写所有被缓冲的输出，删除用tmpfile函数建立的所有临时文件。
  3. 最后调用_exit()函数终止进程。
  
  _exit()做三件事情(main):
  1. 所有属于该进程的打开的文件描述符被关闭。
  2. 所有该进程的子进程都托管给init 1进程
  3. 给该进程的父进程发送一个SIGCHLD信号。
  
  exit执行完清理工作后就调用_exit来终止进程。

### 三. atexit()
  atexit可以注册终止处理程序，ANSI C规定最多可以注册32个终止处理程序。
  终止处理程序的调用与注册次序相反
  #include <stdlib.h>
  int atexit(void (* function)(void));
  
```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

void fun1()
{
    printf("fun1 is called\n");
}

void fun2()
{
    printf("fun2 is called\n");
}

int main(void)
{
    printf("main function\n");
    atexit(fun1);
    atexit(fun2);

    exit(EXIT_SUCCESS);
}
```
  运行结果如下:
```
bogon:process apple$ ./atexit 
main function
fun2 is called
fun1 is called
```
  
  当调用fork时，子进程继承父进程注册的atexit:
```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

#define ERR_EXIT(m) \
    do\
    {\
        perror(m);\
        exit(EXIT_FAILURE);\
    }\
    while (0)\


void fun1()
{
    printf("fun1 is called\n");
}

void fun2()
{
    printf("fun2 is called\n");
}

int main(void)
{
    pid_t pid;
    pid = fork();
    atexit(fun1);
    atexit(fun2);

    if(pid == -1) 
        ERR_EXIT("fork error");

    if(pid == 0) {
        printf("this is child process\n");
    }   

    if(pid > 0) {
        printf("this is parent process\n");
    }   

    return 0;
}
```
  编译并运行，结果如下:
```
bogon:process apple$ gcc atexit_fork.c -o atexit_fork
bogon:process apple$ 
bogon:process apple$ ./atexit_fork 
this is parent process
fun2 is called
fun1 is called
this is child process
fun2 is called
fun1 is called
```
  当atexit注册的函数中有一个没有正常返回或被kill, 则后续的注册函数都不会被执行
```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <signal.h>

void fun1()
{
    printf("fun1 is called\n");
}

void fun2()
{
    printf("fun2 is called\n");
    kill(getpid(), SIGINT);
}

void fun3()
{
    printf("fun3 is called\n");
}


int main(void)
{
    printf("main function\n");

    if(signal(SIGINT, SIG_DFL) == SIG_ERR) {
        perror("signal error");
        exit(EXIT_FAILURE);
    }   

    atexit(fun1);
    atexit(fun2);
    atexit(fun3);

    exit(EXIT_SUCCESS);
}
```
  编译并运行，结果如下:
```
bogon:process apple$ ./atexit_kill 
main function
fun3 is called
fun2 is called
```
  fun1没有被执行，因为fun2中kill了。
  
# 参考链接
  * [linux系统编程之进程](http://www.cnblogs.com/mickole/p/3187409.html)
