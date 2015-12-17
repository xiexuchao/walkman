## 第15章 进程间通信

### 15.1 引言
  第8章说明了进程控制原语，并且观察了如何调用多个进程。但是这些进程之间交换信息的唯一途径就是传送打开的文件，可以经由fork或exec来传递，也可以通过文件系统来传送。本章将说明进程之间相互通信的其他技术--进程间通信(InterProcess Communication, IPC).
  
  过去，Unix系统IPC是各种进程通信方式的统称，但是这些通信方式中极少有能在所有Unix系统实现中进行移植的。随着POSIX和The Open Group(以前是X/Open)标准化的推进和影响的扩大，情况已经得到改善，但差别仍然存在。下图是本书讨论的4中实现锁支持的不同形式的IPC.
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/unix_ipc_summary.png)
  
  

### 15.2 管道
  管道是Unix系统IPC的最古老形式，所有Unix系统都提供此种通信机制。管道有以下两种局限性.
  1. 历史上，它们是半双工的(即数据只能在一个方向上流动)。现在，某些系统提供全双工管道，但是为了最佳的可移植性，我们绝不应预先假定系统支持全双工管道。
  2. 管道只能在具有公共祖先的两个进程之间使用。通常，一个管道由一个进程创建，在进程调用fork之后，这个管道就能在父进程和子进程之间使用了。
  
  我们将看到FIFO没有第二种局限性，Unix域套接字没有这两种限制性。

  尽管有这两种限制，半双工管道仍是最常用的IPC形式。每当在管道中键入一个命令序列，让shell执行时，shell都会为每条命令单独创建一个进程，然后用管道将前一条命令进程的标准输出与后一条命令的标准输入相连接。
  
  管道是通过调用pipe函数创建的:
```
#include <unistd.h>
int pipe(int fd[2]);
```
  经由参数fd返回两个文件描述符: fd[0]为读打开，fd[1]为写打开。fd[1]的输出是fd[0]的输入. 
  
> 最初在4.3BSD和4.4BSD中，管道是用Unix域套接字实现的。虽然Unix域套接字默认是全双工的，但这些操作系统阻碍了用于管道的套接字，以至于这些管道只能以半双工模式操作。
> POSIX.1允许支持全双工管道。对于这些实现，fd[0],fd[1]以读写方式打开。
  
  图15-2给出了两种描绘半双工管道的方法。作图显示管道的两端在一个进程中相互连接， 右图则强调数据需要通过内核在管道中流动。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/half-duplex_pipe.png)
  
  fstat函数对管道的每一段都返回一个FIFO类型的文件描述符。可以用S_ISFIFO宏来测试管道。
  
> POSIX.1规定stat结构的st_size成员对于管道是未定义的。但是当fstat函数应用于管道读端文件描述符时，很多系统在st_size中存储管道中可用于读的字节数。但是，这时不可移植的。

  单个进程中的管道几乎没有任何用处。通常进程会先调用pipe, 接着调用fork, 从而创建从父进程到子进程的IPC通道，反之亦然。下图显示了这种情况:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/half-duplex_pipe_after_fork.png)
  
  fork之后做什么取决于我们想要的数据流的方向。对于从父进程到子进程的管道，父进程关闭管道的读端(fd[0]), 子进程关闭写端(fd[1]). 图15-4显示了在此之后描述符的状态结果:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/pipe_between_parent_child_process.png)
  
  对于一个从子进程到父进程的管道，父进程关闭fd[1], 子进程关闭fd[0].
  
  当管道的一段被关闭，下列两条规则起作用:
  1. 当读(read)一个写端已经关闭的管道时，在所有数据都被读取后，read返回0，表示文件结束。(从技术上讲，如果管道的写端还有进程，就不会产生文件的结束。可以复制一个管道的描述符，使得有多个进程对它具有写打开文件描述符。但是，通常一个管道只有一个读进程和一个写进程。下一节介绍FIFO时，会看到对于单个的FIFO常常有多个写进程。)
  2. 如果写(write)一个读端已经被关闭的管道，则产生信号SIGPIPE. 如果忽略该信号或者捕捉该信号并从处理程序返回，则write返回-1, errno设置为EPIPE.
  
  在写管道(或FIFO)时，常量PIPE_BUF规定了内核的管道缓冲区大小。如果对管道调用write, 而且要求写的字节数小于等于PIPE_BUF，则此操作不会与其他进程对同一管道(或FIFO)的write进行交叉进行. 但是，若多个进程同时写一个管道(或FIFO),而且我们要求写的字节数超过PIPE_BUF，那么我们所写的数据可能会与其他进程所写的数据相互交叉。用pathconf或fpatchconf函数可以确定PIPE_BUF的值。

  实例: 创建一个从父进程到子进程的管道，并且父进程经由该管道相子进程传送数据。
