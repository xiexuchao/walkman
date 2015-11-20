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
  
## exec系列函数(execl, execlp, execle, execv, execvp)使用
### 一. exec替换进程映像
  在进程的创建上Unix采用了一个独特的方法，它将进程创建与加载一个新进程映像分离。这样的好处是有更多的余地对两种操作进行管理。
  
  当我们创建了一个进程之后，通常将子进程替换成新的进程映像，这样可以用exec系列函数来进行。当然，exec系列的函数也可以将当前进程替换掉。
  
  例如，在shell命令执行ps命令，实际上是shell进程调用fork复制一个新的子进程，再利用exec系统调用将新产生的子进程完全替换成ps进程。
  
### 二. exec系列函数(execl, execlp, execle, execv execvp)
  包含头文件<unistd.h>
  功能：
  用exec函数可以把当前进程替换为一个新进程，且新进程与原进程有相同的PID. exec名下是由多个关联函数组成的一个完整系列。
  头文件<unistd.h>
  extern char **environ;
  原型:
  int execl(const char *path, const char *arg, ...);
  int execlp(const char *file, const char *arg, ...);
  int execle(const char *path, const char *arg, ..., char * const envp[]);
  int execv(const char *path, char *const argv[]);
  int execvp(const char *file, char *const argv[]);
  
  参数:
  * path: 表示你要启动程序的名称包括路径名
  * arg: 表示启动程序所带的参数，一般第一个参数为要执行命令名，不是带路径且arg必须以NULL结束。
  
  返回值: 成功返回0，失败返回-1
  注: 上述exec系列函数底层都是通过execve系统调用实现:
  #include <unistd.h>
  int execve(const char *filename, char *const argv[], char *const envp[]);
  描述: execve执行由filename指定的程序。 filename必须为二进制可执行程序或以这种格式开头的脚本。

  以上exec系列函数的区别:
  1. 带l的exec函数: execl, execlp, execle，表示后面的参数以可变参数的形式给出，且都以一个空指针结束。
  2. 带p的exec函数: execlp, execvp,表示第一个参数path不用输入完整路径，只要给出命令名即可，它会在环境变量PATH中查找命令。
  3. 不带l的exec函数: execv, execvp表示命令所需的参数以char *arg[]的形式给出，且arg最后一个元素必须为NULL.
  4. 带e的exec函数: execle表示，将环境变量传递给需要替换的进程
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    printf("entering main process---\n");
    execl("/bin/ls", "ls", "-la", NULL);

    printf("exiting main process ----\n");
    return 0;
}
```
  编译并运行，结果如下:
```
bogon:process apple$ ./exec_test 
entering main process---
total 128
drwxr-xr-x  10 apple  staff   340 11 20 16:08 .
drwxr-xr-x   5 apple  staff   170 11 20 15:28 ..
-rwxr-xr-x   1 apple  staff  8584 11 20 15:31 atexit
-rw-r--r--   1 apple  staff   269 11 20 15:31 atexit.c
-rwxr-xr-x   1 apple  staff  8680 11 20 15:36 atexit_fork
-rw-r--r--   1 apple  staff   560 11 20 15:36 atexit_fork.c
-rwxr-xr-x   1 apple  staff  8800 11 20 15:43 atexit_kill
-rw-r--r--   1 apple  staff   498 11 20 15:43 atexit_kill.c
-rwxr-xr-x   1 apple  staff  8480 11 20 16:08 exec_test
-rw-r--r--   1 apple  staff   220 11 20 16:07 exec_test.c
```
  可见，利用execl将当前进程main替换掉，所有最后的那条打印语句不会输出。
  
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    printf("entering main process---\n");
    execlp("ls", "ls", "-la", NULL);

    printf("exiting main process ----\n");
    return 0;
}
```
  编译并执行， 结果和上个例子一样。区别在于execlp无需输入可执行程序的完整路径，execlp会自动在PATH中查找给定的命令名，并执行。
  
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    printf("entering main process---\n");
    int ret;
    char *argv[] = {"ls", "-la", NULL};
    execvp("ls", argv);

    if(ret == -1) {
        perror("execl error");
    }   

    printf("exiting main process ----\n");
}
```

  *e测试
  
```
//hello.c
#include <unistd.h>
#include <stdio.h>

extern char** environ;

int main(void)
{
    printf("hello pid=%d\n", getpid());

    int i;
    for(i = 0; environ[i] != NULL; ++i)
    {   
        printf("%s\n", environ[i]);
    }   

    return 0;
}
```

  先使用execl调用hello,打印环境变量， 默认的环境变量被打印出来。
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    printf("entering main process---\n");
    int ret;
    ret = execl("./hello", "hello", NULL);

    if(ret == -1) 
        perror("execl error");

    printf("exiting main process ----\n");
}
```
  编译并执行， 结果如下:
