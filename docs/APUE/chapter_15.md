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
  常见的操作是创建一个连接到另一个进程的管道，然后读取其输出或向其输入端发送数据，为此，标准I/O库提供了两个函数popen和pclose。 这两个函数实现的操作是: 创建一个管道，fork一个子进程，关闭并未使用的管道端，执行一个shell运行命令，然后等待命令终止。
  
```
#include <stdio.h>
FILE *popen(const char *cmdstring, const char *type);

int pclose(FILE *fp);
```
  函数popen先执行fork, 然后调用exec执行cmdstring, 并且返回一个标准I/O文件指针。
  * 如果type是"r", 则文件指针连接到cmdstring的标准输出。
  * 如果type是"w", 则文件指针连接到cmdstring的标准输入。
  如下图:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/popen_r_dialog.png)
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/popen_w_dialog.png)
  
  有一种方法可以帮助我们记住popen的最后一个参数及其作用，这就是与fopen进行类比。如果type为"r", 则返回的文件指针是可读的，如果type为"w", 那么返回的文件指针是可写的。

  pclose函数关闭标准I/O流，等待命令终止，然后返回shell的终止状态。如果shell不能被执行，则pclose返回的终止状态与shell已执行exit(127)一样。
  
  cmdstring以Bourne shell以下方式执行: `sh -c cmdstring`
  
  这表示shell将扩展cmdstring中的任何特殊字符。例如，可以使用:
  fp = popen("ls *.c", "r");
  或者
  fp = popen("cmd 2>&1", "r");
  
  实例: 用popen实现前面的分页程序，代码如下:
```
#include "apue.h"

#include <sys/wait.h>

#define PAGER "${PAGER:-more}" /** environment variable, or default **/

int
main(int argc, char *argv[])
{
    char line[MAXLINE];
    FILE *fpin, *fpout;

    if(argc != 2)
        err_quit("Usage: %s <pathname>", argv[0]);

    if((fpin = fopen(argv[1], "r")) == NULL)
        err_sys("can't open %s", argv[1]);

    if((fpout = popen(PAGER, "w")) == NULL)
        err_sys("popen error");

    /** copy argv[1] to pager **/
    while(fgets(line, MAXLINE, fpin) != NULL) {
        if(fputs(line, fpout) == EOF)
            err_sys("fputs error to pipe");
    }   

    if(ferror(fpin))
        err_sys("fgets error");

    if(pclose(fpout) == -1) 
        err_sys("pclose error");

    exit(0);
}
```
  使用popen减少了需要编写的代码量。shell命令${PAGER:-more}的意思是，如果shell变量PAGER已经定义，且值非空，则使用其值，否则使用字符串more.
  
  下面我们自己编写popen和pclose的实现:
```
#include "apue.h"
#include <errno.h>
#include <fcntl.h>
#include <sys/wait.h>

/*
 * Pointer to array allocated at run-time.
 */
static pid_t    *childpid = NULL;

/*
 * From our open_max(), {Prog openmax}.
 */
static int        maxfd;

FILE *
popen(const char *cmdstring, const char *type)
{
    int        i;  
    int        pfd[2];
    pid_t    pid;
    FILE    *fp;

    /* only allow "r" or "w" */
    if ((type[0] != 'r' && type[0] != 'w') || type[1] != 0) {
        errno = EINVAL;
        return(NULL);
    }   

    if (childpid == NULL) {        /* first time through */
        /* allocate zeroed out array for child pids */
        maxfd = open_max();
        if ((childpid = calloc(maxfd, sizeof(pid_t))) == NULL)
            return(NULL);
    }   

    if (pipe(pfd) < 0)
        return(NULL);    /* errno set by pipe() */
    if (pfd[0] >= maxfd || pfd[1] >= maxfd) {
        close(pfd[0]);
        close(pfd[1]);
        errno = EMFILE;
        return(NULL);
    }
    if ((pid = fork()) < 0) {
        return(NULL);    /* errno set by fork() */
    } else if (pid == 0) {                            /* child */
        if (*type == 'r') {
            close(pfd[0]);
            if (pfd[1] != STDOUT_FILENO) {
                dup2(pfd[1], STDOUT_FILENO);
                close(pfd[1]);
            }
        } else {
            close(pfd[1]);
            if (pfd[0] != STDIN_FILENO) {
                dup2(pfd[0], STDIN_FILENO);
                close(pfd[0]);
            }
        }

        /* close all descriptors in childpid[] */
        for (i = 0; i < maxfd; i++)
            if (childpid[i] > 0)
                close(i);

        execl("/bin/sh", "sh", "-c", cmdstring, (char *)0);
        _exit(127);
    }

    /* parent continues... */
    if (*type == 'r') {
        close(pfd[1]);
        if ((fp = fdopen(pfd[0], type)) == NULL)
            return(NULL);
    } else {
        close(pfd[0]);
        if ((fp = fdopen(pfd[1], type)) == NULL)
            return(NULL);
    }

    childpid[fileno(fp)] = pid;    /* remember child pid for this fd */
    return(fp);
}

int
pclose(FILE *fp)
{
    int        fd, stat;
    pid_t    pid;

    if (childpid == NULL) {
        errno = EINVAL;
        return(-1);        /* popen() has never been called */
    }

    fd = fileno(fp);
    if (fd >= maxfd) {
        errno = EINVAL;
        return(-1);        /* invalid file descriptor */
    }
    if ((pid = childpid[fd]) == 0) {
        errno = EINVAL;
        return(-1);        /* fp wasn't opened by popen() */
    }

    childpid[fd] = 0;
    if (fclose(fp) == EOF)
        return(-1);

    while (waitpid(pid, &stat, 0) < 0)
        if (errno != EINTR)
            return(-1);    /* error other than EINTR from waitpid() */

    return(stat);    /* return child's termination status */
}
```
  虽然，popen的核心代码与本章中前面用过的代码类似，但是增加了很多需要处理的细节。
  首先每次调用popen时，应当记住所创建的子进程的进程ID,以及其描述符或FILE指针。 我们选择在数组childpid中保存子进程ID, 并用文件描述符作为其下标。于是，当以FILE指针作为参数调用pclose时，调用标准函数fileno得到文件描述符，然后取得子进程ID, 并用其作为waitpid的参数。因为一个进程可能调用popen多次，所以在动态分配childpid数组时(第一次调用popen时)，其数组长度应当是最大文件描述符数，于是该数组中可以存放与最大文件描述符相同的子进程ID数。
  
  注意，2-17中的open_max函数可以返回打开文件的最大数的近似值，如果这个值与系统不相关的花。注意不要使用那种其值大于(或等于)open_max返回值的管道文件描述符。对于popen, 如果open_max函数返回值恰巧非常小，那我们就会关闭管道文件描述符并将errno设置为EMFILE, 以此表明这里的很多文件描述符是打开的，最后返回-1. 对于pclose，如果对应于文件指针参数的描述符比所期望的大，则将errno设置为EINVAL，并返回-1.
  
  调用pipe和fork然后为popen函数中的每个进程复制合适的描述符，这个过程和我们在本章前面所做的类似。
  
  POSIX.1要求popen关闭那些以前调用popen打开的、现在仍然在子进程中打开着的I/O流。为此，在子进程中从头逐个检查childpid数组的各个元素，关闭仍旧打开着的描述符。
  
  若pclose的调用者已经为信号SIGCHLD设置了一个信号处理程序，则pclose中的waitpid调用将返回一个错误EINTR. 因为允许调用者捕捉到此信号(或者任何其他可能中断waitpid调用的信号)，所以当waitpid被一个捕捉到的信号中断时，我们只是再次调用waitpid.
  
  注意，如果应用程序调用waitpid，并且获得了popen创建的子进程的退出状态，那么我们会在应用程序调用pclose时调用waitpid，如果发现子进程已不再存在，将返回-1, 将errno设置为ECHILD. 这正是这种情况下POSIX.1所要求的。
  