```
#include "apue.h"

int
main(void)
{
    int n;
    int fd[2];
    pid_t pid;
    char line[MAXLINE];

    if(pipe(fd) < 0)
        err_sys("pipe error");

    if((pid = fork()) < 0) {
        err_sys("fork error");
    } else if(pid > 0) { /** parent **/
        close(fd[0]); /** close parent read side of pipe **/
        write(fd[1], "hello world\n", 12);
    } else { /** child **/
        close(fd[1]); /** close child write side of pipe **/
        n = read(fd[0], line, MAXLINE);
        write(STDOUT_FILENO, line, n); 
    }   

    exit(0);
}
```
  注意，这里的管道方向和图15-4中是一致的。
  
  在上面的例子中，直接对管道描述符调用了read和write。更有趣的是将管道描述符复制到标准输入或标准输出上。通常，子进程会在执行此程序之后另一程序，该程序或者从标准输入(已创建的管道)读数据，或者将数据写至其标准输出(该管道)。
  
  实例: 试着写一个程序，其功能是每次一页的显示已产生的输出。已经有很多unix系统公共例程具有分业功能，因此无序再构建一个新的分页程序，只要调用用户最喜爱的分页程序就可以了。 为了避免先将所有数据写到一个临时文件中，然后再调用系统中有关程序显示该文件，我们希望通过管道将输出直接送到分页程序。为此，先创建一个管道，fork一个子进程，使子进程的标准输入称为管道的读端，然后调用exec，执行用的分页程序。下面程序实现了如何实现这种操作。(本例中要求再命令行有一个参数指定要显示的文件名称。通常，这种类型的程序要求在终端上显示的数据已经再存储器中了。)
```
#include "apue.h"
#include <sys/wait.h>

#define DEF_PAGER "/usr/bin/more" /** default pager program **/

int
main(int argc, char *argv[])
{
    int         n;  
    int         fd[2];
    pid_t       pid;
    char        *pager, *argv0;
    char        line[MAXLINE];
    FILE        *fp;

    if(argc != 2)
        err_quit("Usage: %s <pathname>", argv[0]);

    if((fp = fopen(argv[1], "r")) == NULL)
        err_sys("can't open %s", argv[1]);

    if(pipe(fd) < 0)
        err_sys("pipe error");

    if((pid = fork()) < 0) {
        err_sys("fork error");
    } else if(pid > 0) { /** parent **/
        close(fd[0]); /** close parent read end **/

        /** parent copies argv[1] to pipe **/
        while(fgets(line, MAXLINE, fp) != NULL) {
            n = strlen(line);
            if(write(fd[1], line, n) != n) {
                err_sys("write error to pipe");
            }   
        }   

        if(ferror(fp))
            err_sys("fgets error");

        close(fd[1]); /** close write end of pipe for reader **/

        if(waitpid(pid, NULL, 0) < 0)
            err_sys("waitpid error");
        exit(0);
    } else { /** child **/
        close(fd[1]); /** close child write end **/

        /** dup2 fd[0] to stdin **/
        if(fd[0] != STDIN_FILENO) {
            if(dup2(fd[0], STDIN_FILENO) != STDIN_FILENO)
                err_sys("dup2 error to stdin");
            close(fd[0]); /** don't need read this after dup2; **/
        }

        /** get arguments for execl() **/
        if((pager = getenv("PAGER")) == NULL)
            pager = DEF_PAGER;

        if((argv0 = strrchr(pager, '/')) != NULL)
            argv0++; /** step past rightmost slash **/
        else
            argv0 = pager; /** no slash in pager **/

        if(execl(pager, argv0, (char *)0) < 0)
            err_sys("execl error for %s", pager);
    }

    exit(0);
}
```
  在调用fork之前，先创建一个管道。调用fork，父进程关闭其读端，子进程关闭其写端。然后子进程调用dup2，使其标准输入成为管道的读端。当执行分页程序时，其标准输入将是管道的读端。
  
  将一个描述符复制到另一个上(在子进程中，fd[0]被复制到标准输入)，在复制之前应当比较该描述符的值是否已经具有希望的值。如果该描述符已经具有希望的值，并且调用了dup2和，那么该描述符的副本将被关闭。在本程序中，如果shell没有打开标准输入，那么程序开始处的fopen应已使用描述符0，也就是最小未使用的描述符，所以fd[0]绝不会等于标准输入。尽管如此，无论何时调用dup2和close将一个描述符复制到另一个上，作为一种保护性的编程措施，都要先将两个描述符进行比较。
  
  请注意，我们是如何尝试使用环境变量PAGER获得用户分页程序名称的。如果这种操作没有成功，即使用系统默认值。这是环境变量的常见用法。
  
  实例: 回忆一下8.9节中的函数:TELL_WAIT、TELL_PARENT、TELL_CHILD、WAIT_PARENT和WAIT_CHILD。前面使用信号实现的，下面我们提供一个使用管道的实现。