```
bogon:process apple$ ./exec_test3
entering main process---
hello pid=24479
PATH=/opt/local/bin:/opt/local/sbin:/opt/local/bin:/opt/local/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
PWD=/Users/apple/development/clang/process
LANG=zh_CN.UTF-8
XPC_FLAGS=0x0
XPC_SERVICE_NAME=0
HOME=/Users/apple
SHLVL=1
LOGNAME=apple
_=./exec_test3
OLDPWD=/Users/apple/development/clang
```

  然后使用execle函数自己给的需要传递的环境变量信息。
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    char *const envp[] = {"AA=11", "BB=22", NULL};
    printf("entering main process---\n");
    int ret;
    ret = execle("./hello", "hello", NULL, envp);

    if(ret == -1) 
        perror("execl error");

    printf("exiting main process ----\n");
    return 0;
}
```
  然后编译运行，结果如下:
```
bogon:process apple$ ./execle
entering main process---
hello pid=24516
AA=11
BB=22
```
  这个时候hello打印的仅仅是execle传入的环境变量，而非系统默认的环境变量了。
  
  
  总结: 
  * exec*带l: 表示命令参数以可变参数的形式传递给待调用命令，最后一个可变参数为NULL。 execl, execle, execlp
  * exec*带p: 可以不用提供待执行命令的完整路径，由系统PATH变量自动查找. execlp, execvp
  * exec*带e: 表示传入需要使用的环境变量，而非使用系统默认环境变量. execle
  * exec*不带l: 表示传递给命令的参数以数组形式传入，数组最后一个元素为NULL. execv, execvp
  
### 三. fcntl()函数中的FD_CLOEXEC标识在exec系列函数中的作用
  #include <unistd.h>
  #include <fcntl.h>

  int fcntl(int fd, int cmd, ... /* arg */);
  文件描述符标志
  随后的命令操作文件描述符相关的标志。 当前， 只定义了一个类似的标志:
  FD_CLOEXEC: close-on-exec标志。如果FD_CLOEXEC位是0，文件描述符在execve(2)的整个过程依然保持打开，否则关闭。
  
  F_GETFD(void): 读取文件描述符标志， arg被忽略
  F_SETFD(long): 设置文件描述符标志到由arg指定的值上
  
  如: fcntl(fd, FSETFD, FD_CLOEXEC);
  
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>

int main(void)
{
    printf("entering main process---\n");
    int ret = fcntl(1, F_SETFD, FD_CLOEXEC);
    if(ret == -1) {
        perror("fcntl error");
    }   

    int val;
    val = execlp("ls", "ls", "-la", NULL);

    if(val == -1) {
        perror("execl error");
    }   

    printf("exiting main process ----\n");
    return 0;
}
```
  编译并执行，结果如下:
```
bogon:process apple$ ./fcntl 
entering main process---
```
  fcntl(1, F_SETFD, FD_CLOEXEC); 将标准输出关闭， 调用execlp("ls", "ls", "-ls", NULL); 无法再将结果输出到标准输出了。
  
  
## 父进程查询子进程的退出, wait, waitpid
### 僵尸进程
  当一个子进程先于父进程结束运行时，它与父进程之间的关联还会保持到父进程也正常地结束运行，或者父进程调用了wait才告终。
  子进程退出时，内核将子进程置为僵尸状态，这个进程称为僵尸进程，它只保留最小地一些内核数据结构，一边父进程查询子进程地退出状态。
  
  进程表中代表子进程地数据项不会立刻释放的，虽然不再活跃了， 可子进程还停留再系统里， 因为它的退出码还需要保存起来以备父进程中后续的wait调用使用。 他将称为一个僵进程。
  
### 如何避免僵尸进程
  调用wait或者waitpid函数查询子进程退出状态， 此方法父进程会被挂起。
  
  如果不想让父进程挂起，可以再父进程中加入一条语句: signal(SIGCHLD, SIG_IGN);表示父进程忽略SIGCHLD信号，该信号是子进程退出的时候项父进程发送的。
  
### SIGCHLD信号
  当子进程退出的时候， 内核会向父进程发送SIGCHLD信号，子进程的退出是个异步事件(子进程可以再父进程运行的任何时刻终止)。
  如果不想让子进程变成僵尸进程可再父进程中加入:signal(SIGCHLD, SIG_IGN);
  如果将此信号的处理方式设为忽略，可以让内核把僵尸子进程交给init进程去处理，省去了大量僵尸进程占用系统资源。
  