> 如果一个信号中断了wait, pclose的一些早期版本会返回错误EINTR. pclose的一些早期版本在wait期间，会阻塞或忽略信号SIGINT, SIGQUIT和SIGHUP. 这是POSIX.1所不允许的。

  注意，popen决不应由设置用户ID或设置组ID程序调用。当它执行命令时，popen等同于: execl("/bin/sh", "sh", "-c", command, NULL);
  
  它在从调用者继承的环境中执行shell,并由shell解释执行command. 一个恶意用户可以操控这种环境，使得shell能以设置ID文件模式所授予的提升了的权限以及非预期的方式执行命令。
  
  popen非常适合执行简单的过滤器程序，它变换运行命令的输入或输出。当命令希望构造它自己的管道时，就是这种情况。
  
  实例:
  考虑一个应用程序，他向标准输出写一个提示，然后从标准输入读一行。使用popen，可以在应用程序和输入之间插入一个程序以便对输入进行变换处理。图15-13显示了这种情况下的进程安排。
  
  对输入进行变换可能是路径名扩充，或是提供一种历史机制(记住以前输入的命令)。
  
  下面的程序用于简单演示这个操作过滤程序。它将标准输入复制到标准输出，在复制时将大写字母转换成小写字母。在写完换行符之后，要仔细冲洗(fflush)标准输出，这样做的理由将在下一节介绍协同进程时讨论。
```
#include "apue.h"

#include <ctype.h>

int
main(void)
{
    int c;
    while((c = getchar()) != EOF) {
        if(isupper(c))
            c = tolower(c);

        if(putchar(c) == EOF)
            err_sys("output error");

        if(c == '\n')
            fflush(stdout);
    }   

    exit(0);
}
```
  这个过滤程序编译成可执行文件myuclc, 然后下面程序会用popen调用它。
```
#include "apue.h"                                                                      
#include <sys/wait.h>                                                                  
                                                                                       
int                                                                                    
main(void)                                                                             
{                                                                                      
    char line[MAXLINE];                                                                
    FILE *fpin;                                                                        
                                                                                       
    if((fpin = popen("./myuclc", "r")) == NULL)                                        
        err_sys("popen error");                                                        
                                                                                       
    for( ;; ) {                                                                        
        fputs("prompt> ", stdout);                                                     
        fflush(stdout);                                                                
                                                                                       
        if(fgets(line, MAXLINE, fpin) == NULL) /** read from pipe **/                  
            break;                                                                     
                                                                                       
        if(fputs(line, stdout) == EOF)                                                 
            err_sys("fputs error to pipe");                                            
    }                                                                                  
                                                                                       
    if(pclose(fpin) == -1)                                                             
        err_sys("pclose error");                                                       
                                                                                       
    putchar('\n');                                                                     
    exit(0);                                                                           
}
```
  因为标准输出通常是行缓冲的，而提示并不包含换行符，所以在写了提示后，需要调用fflush.

### 15.4 协同进程
  Unix系统过滤程序从标准输入读取数据，向标准输出写数据。几个过滤程序通常在shell管道中线性连接。当一个过滤程序既产生某个过滤程序的输入，又读取该过滤程序的输出时，它就变成了协同进行(coprocess).
  
  Korn shell提供了协同进程。 Bourne shell, Bourne-again shell和C shell并没有提供将进程连接成协同进程的方法。协同进程通常在shell后台运行，其标准输入和标准输出通过管道连接到另一程序。虽然初始化一个协同进程，并将其输入和输出连接到另一个进程的shell语法十分奇特，但是协同进程的工作方式在C程序中也是非常有用的。
  
  popen只提供连接到另一个进程的标准输入或标准输出的一个单向管道，而协同进程则有连接到另一个进程的两个单向管道: 一个连接到其标准输入，另一个则来自其标准输出。我们想将数据写到其标准输入，经其处理后，再从其标准输出读取数据。
  
  实例: 
  让我们通过一个实例来观察协同进程。进程创建两个管道:一个是协同进程的标准输入，另一个是协同进程的标准输出。下图展示了这种安排:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/coprocess_dialog.png)

  下面程序是一个简单的协同进程，它从标准输入读取两个数，计算它们的和，然后将和写至其标准输出。(协同进程通常会做较此有意义的工作。设计本实例的目的是为了帮助了解将进程连接起来所需的各种管道设施。)
```
#include "apue.h"

int
main(void)
{
    int     n, int1, int2;
    char    line[MAXLINE];

    while ((n = read(STDIN_FILENO, line, MAXLINE)) > 0) {
        line[n] = 0;        /* null terminate */
        if (sscanf(line, "%d%d", &int1, &int2) == 2) {
            sprintf(line, "%d\n", int1 + int2);
            n = strlen(line);
            if (write(STDOUT_FILENO, line, n) != n)
                err_sys("write error");
        } else {
            if (write(STDOUT_FILENO, "invalid args\n", 13) != 13) 
                err_sys("write error");
        }   
    }   
    exit(0);
}
```
  对此程序编译，将其可执行目标代码存入名add2的文件。
  
  然后下面的程序从其标准输入读取两个数之后调用add2协同进程，并将协同进程送来的值写到其标准输出。