```
#include "apue.h"

static int pfd1[2], pfd2[2];

/**
 * 在fork之前调用，先创建两个管道，用于父子进程之间的通信所用
 */
void
TELL_WAIT(void)
{
    if(pipe(pfd1) || pipe(pfd2))
        err_sys("pipe error");
}


/**
 * 父进程使用管道pfd1向子进程发送通知 pfd1[1] --> write p to --> pfd1[0];
 * 子进程使用管道pfd2向父进程发送通知 pfd2[1] --> write c to --> pfd2[0];
 */

/**
 * 子进程通知父进程，通过向pfd2[1]写入c字符
 */
void
TELL_PARENT(pid_t pid)
{
    if(write(pfd2[1], "c", 1) != 1)
        err_sys("write error");
}

/**
 * 等待父进程，通过read阻塞进程，直到读取到输入，并且判断是否为字符p
 */
void
WAIT_PARENT(void)
{
    char c;
    if(read(pfd1[0], &c, 1) != 1) /** 一直阻塞直到pfd1[0]有数据就绪 **/
        err_sys("read error");

    if(c != 'p')
        err_quit("WAIT_PARENT: incorrect data");
}
void
TELL_CHILD(pid_t pid)
{
    if(write(pfd1[1], "p", 1) != 1)
        err_sys("write error");
}

void
WAIT_CHILD(void)
{
    char c;
    if(read(pfd2[0], &c, 1) != 1)
        err_sys("read error");

    if(c != 'c')
        err_quit("WAIT_CHILD: incorrect data");
}
```
  在调用fork之前创建了两个管道。父进程在调用TELL_CHILD时，经由上一个管道写一个字符p，子进程在调用TELL_PARENT是，经由下面的管道写一个字符c. 相应的WAIT_xxx函数调用read读一个字符，没有都到字符时阻塞(休眠等待)。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/tellwait_with_two_pipes.png)
  
  注意，每一个管道都有一个额外的读取进程，这没有关系。也就是说，除了子进程从pfd[0]读取，父进程上也有一个管道的读。因为父进程没有执行对该管道的读操作，所以这不会影响我们。
  
  
### 15.3 popen和pclose函数

### 15.4 协调

### 15.5 FIFO

### 15.6 XSI IPC

### 15.7 标识符和key

### 15.8 权限结构

### 15.9 配置限制

### 15.10 优点缺点

### 15.11 内存队列

### 15.12 信号灯

### 15.13 共享内存

### 15.14 POSIX信号灯

### 15.15 Client-Server属性

### 15.16 总结