```
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <stdlib.h>

int main(void)
{
    pid_t pid;

    if(signal(SIGCHLD, SIG_IGN) == SIG_ERR)
    {   
        perror("signal error");
        exit(EXIT_FAILURE);
    }   

    pid = fork();
    if(pid == -1) 
    {   
        perror("fork error");
        exit(EXIT_FAILURE);
    }   

    if(pid == 0) {
        printf("this is child process\n");
        exit(0);
    }   


    if(pid > 0) {
        sleep(10);
        printf("this is parent process\n");
    }   

    return 0;
}
```
  编译并运行， 可看到， 子进程退出了， 但查看进程， 看不到退出子进程为僵尸进程。
  
### wait()函数
  #include <sys/types.h>
  #include <sys/wait.h>
  
  pid_t wait(int *status);
  
  进程一旦调用了wait, 就立即阻塞自己，由于wait自动分析是否当前进程的某个子进程已经退出，如果让它找到了这样的一个已经变成僵尸的子进程，wait就会收集这个子进程的信息，并把它彻底销毁后返回；如果没有找到这样的一个子进程，wait会一直阻塞在这里，知道有一个出现为止。
  
  参数status用来保存被收集进程退出时的一些状态，它是一个指向int类型的指针。如果我们对这个子进程时如何死掉的毫不在意，指向把这个僵尸进程消灭掉，(事实上，绝大多数情况下，我们都会这样想)，我们就可以设定这个参数为NULL,就想下面这样。
  pid = wait(NULL);
  
  如果成功，wait会返回被收集的子进程的进程ID,如果调用进程没有子进程，调用就会失败，此时wait返回-1, 同时errno被设置为ECHILD。
  
  wait系统调用会使得父进程暂停执行，直到它的一个子进程结束为止。 返回的时子进程的PID,它通常时结束的子进程。
  状态信息允许父进程判定子进程的退出状态，即从子进程的main函数返回的值或子进程中exit语句的退出码。如果status不是一个空指针，状态信息将被写入它指向的位置。
  
  可以上述的一些宏判断子进程的退出情况的宏:
  * WIFEXITED(status): 如果子进程正常结束，该宏返回一个非零值
  * WEXITSTATUS(status): 如果WIFEXITED非零，该宏返回子进程退出码
  * WIFSIGNALED(status): 如果子进程因为捕获信号而终止，返回非零值
  * WTERMSIG(status): 如果WIFSIGNALED非零，返回信号代码
  * WIFSTOPPED(status): 如果子进程被暂停，返回一个非零值
  * WSTOPSIG(status): 如果WIFSTOPPED非零，返回一个信号代码

```
#include <stdio.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    pid_t pid;
    pid = fork();
    if(pid < 0) {
        perror("fork error");
        exit(EXIT_FAILURE);
    }   

    if(pid == 0) {
        printf("this is child process\n");
        exit(100);
    }   

    int status;
    pid_t ret;
    ret = wait(&status);

    if(ret < 0) {
        perror("wait error");
        exit(EXIT_FAILURE);
    }   

    printf("ret = %d pid = %d\n", ret, pid);

    if(WIFEXITED(status))
        printf("child exited normal exit status = %d\n", WEXITSTATUS(status));

    else if(WIFSIGNALED(status))
        printf("child exited abnormal signal number = %d\n", WTERMSIG(status));

    else if(WIFSTOPPED(status))
        printf("child stopped signal number = %d\n", WSTOPSIG(status));

    return 0;
}
```

  编译并运行，结果如下:
```
bogon:process apple$ ./wait1 
this is child process
ret = 24830 pid = 24830
child exited normal exit status = 100
```
  子进程正常退出， WIFEXITED(status)返回值大于0， 通过WEXITSTATUS(status)可以获取子进程退出码.
  然后将子进程的exit(100),替换为abort(), 那么执行结果如下:
```
bogon:process apple$ ./wait1 
this is child process
ret = 24849 pid = 24849
child exited abnormal signal number = 6
```
  子进程异常退出，WIFSIGNALED(status)为真， 可用WTERMSIG(status)获取信号码。
  
  
### waitpid()函数
  #include<sys/types.h>
  #include<sys/wait.h>
  
  pid_t waitpid(pid_t pid, int * status, int options);
  
# 参考链接
  * [linux系统编程之进程](http://www.cnblogs.com/mickole/p/3187409.html)