```
#include "apue.h"

static void    sig_pipe(int);        /* our signal handler */

int
main(void)
{
    int      n, fd1[2], fd2[2];
    pid_t    pid;
    char    line[MAXLINE];

    if (signal(SIGPIPE, sig_pipe) == SIG_ERR)
        err_sys("signal error");

    if (pipe(fd1) < 0 || pipe(fd2) < 0)
        err_sys("pipe error");

    if ((pid = fork()) < 0) {
        err_sys("fork error");
    } else if (pid > 0) {                            /* parent */
        close(fd1[0]);
        close(fd2[1]);

        while (fgets(line, MAXLINE, stdin) != NULL) {
            n = strlen(line);
            if (write(fd1[1], line, n) != n)
                err_sys("write error to pipe");
            if ((n = read(fd2[0], line, MAXLINE)) < 0)
                err_sys("read error from pipe");
            if (n == 0) {
                err_msg("child closed pipe");
                break;
            }   
            line[n] = 0;    /* null terminate */
            if (fputs(line, stdout) == EOF)
                err_sys("fputs error");
        }   

        if (ferror(stdin))
            err_sys("fgets error on stdin");
        exit(0);
    } else {                                    /* child */
        close(fd1[1]);
        close(fd2[0]);
        if (fd1[0] != STDIN_FILENO) {
            if (dup2(fd1[0], STDIN_FILENO) != STDIN_FILENO)
                err_sys("dup2 error to stdin");
            close(fd1[0]);
        }   

        if (fd2[1] != STDOUT_FILENO) {
            if (dup2(fd2[1], STDOUT_FILENO) != STDOUT_FILENO)
                err_sys("dup2 error to stdout");
            close(fd2[1]);
        }
        if (execl("./add2", "add2", (char *)0) < 0)
            err_sys("execl error");
    }
    exit(0);
}

static void
sig_pipe(int signo)
{
    printf("SIGPIPE caught\n");
    exit(1);
}
```
  这个程序创建了两个管道，父进程、子进程各自关闭它们不需要使用的管道端。必须使用两个管道: 一个用于协作进程的标准输入，另一个则用作它的标准输出。然后，子进程调用dup2使管道描述符移至其标准输入和标准输出，最后调用了execl. 
  若编译上面的程序，他会按预期工作。此外，如果程序正在等待用户输入的时候杀死了add2协同进程，然后又输入两个数，那么程序对没有读进程的管道进行写操作时，会调用信号处理程序。
  
  实例: 在协同进程add2中，我们故意使用了底层I/O(Unix系统调用):read和write。 如果使用标准I/O来改写该协同程序，会怎么样呢?
  下面是改写的add2.c代码。
```
#include "apue.h"

int
main(void)
{
    int     int1, int2;
    char    line[MAXLINE];

/*    if(setvbuf(stdin, NULL, _IOLBF, 0) != 0)
        err_sys("setvbuf error");

    if(setvbuf(stdout, NULL, _IOLBF, 0) != 0)
        err_sys("setvbuf error");*/

    while (fgets(line, MAXLINE, stdin) != NULL) {
        if (sscanf(line, "%d%d", &int1, &int2) == 2) {
            if(printf("%d\n", int1 + int2) == EOF)
                err_sys("printf error");
        } else {
            if(printf("invalid args\n") == EOF)
                err_sys("printf error");
        }   
    }   
    exit(0);
}
```
  调用这个协同程序，则它不能工作。问题在默认的标准I/O缓冲机制上。当调用上面的程序时，对标准输入的第一个fgets引起标准I/O库分配一个缓冲区，并选择缓冲的类型。因为标准输入是一个管道，所以标准I/O是全缓冲的。标准输出也是如此。当add2从其标准输入读取而发生阻塞时，调用这个协作程序的程序从管道读时也发生阻塞，于是产生了死锁。
  
  这里，可以对将要运行的这一协同进程加以控制。 开启while循环上面的注释代码块即可。这些代码使得: 当有一行可用时，fgets就返回(设置stdin和stdout为行级缓冲)；当输出一个换行符时，printf立即执行fflush。对setvbuf进行的这些显示调用使得add2协同进程就可以正常工作了。
  
  如果不能修改协同程序的目标程序，则需要使用其他技术。例如，如果在程序中使用awk(1)作为协同进程(代替add2程序)，则下列命令行不能工作:
```
#!/bin/awk -f
{ print $1 + $2 }
```
  不能工作的原因还是标准I/O的缓冲机制问题。但是在这种情况下，无法改变awk的工作方式(除非有awk源代码)。我们不能修改awk的可执行代码，于是也就不能更改处理其标准I/O缓冲的方式。
  
  对这种问题的一般解决方法是使被调用的协同进程认为它的标准输入和输出都被连接到了一个终端。这使得协同进程中的标准I/O例程对这两个I/O流进行行缓冲，这类似于前面所做的显示调用setvbuf. 第19章将用伪终端实现这种方法。
  

### 15.5 FIFO
  FIFO有时被称为命名管道。未命名的管道只能在两个相关的进程之间使用，而且这两个相关的进程还要有一个共同创建了它们的祖先进程。但是FIFO，不相关的进程也能交换数据。
  
  第14章中已经提及FIFO是一种文件类型。通过stat结构的st_mode成员编码可以知道文件是否是FIFO类型。可以用S_ISFIFO宏对此进行测试。
  创建FIFO类似于创建文件。确实，FIFO的路径名存在于文件系统中。
```
#include <sys/stat.h>
int mkfifo(const char *path, mode_t mode);
int mkfifoat(int fd, const char *path, mode_t mode);
```
  mkfifo函数中的mode参数规格说明与open函数中的mode的相同。新FIFO的用户和组的所有权规则与4.6节所述相同。
  mkfifoat函数和mkfifo函数相似，但是mkfifoat函数可以被用来在fd文件描述符表示的目录相关的位置创建一个FIFO. 像其他*at函数一样，这里有三种情形:
  1. 如果path参数指定的是绝对路径名，则fd参数会被忽略掉，并且mkfifoat函数的行为和mkfifo类似。
  2. 如果path参数指定的是相对路径名，则fd参数是一个打开目录的有效文件描述符，路径名和目录有关。
  3. 如果path参数指定的是相对路径名，并且fd参数有一个特殊值AT_FDCWD, 则路径名以当前目录开始，mkfifoat和mkfifo类似。
  
  当我们用mkfifo或和mkfifoat创建FIFO时，要用open来打开它。确实，正常的文件I/O函数(如close, read, write和unlink)都需要FIFO.
> 应用程序可以用mknod和mknodat函数创建FIFO. 因为POSIX.1原先并没有包括mknod函数，所以mkfifo是专门为POSIX.1设计的。mknod和mknodat函数现在以及包括在POSIX.1的XSI扩展中。
> POSIX.1也包括了对mkfifo(1)命令的支持。本书讨论的4中平台都提供此命令。因此，可以用一条shell命令创建一个FIFO, 然后用一般的shell I/O重定向对其进行访问。

  当open一个FIFO时，非阻塞标志(O_NONBLOCK)会产生下列影响。
  * 在一般情况下(没有指定O_NONBLOCK),只读open要阻塞到某个其他进程为写而打开这个FIFO为止。类似的，只写open要阻塞到某个其他进程为读而打开它为止。
  * 如果指定了O_NONBLOCK，则只读open立即返回。但是，如果没有进程为读而打开一个FIFO, 那么只写open将返回-1,并将errno设置成ENXIO.
  
  类似于管道，若write一个尚无进程为读而打开的FIFO,则产生信号SIGPIPE.若某个FIFO的最后一个写进程关闭了该FIFO, 则将为该FIFO的读进程产生一个文件结束标志。

  一个给定的FIFO有多个写进程是常见的。这就意味着，如果不希望多个进程所写的数据交叉，则必须考虑原子写操作。和管道一样，常量PIPE_BUF说明了可被原子的写到FIFO的最大数据量。
  FIFO有以下两种用途。
  1. shell命令使用FIFO将数据从一条管道传送到另一条时，无需创建中间临时文件。
  2. 客户进程-服务器进程应用程序中，FIFO用作聚集点，在客户进程和服务器进程两者之间传递数据。
  
  我们各用一个实例来说明这两种用途。
  实例: 用FIFO复制输出流
  FIFO可用于复制一系列shell命令的输出流。这就防止了将数据写向中间磁盘文件(类似于用管道来避免中间磁盘文件)。但是不同的是，管道只能用于两个进程之间的线性连接，而FIFO是有名字的，因此它可用于非线性连接。
  考虑这样的一个过程，它需要对一个经过过滤的输入流进行两次处理。下图显示了这种安排:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/fifo_2_way_done.png)

  使用FIFO和Unix程序tee(1)就可以实现这样的过程而无需使用临时文件。(tee程序将其标准输入同时复制到其标准输出已经其命令行中命令的文件中。)
```
mkfifo fifo1
prog3 < fifo1 & prog1 < infile | tee fifo1 | prog2
```
  创建FIFO，然后启动prog3从FIFO读数据。然后启动prog1, 用tee将其输出发送到FIFO和prog2. 下图显示了这种安排。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/fifo_send_to_2procesess.png)
  
  实例: 使用FIFO进行客户进程－服务器进程通信
  FIFO的另一个用途是在客户端进程和服务器进程之间传送数据。如果有一个服务器进程，它与很多客户进程有关，每个客户端都可将其请求写到一个该服务器进程创建的众所周知的FIFO中(众所周知的意思是: 所有需要与服务器进程联系的客户进程都知道该FIFO的路径名)。如下图:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/server_wellknown_fifo.png)
  
  因为该FIFO有多个写进程，所以客户进程发送给服务器进程的请求长度要小于PIPE_BUF字节。这样就能避免客户进程的多次写之间的交叉。
  
  在这种类型的客户进程-服务器进程通信中使用FIFO的问题是: 服务器进程如何将回答送回各个客户进程。不能使用单个FIFO, 因为客户进程不可能知道何时去读它们的响应以及何时响应其他客户进程。一种解决方法是，每隔客户进程都在其请求中包含它的进程ID, 然后服务器进程为每个客户端进程创建一个FIFO, 所使用的路径名是以客户进程的进程ID为基础的。例如，服务器进程可以用名字/tmp/serv1.XXXX创建FIFO, 其中XXXX被替换成客户进程的进程ID. 如下图所示:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/server_client_communication_with_fifo.png)
  
  虽然这种安排可以工作，但服务器进程不能判断一个客户进程是否崩溃终止，这就使得客户进程专用FIFO会遗留在文件系统中。另外，服务器进程还必须捕捉SIGPIPE信号，因为客户端进程在发送一个请求后，有可能没有读取响应就终止了，于是留下一个只有写进程(服务器进程)而无读进程的客户进程专用FIFO.
  
  按上图的安排，如果服务器进程以只读的方式打开众所周知的FIFO(因为它只需要读该FIFO)，则每当客户进程个数从1变成0时，服务器进程就将在FIFO中读到(read)一个文件结束标志。为使服务器进程免于处理这种情况，一种常见的技巧时使服务器进程以读写方式打开众所周知的FIFO.
  

### 15.6 XSI IPC
  有3种称作XSI IPC的IPC: 消息队列、信号量以及共享存储器。它们之间有很多相似之处。本节先介绍它们相类似的特征，后面几节将说明这些IPC各自的特殊功能。
> XSI IPC函数是紧密的基于System V的IPC函数的。这3种类型的XSI IPC源自于1970年的一种称为Columbus Unix的AT & T内部版本。后来它们被添加到System V上。由于XSI IPC不使用文件系统命名空间，而是构造了它们自己的命名空间，为此常常受到批评。

#### 15.6.1 标识符和key
  每个内核中的IPC结构(消息队列、信号量或共享存储段)都用一个非负整数的标识符(identifier)加以引用。例如，要向一个消息队列发送消息或从一个消息队列取消息，只需要知道其队列标志符。与文件描述符不同，IPC标识符不是小的整数。当一个IPC结构被创建，然后又被删除时，与这种结构相关的标识符连续加1，直至达到一个整数的最大正值，然后又回转到0.
  
  标识符是IPC对象的内部名。为使多个合作进程能够在同一IPC对象上汇聚，需要提供一个外部命名方案。为此，每个IPC对象都与一个键(key)相关联，将这个键作为该对象的外部名。
  
  无论何时创建IPC结构(通过调用msgget,semget或shmget创建)，都应指定一个键。这个键的数据类型是基本系统数据类型key_t, 通常在头文件<sys/types.h>中被指定为长整型。这个键由内核变换成标识符。
  
  有多种方法使客户进程和服务器进程在同一个IPC结构上汇聚。
  1. 服务器进程可以指定IPC_PRIVATE创建一个新IPC结构，将返回的标识符存放在某处(如一个文件)以便客户进程取用。键IPC_PRIVATE保证服务器进程创建一个新IPC结构。这种技术的缺点是: 文件系统操作需要服务器进程将整型标识写到文件中，此后客户进程又要读这个文件取得此标识符。IPC_PRIVATE键可用于父进程子关系。父进程指定IPC_PRIVATE创建一个新的IPC结构，所返回的标识符可供fork后的子进程使用。 接着子进程又可将此标识符作为exec函数的一个参数传递给一个新程序。
  2. 可以在一个公用文件中定义一个客户进程和服务器进程都认可的键。然后服务器进程指定此键创建一个新的IPC结构。 这种方法的问题是该键可能已与一个IPC结构相结合，在此情况下，get函数(msgget, semget, shmget)出错返回。服务器进程必须处理这一错误，删除已存在的IPC结构，然后试着再创建它。
  3. 客户进程和服务器进程认同一个路径名和项目ID(项目ID是0~255之间的字符值)，接着调用函数fork将这两个值变换为一个键，然后在方法(2)使用此键，ftok提供的唯一服务就是由一个路径名和项目ID产生的一个键。
```
#include <sys/ipc.h>
key_t ftok(const char *path, int id);
```
  path参数必须引用一个现有的文件。当产生键时，只使用id参数的低8位。
  ftok创建的键通常是用下列方式构成的: 按给定的路径名取得其stat结构中的部分st_dev和st_info字段，然后再将它们与项目ID组合起来。如果两个路径名引用的是两个不同的文件，那么ftok通常会为这两个路径名返回不同的键。但是，因为i节点编号和键通常都存放在长整型中，所以创建键时可能会丢失信息。这意味着，对于不同文件的两个路径名，如果使用同一项目ID，那么可能产生相同的键。
  3个get函数(msgget, semget, 和shmget)都有两个类似的参数: 一个key和整型flag. 在创建新的IPC结构(通常由服务器进程创建)时，如果key是IPC_PRIVATE或者和当前某种类型的IPC结构无关，则需要指明flag的IPC_CREAT标志位。为了引用一个现有队列(通常由客户进程创建)，key必须等于队列创建时指明的key的值，并且IPC_CREAT必须不被指明。
  
  注意，绝不能指定IPC_PRIVATE作为键来引用一个现有队列，因为这个特殊的键值总是用于创建一个新队列。为了引用一个用IPC_PRIVATE键创建的现有队列，一定要知道这个相关标识符，然后在其他IPC调用中(如msgsnd, msgrcv)使用该标识符，这样可以绕过get函数。
  
  如果创建一个新的IPC结构，而且要确保没有引用具有同一标识符的一个现有IPC结构，那么必须在flag中同时指定IPC_CREAT和IPC_EXCL位。这样做了以后，如果IPC结构已经存在就会造成出错，返回EEXIST(这与指定了O_CREAT和O_EXCL标志的open相类似。)
  
  
#### 15.6.2 权限结构
  XSI IPC为每一个IPC结构关联了一个ipc_perm结构。该结构规定了权限和所有者，它至少包括以下成员:
```
struct ipc_term{
  uid_t uid; /** 所有者的有效用户ID **/
  gid_t gid; /** 所有者的有效组ID **/
  uid_t cuid; /** 创建者有效用户ID **/
  gid_t cgid; /** 创建者有效组ID **/
  mode_t mode; /** 访问模式 **/
  ...
}
```
  每个实现会包括另外一些成员。如欲了解所有系统的其他完整定义，可参见<sys/ipc.h>. 
  在创建IPC结构时，对所有字段都赋初值。以后可以调用msgctl、semctl、shmctl修改uid,gid和mode字段。 为了修改这些值，调用进程必须是IPC结构的创建者或超级用户。修改这些字段类似于对文件调用chown和chmod.
  
  mode字段的值类似于图4-6所示的值，但是对于任何IPC结构都不存在执行权限。另外，消息队列和共享存储使用术语读和写，而信号量则用术语读和更改。 图15-24显示了每种IPC的6种权限.
```
权限          位
--------------------
用户读        0400
用户写(更改)  0200
--------------------
组读          0040
组写(更改)    0020
--------------------
其他读        0004
其他写(更改)  0002
--------------------
```
  某些实现定义了标识每种权限的符号常量，但是这些常量并不包括在Single Unix Specification中。


#### 15.6.3 配置限制
  所有3种形式的XSI IPC都有内置限制。大多数限制可以通过重新配置内核来改变。在对这3种形式的IPC的每一种进行描述时，我们会指出它们的限制。
> 在报告和修改限制方面，每种平台都有自己的方法。Free BSD 8.0、Linux 3.2.0和Mac OS X 10.6提供了sysctl命令来观察和修改内核配置参数。在Solaris 10中，可以用prctl命令来改变内核IPC限制。
> 在Linux中，可以运行ipcs -l来显示IPC相关的限制。在FreeBSD中，等效的命令是ipcs -T. 在Solaris中，可以通过运行sysdef -y来找到可调节参数。


#### 15.6.4 优点缺点
  XSI IPC的一个基本问题是:IPC结构是在系统范围内起作用的，没有引用计数。例如，如果进程创建了一个消息队列，并且在该队列中放入了几则消息，然后终止，那么该消息队列及其内容不会被删除。它们会一直留在系统中直到发生下列动作为止:
  * 由某个进程调用msgrcv或msgctl读消息或删除消息队列
  * 某个进程执行ipcrm(1)命令删除消息队列；
  * 正在自举的系统删除消息队列
  
  将此与管道相比，当最后一个引用管道的进程终止时，管道就被完全地删除了。对于FIFO而言，在最后一个引用FIFO地进程终止时，虽然FIFO地名字仍保留在系统中，直至显示地删除，但是留在FIFO中地数据已被删除。

  XSI IPC地另一个问题是: 这些IPC结构在文件系统中没有名字。我们不能用第三章和第四章中所述地函数来访问它们或修改它们地属性。为了支持些IPC对象，内核增加了十几个全新地系统调用(msgget, semop, shmat等). 我们不能用ls命令查看IPC对象，不能用rm删除它们。也不能用chmod命令修改它们的访问权限。于是，又增加了两个新命令ipcs(1), ipcrm(1). 
  因为这些形式的IPC不能使用文件描述符，所以不能对他们使用多路转接I/O函数(select和poll). 这使得它很难一次使用一个以上这样的IPC结构，或者在文件或设备I/O中使用这样的IPC结构。例如，如果没有某种形式的忙等循环(busy-wait loop)，就不能使一个服务器进程等待将要放在两个消息队列中任意一个中的消息。
  
### 15.7 消息队列
  消息队列是消息的链接表，存储在内核中，由消息队列标识符标识。在本节中，我们把消息队列简称队列，其标识符简称队列ID.
> Single Unix Specification的消息传送选项中包括一种替代的IPC消息队列接口，该接口来源于POSIX实时扩展。本书不讨论这个接口。

  msgget用于创建一个新队列或打开一个现有队列。msgsnd将新消息添加到队列尾端。每个消息包含一个正的长整型类型的字段，一个非负的长度以及十几数据字节(对应于长度)，所有这些都在将消息添加到队列时，传送给msgsnd、msgrcv用于从队列中取消息。我们并不一定要已先进先出次序取消息，也可按消息的类型字段取消息。
  每个队列都有一个msqid_ds结构与其关联:
```
struct msqid_ds {
  struct ipc_perm; /** ipc权限  **/
  msgqnum_t msg_qnum; /** 队列中信息的数量 **/
  msglen_t msg_qbytes; /** 队列的最大字节数 **/
  pid_t msg_lspid; /** 上一个msgsnd()的pid **/
  pid_t msg_lrpid; /** 上一个msgrcv()的pid **/
  time_t msg_stime; /** 上一个msgsnd()的时间 **/
  time_t msg_rtime; /** 上一个msgrcv()的时间 **/
  time_t msg_ctime; /** 上次修改时间 **/
  
  ...
}
```
  此结构定义了队列的当前状态。结构中所示的各成员是由Single Unix Specification定义的。具体实现可能包括标准中没有定义的另一些字段。
  
  下图列出了影响消息队列的系统限制。"导出的"表示这种限制来源于其他限制。例如，在Linux系统中，最大消息数是根据最大队列数和队列中所允许的最大数据量来决定。其中最大队列数还要根据系统上安装RAM的数量来决定。注意，队列的最大字节数限制进一步限制了队列中要存储的消息的最大长度。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/msgq_sys_limits.png)
  
  调用的第一个函数通常是msgget, 其功能是打开一个现有队列或创建一个新队列。
```
#include <sys/msg.h>
int msgget(key_t key, int flag);
```
  15.6.1节说明了将key变换成一个标识符的规则，并且讨论了是创建一个新队列还是引用一个现有队列。在创建新队列时，要初始化msgid_ds结构的下列成员。
  * ipc_perm结构按15.6.2节中所述进行初始化。该结构中的mode成员按flag中的相应权限位设置。这些权限用图15-24中的值指定。
  * msg_qnum, msg_lspid, msg_lrpid, msg_stime和msg_rtime都设置为0.
  * msg_ctime设置为当前时间
  * msg_qbytes设置为系统限制值。

  若执行成功，msgget返回非负队列ID. 此后，该值就可以用于其他3个消息队列函数。
  msgctl函数对队列执行多种操作。它和另外两个与信号量及共享存储有关的函数(semctl和shmctl)都是XSI IPC的类似于ioctl的函数(亦即垃圾桶函数)。
```
#include <sys/msg.h>
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
```
  cmd参数指定对msqid指定的队列要执行的命令。
  * IPC_STAT: 取此队列的msqid_ds结构，并将它存放在buf指向的结构中。
  * IPC_SET: 将字段msg_perm.uid, msg_perm.gid, msg_perm.mode和msg_qbytes从buf指向的结构复制到与这个队列相关的msqid_ds结构中。此命令只能由下列两种进程执行: 一种是其有效用户ID等于msg_perm.cuid或msg_perm.uid; 另一种是具有超级用户特权的进程。只有超级用户才能增加msg_qbytes的值。
  * IPC_RMID: 从系统中删除该消息队列以及仍在该队列中的所有数据。这种删除立即生效。仍在使用这一消息队列的其他进程在它们下一次试图对此队列进行操作时，得到EIDRM错误。此命令只能由以下两种进程执行: 一种是其有效用户ID等于msg_perm.cuid或msg_perm.uid; 另一种是具有超级用户特权的进程。
  上面这三个命令(IPC_STAT, IPC_SET, IPC_RMID)可用于信号量和共享存储。

  调用msgsnd将数据放到消息队列中
```
#include <sys/msg.h>
int msgsnd(int msqid, const void *ptr, size_t nbytes, int flag);
```
  正如前面提及的，每个消息都由3部分组成: 一个正的长整型类型的字段，一个非负的长度(ntypes)以及实际数据字节数(相应于长度)，消息总是放在队列的尾端。
  ptr参数指向一个长整型数，它包含了正的整型消息类型，其后紧挨着的是消息数据(若nbytes长度为0，则无消息数据)。若发送的最长消息是512字节的，则可定义下列结构:
```
struct mymesg {
  long mytype; /** 正的消息类型 **/
  char mtext[512]; /** message data, of length nbytes **/
}
```

  ptr就是一个指向mymesg结构的指针。接收者可以使用消息类型以非先进先出的次序取消息。
> 某些平台既支持32位环境，又支持64位环境。这影响到长整型和指针的大小。例如，在64位SPARC系统中，Solaris允许32位应用程序和64位应用程序同时存在。如果一个32位应用程序要经由管道或套接字与一个64位应用程序交换此结构，就会出问题。因为在32位应用程序中，长整型的大小是4字节，而在64位应用程序中，长整型的大小是8字节。这意味着，32位应用程序期望mtext字段在结构起始地址后的第4个字节处开始，而64位应用程序的mtype字段的一部分会被32位应用程序视为mtext的字段组成部分，而32位应用的mtext前4字节会被64位应用程序解释为mtype字段的组成部分。

> 但是， XSI消息队列就不会发生这种问题。Solaris实现的IPC系统调用的32位版本和64位版本具有不同的入口点。这些系统调用知道如何处理32位应用程序与64位应用程序的通信操作，并对类型字段做了特殊处理以避免它干扰消息的数据部分。唯一的潜在问题是，当64位应用程序向32位应用程序发送消息时，如果它在8字节类型字段中设置的值大于32位应用程序中的4字节类型字段可表示的值，那么32位应用程序在其mtype字段中得到的将是一个截短了的类型值。

  参数flag的值可以指定位IPC_NOWAIT。 这类似于I/O的非阻塞I/O标志。若消息队列已满(或者是队列中的消息总数等于系统限制值)，则指定IPC_NONWAIT使得msgsnd立即出错返回EAGAIN. 如果没有指定IPC_NOWAIT， 则进程会一直阻塞到: 有空间可以容纳要发送的消息；或从系统中删除了此队列；或捕捉到一个信号，并从信号处理程序返回。在第二种情况下，会返回EIDRM错误("标识符被删除")。最后一种情况则返回EINTER错误。
  注意，对删除消息队列的处理不是很完善。因为每个消息队列没有维护引用计数器(打开文件有这种计数器)，所以在队列被删除以后，仍在使用这一队列的进程在下次对队列进行操作时会出错返回。信号量机构也以同样的方式处理其删除。相反，删除一个文件时，要等到使用该文件的最后一个进程关闭了它的文件描述符以后，才能删除文件中的内容。
  
  当msgsnd返回成功时，消息队列相关的msqid_ds结构会随之更新，表明调用的进程ID(msg_lspid)、调用的时间(msg_stime)以及队列中新增的消息(msg_qnum). 
  
  msgrcv从队列中取用消息。
```
#include <sys/msg.h>
ssize_t msgrcv(int msqid, void *ptr, size_t nbytes, long type, int flag);
```
  和msgsnd一样，ptr参数指向一个长整型数(其中存储的是返回的消息类型)，其后跟随的是存储实际消息数据的缓冲区。nbytes指定数据缓冲区的长度。若返回的消息长度大于nbytes，而且在flag中设置了MSG_NOERROR位，则该消息会被截断(在这种情况下，没有通知告诉我们消息截断了，消息被截去的部分被丢弃)。如果没有设置这一标志，而消息又太长，则出错返回E2BIG(消息仍留在队列中)。
  参数type可以指定想要哪种消息。
  * type == 0: 返回队列中的第一个消息
  * type > 0: 返回队列中消息类型为type的第一个消息
  * type < 0: 返回队列中消息类型值小于等于type绝对值的消息，如果这种消息有若干个，则取类型值最小的消息。
  
  type值非0用于以非先进先出次序读消息。例如，若应用程序对消息赋予优先权，那么type就可以是优先权位值。如果一个消息队列由多个客户进程和一个服务器进程使用，那么type字段可以用来包含客户进程的进程ID(只要进程ID可以存放在长整型中)。

  可以将flag值指定位IPC_NOWAIT, 使操作不阻塞，这样，如果没有所指定类型的消息可用，则msgrcv返回-1, error设置位ENOMSG。如果没有指定IPC_NOWAIT，则进程会一直阻塞到有了指定类型的消息可用，或者从系统删除了此队列(返回-1, errno设置为EINTR).
  msgrcv成功执行时，内核会更新与该消息队列相关联的msgid_ds结构，以指示调用者的进程ID(msg_lrpid)和调用时间(msg_rtime), 并指示队列中的消息数减少了1个(msg_qnum).
  
  

### 15.8 信号量
  信号量与已经介绍过的IPC机构(管道、FIFO以及消息队列)不同。它是一个计数器，用于为多个进程提供对共享数据对象的访问。
> Single Unix Specification包括了另外一套信号量接口，该接口原来是实时扩展的一部分。我们将在15.10节讨论这种接口。

  为了获得共享资源，进程需要执行下列操作。
  1. 测试控制该资源的信号量
  2. 若此信号量的值为正，则进程可以使用该资源。在这种情况下，进程会将信号量值减1，表示它使用了一个资源单位。
  3. 否则，此信号量的值为0，则进程进入休眠状态，直至信号量大于0.进程被唤醒后，它返回第1步。

  当进程不再使用由一个信号量控制的共享资源时，该信号量值增加1.如果有进程正在休眠等待此信号量，则唤醒它们。
  
  为了正确的实现信号量，信号量值的测试及减1操作应该是原子操作。为此，信号量通常是在内核中实现的。
  
  常用的信号量形式被称为二元信号量(binary semaphore). 它控制单个资源，其初始值为1. 但是，一般而言，信号量的初值可以是任意一个正值，该值表明有多少个共享资源单位可供共享应用。
  
  遗憾的是，XSI信号量与此相比要复杂的多。以下3种特性造成了这种不必要的复杂性。
  1. 信号量并非是单个非负值，而必须定义为包含有一个或多个信号量值的集合。当创建信号量时，要指定集合中信号量值的数量。
  2. 信号量的创建(semget)是独立与它的初始化(semctl)的。这是一个致命的缺点，因为不能原子的创建一个信号量集合，并且对该集合中各个信号量值赋初值。
  3. 即使没有进程正在使用各种形式的XSI IPC, 它们依然是存在的。有的程序在终止时并没有释放已经分配给它的信号量，所以我们不得不为这种程序担心。后面将要说明的undo功能就是处理这种情况的。
  内核为每隔信号量集合维护着一个semid_ds结构:
```
struct semid_ds{
  struct ipc_perm sem_perm; 
  unsigned short sem_nsems;
  time_t sem_otime; /** 上次semop()时间 **/
  time_t sem_ctime; /** 上次修改时间 **/
  ...
}
```
  Single Unix Specification定义了上面所示的各字段，但是具体实现可在semid_ds结构中定义添加的成员。
  
  每个信号量由一个无名结构表示，它至少包含下面成员:
```
struct {
  unsigned short semval;      /** 信号量值，总是大于等于0 **/
  pid_t sempid;      /** 上次操作的pid **/
  unsigned short semncnt;    /** # processes awaiting semval > curval **/
  unsigned short semzcnt;    /** # processes awaiting semval == 0 **/
  ...
}
```
  下图列出了影响信号量集合的系统限制:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/semapore_sys_limits.png)

  当我们想使用XSI信号量时，首先需要通过调用semget函数来获得一个信号量ID.
```
#include <sys/sem.h>
int semget(key_t, int nsems, int flags);
```
  15.6.1节说明了将key变换为标识符的规则，讨论了是创建一个新集合，还是引用一个现有集合。创建一个新集合时，要对semid_ds结构的下列成员赋初值。
  * 按15.6.2节中所述，初始化ipc_perm结构。该结构中的mode成员被设置为flag中的相应权限位。这些权限是15-24中值设置的。
  * sem_otime设置为0
  * sem_ctime设置为当前时间
  * sem_nsems设置为nsems
  nsems是该集合中信号量数。如果是创建新集合(一般在服务器进程中)，则必须指定nsems. 如果是引用现有集合(一个客户进程)，则nsems指定为0.

  semctl函数包含了多种信号量操作。
```
#include <sys/sem.h>
int semctl(int semid, int semnum, int cmd, ... /* union semun arg */);
```
  第四个参数是可选的，是否使用取决于所请求的命令，如果使用该参数，则其类型为semun，它是多个命令特定参数的联合体(union):
```
union semun{
  int val; /** for SETVAL **/
  struct semid_ds *buf; /** for IPC_STAT and IPC_SET **/
  struct short *array; /** for GETALL and SETALL **/
}
```

  注意，这个选项参数是一个联合体，而非指向联合的指针。
  
> 通常应用程序必须定义semun集合。然而，在FreeBSD 8.0中，semun已经由<sys/sem.h>为我们定义好了。

  cmd参数指定下列10种命令中的一种，这些命令是运行在semid指定的信号量集合上的。其中有5种命令是针对一个特定的信号量值的，它们用semnum指定该信号量集合中的一个成员。semnum值在0和nsems-1之间，包括0和semnum-1.
  * IPC_STAT: 对此集合取semid_ds结构，并存储在由arg.buf指向的结构中。
  * IPC_SET: 按arg.buf指向的结构中的值，设置与此集合相关的结构中的sem_perm.uid,sem_perm.gid和sem_perm.mode字段。此命令只能由两种进程执行: 一种是其有效用户ID等于sem_perm.cuid或sem_perm.uid的进程。 另一种是具有超级用户特权的进程。
  * IPC_RMID: 从系统中删除该信号量集合。这种删除是立即发生的。删除时仍在使用此信号量集合的其他进程，在它们试图对此信号集合进行操作时，将出错返回EIDRM。 此命令只能由两种进程执行:  一种是其有效用户ID等于sem_perm.cuid或sem_perm.uid的进程。 另一种是具有超级用户特权的进程。
  * GETVAL: 返回成员semnum的semval值。
  * SETVAL: 返回成员semnum的semval值。该值由arg.val指定。
  * GETPID: 返回成员semnum的sempid值。
  * GETNCNT: 返回成员semnum的semncnt值。
  * GETZCNT: 返回成员semnum的semzcnt值。
  * GETALL: 取该集合中所有的信号量值。这些值存储在arg.array指向的数组中。
  * SETALL: 将该集合中所有的信号量值设置成arg.array指向的数组中。
  对于GETALL以外的所有GET命令，semctl函数都返回相应值。对于其他命令，若成功则返回0，若出错，则设置errno并返回-1.

  函数semop()自动执行信号量集合上的操作数组。
```
#include <sys/sem.h>
int semop(int semid, struct sembuf semoparray[1], size_t nops);
```
  参数semoparray是一个指针，它指向一个由sembuf结构表示的信号量操作数组:
```
struct sembuf{
  unsigned short sem_num; /** member # in set (0, 1, ..., nsems-1) **/
  short          sem_op;  /** operation(negative, 0, or pasitive **/
  short          sem_flg; /** IPC_NOWAIT, SEM_UNDO **/
};
```
  参数nops规定该数组中操作的数量(元素数)。
  对集合中每个成员的操作由相应的sem_op值规定。此值可以是负值、0或正值。
  
### 15.9 共享存储
  共享存储允许两个或多个进程共享一个给定的存储区。因为数据不需要在客户进程和服务器进程之间复制，所以这是最快的一种IPC.使用共享存储时要掌握的唯一窍门是，在多个进程之间同步访问一个给定存储区。若服务器进程正在将数据放入共享存储区，则在它做完这一操作之前，客户进程不应当去取这些数据。通常，信号量用于同步共享存储访问。(不过正如前节最后部分所述，也可以用记录锁或互斥量。)
  
> Single Unix Specification在其存储对象选项中包括了访问共享存储的替代接口，这些接口源于实时扩展。

  我们已经看到了共享存储的一种形式，就是在多个进程将同一个文件映射到它们的地址空间的时候。XSI共享存储和内存映射的不同之处在于，前者没有相关的文件。XSI共享存储段是内存的匿名段。
  
  内核为每个共享存储段维持着一个结构，该结构至少要为每个共享段包含以下成员:
```
struct shmid_ds{
  struct ipc_perm shm_perm;
  size_t shm_segsz; /** 段尺寸，单位字节 **/
  pid_t shm_lpid; /** 上次shmop()的pid **/
  pid_t shm_cpid; /** 创建者pid **/
  shmatt_t shm_nattch; /** 当前附加的数量 **/
  time_t shm_atime; /** last attach time; **/
  time_t shm_dtime; /** last detach time; **/
  time_t shm_ctime; /** last-change time; **/
  ...
}
```
  (按照支持共享存储段的需要，每种实现会增加其他结构成员。)
  
  shmatt_t类型定义为无符号整型，它至少与unsigned short一样大。下图列出了影响共享存储的系统限制。
  
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/share_memory_sys_limits.png)
  
  调用的第一个函数通常是shmget, 它获得一个共享存储标识符号。
```
#include <sys/shm.h>
int shmget(key_t key, size_t size, int flag);
```
  15.6.1节说明了将key变换成一个标识符的规则，以及是创建一个新共享存储段，还是引用一个现有的共享存储段。当创建一个新段时，初始化shmid_ds结构的下列成员:
  * ipc_perm: 类似其他IPC
  * shm_lpid, shm_nattach, shm_atime和shm_dtime都设置为0
  * shm_ctime设置为当前时间
  * shm_segsz设置为请求的size
  
  参数size是该共享存储段的长度，以字节为单位。实现通常将其向上取为系统页长的整数倍。但是，若应用指定的size并非系统页长的整数倍，那么最后一页的余下部分是不可使用的。 如果正在创建的一个新段(通常在服务器中)，则必须指定其size. 如果正在引用一个现存的段(一个客户进程)，则size指定为0. 当创建一个新段时，段内的内容初始化为0.

  shmctl函数对共享存储段执行多种操作:
```
#include <sys/shm.h>
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```
  cmd参数指定下列5种命令中的一种，使其在shmid指定的段上执行。
  * IPC_STAT: 取此段的shmid_ds结构，并将它存储在由buf指向的结构中。
  * IPC_SET: 按buf指向的结构中的值设置与此共享存储段相关的shmid_ds结构中的下列3个字段: shm_perm.uid, shm_perm.gid, shm_perm.mode. 此命令只能由下列两种进程执行:一种是其有效用户ID等于sem_perm.cuid或sem_perm.uid的进程。 另一种是具有超级用户特权的进程。
  * IPC_RMID: 从系统删除该共享存储段。因为每个共享存储段维护着一个连接计数(shmid_ds结构中的shm_attch字段)，所以除非使用该段的最后一个进程终止或与该段分离，否则不会实际上删除该存储段。不管此段是否仍在使用，该段标识符都会被立即删除，所以不能在用shmat与该段连接。此命令只能由下列两种进程执行:一种是其有效用户ID等于sem_perm.cuid或sem_perm.uid的进程。 另一种是具有超级用户特权的进程。
  
  Linux和Solaris提供了另外两种命令，但它们并非Single Unix Specification的组成部分。
  * SHM_LOCK: 在内存中对共享存储段加锁。此命令只能由超级用户执行。
  * SHM_UNLOCK: 解锁共享存储段。此命令只能由超级用户执行。
  
  一旦创建了一个共享存储段，进程就可以调用shmat将其连接到它的地址空间中。
```
#include <sys/shm.h>
void *shmat(int shmid, const void *addr, int flag);
```
  共享存储段连接到调用进程的那个地址上与addr参数以及flag中是否指定SHM_RND位有关。
  * 如果addr为0，则此段连接到由内核选择的第一个可用地址上。这是推荐的使用方法。
  * 如果addr非0，并且没有指定SHM_RND， 则此段连接到addr锁指定的地址上。
  * 如果addr非0，并且指定了SHM_RND, 则此段连接到(addr - (addr mod SHMLBA))所表示的地址上。SHM_RND命令的意思是取整。SHMLBA的意思是"低边界地址倍数"，它总是2的乘方。该算式是将地址向下取最近一个SHMLBA的倍数。
  除非只计划在一种硬件上运行应用程序(这在当今是不可能的)，否则不应指定共享存储段所连接到的地址。而是应当指定addr为0，以便由系统选择地址。
  如果flag中设置了SHM_RDONLY位，则以只读方式连接此段，否则以读写方式连接此段。

  shmat的返回值是该段所连接的实际地址，如果出错则返回-1. 如果shmat成功执行，那么内核将使与该共享存储段相关的shmid_ds结构中的shm_nattch计数器增加1.
  
  当对共享存储段的操作已经结束，则调用shmdt与该段分离。注意，这并不从系统中删除其标识符以及其相关的数据结构。该标识符依然存在，知道某个进程(一般是否服务器进程)带IPC_RMID命令的调用shmctl特地的删除它为止。
  
```
#include <sys/shm.h>
int shmdt(const void *addr);
```
  addr参数是以前调用shmat时的返回值。如果成功，shmat将使相关shmid_ds结构中的shm_nattch计数器值减1.
  
  实例: 内核将以地址0连接的共享存储段放在什么位置上与系统密切相关。下面程序打印了一些特定系统存放各种类型的数据的位置信息。
```
#include "apue.h"
#include <sys/shm.h>

#define ARRAY_SIZE  40000
#define MALLOC_SIZE 100000
#define SHM_SIZE    100000
#define SHM_MODE    0600 /** user read/write **/

char array[ARRAY_SIZE]; /** uninitialized data = bss **/

int main(void)
{
    int shmid;
    char *ptr, *shmptr;

    printf("array[] from %p to %p\n", (void *)&array[0], (void *)&array[ARRAY_SIZE]);
    printf("stack around  %p\n", (void *)&shmid);

    if((ptr = malloc(MALLOC_SIZE)) == NULL)
        err_sys("malloc error");
    printf("malloced from %p to %p\n", (void *)ptr, (void *)ptr+MALLOC_SIZE);

    if((shmid = shmget(IPC_PRIVATE, SHM_SIZE, SHM_MODE)) < 0)
        err_sys("shmget error");

    if((shmptr = shmat(shmid, 0, 0)) == (void *)(-1))
        err_sys("shmat error");

    printf("shared memory attached from %p to %p\n", (void *)shmptr, (void *)shmptr+SHM_SIZE);

    if(shmctl(shmid, IPC_RMID, 0) < 0)
        err_sys("shmctl error");

    exit(0);
}
```
  

### 15.10 POSIX信号灯

### 15.11 Client-Server属性

### 15.12 总结
