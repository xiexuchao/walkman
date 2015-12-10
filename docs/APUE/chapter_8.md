## 第八章 进程控制

### 8.1 引言
  本章介绍Unix的进程控制，包括创建新进程、执行程序和进程终止。还将说明进程的各种ID--实际、有效和保存的用户和组ID,以及它们如何受到进程控制原语的影响。本章也包括了解释器文件和system函数。本章以大多数unix系统所提供的进程会计机制结束。这使我们从一个不同角度了解进程控制功能。

### 8.2 进程标识符
  每个进程都有一个非负整型的唯一进程ID, 因为进程ID标识符总是唯一的，常将其用作其他标识符的一部分以确保其唯一性。
  
  进程ID虽然唯一，但是还是会重用的， 它们的ID会变成候选者以待重用。很多Unix系统实现延迟重用算法， 然而，新创建的进程赋予的ID都是和最近终止的进程ID不同的。 这样就防止新进程错误， 使用了前面使用的相同ID.
  
  有某些专用的进程:进程ID为0的是调度进程，常常被称为交换进程(swapper). 该进程并不执行任何磁盘上的程序--它是内核的一部分，因此也被称为系统进程。进程ID为1的通常是init进程，在自举过程(bootstrap)结束时由内核调用.该进程的程序文件在unix早期版本中是/etc/init, 在较新版本中是/sbin/init. 此进程负责在内核自举后启动一个unix系统。init通常读与系统有关的初始化文件(/etc/rc*文件),并将系统引导到一个状态(例如多用户)，init进程绝不会终止，它是一个普通的用户进程(与交换进程不同，它不是内核中的系统进程)，但是它以超级用户特权运行。本章稍后会说明init如何称为所有孤儿进程的父进程。
  
  在某些Unix的虚存实现中，进程ID 2是页精灵进程(pagedeamon), 此进程负责支持虚存系统的请求页操作。与交换进程一样，页精灵进程也是内核进程。
  
  除了进程ID, 每个进程还有一些其他标识符。下列函数返回这些标识符。
```
#include <sys/types.h>
#include <unistd.h>

pid_t getpid(void);       /** 返回调用进程的进程ID **/
pid_t getppid(void);      /** 返回进程的父进程ID **/
uid_t getuid(void);       /** 调用进程的实际用户ID **/
uid_t geteuid(void);      /** 调用进程的有效用户ID **/
gid_t getgid(void);       /** 调用进程的实际组ID **/
gid_t getegid(void);      /** 调用进程的有效组ID **/
```
  注意，这些函数都没有出错返回，在下一节中讨论fork函数时，将进一步讨论父进程ID, 4.4节中已经讨论了实际和有效用户及组ID.
  
  
### 8.3 fork函数
  一个现存进程调用fork函数是unix内核创建一个新进程的唯一方法(这并不适用于前节提及的交换进程、init进程和页精灵进程。 这些进程是由内核作为自举过程的一部分以特殊方式创建的)。
```
#include <sys/types.h>
#include <unistd.h>
pid_t fork(void);
```
  由fork创建的新进程被称为子进程(child process). 该函数被调用一次，但返回两次。两次返回的区别是子进程的返回值是0，而父进程的返回值是新子进程的进程ID. 将子进程ID返回给父进程的理由是:因为一个进程的子进程可以多于一个，所以没有一个函数使一个进程可以获得其所有子进程的进程ID. fork使子进程得到返回值0的理由是: 一个进程只会有一个父进程，所以子进程总是可以调用getppid()以获得其父进程的进程ID(进程ID 0总是由交换进程使用，所以一个子进程的进程ID不可能为0)。
  子进程和父进程继续执行fork之后的指令。子进程是父进程的复制品。例如，子进程获得父进程数据空间、堆和栈的复制品。注意，这是子进程所拥有的拷贝。父子进程并不共享这些存储空间部分。如果正文段是只读的，则父子进程共享正文段。
  
  现在很多实现并不做一个父进程数据段和堆的完全拷贝，因为在fork之后经常跟随着exec。作为替代，使用了在写时复制(Copy-On-Write, COW)的技术。这些区域由父子进程共享，而且内核将它们的存取许可权改变为只读的。如果有进程试图修改这些区域，则内核为有关部分，典型的是虚存系统中的页， 做一个拷贝。
```
#include "apue.h"

#include <sys/types.h>

int glob = 6;       /** external variable in initialized data **/
char buf[] = "a write to stdout\n";

int
main(void)
{
    int var; /** automatic variable on the stack **/
    pid_t pid;


    var = 88; 
    if(write(STDOUT_FILENO, buf, sizeof(buf) - 1) != sizeof(buf) - 1)
        err_sys("write error");

    printf("before fork\n");     /** we don't flush stdout **/
    if((pid = fork()) < 0)
        err_sys("fork error");
    else if(pid == 0) { /** child process **/
        glob++;         /** modify variables **/
        var++;
    } else {
        sleep(2);
    }   

    printf("pid = %d, glob = %d, var = %d\n", getpid(), glob, var);
    exit(0);
}
```
  编译并运行结果如下:
```
bogon:process apple$ ./proc1 
a write to stdout
before fork
pid = 77066, glob = 7, var = 89
pid = 77065, glob = 6, var = 88
bogon:process apple$ ./proc1 > temp.txt
bogon:process apple$ cat temp.txt
a write to stdout
before fork
pid = 77069, glob = 7, var = 89
before fork
pid = 77068, glob = 6, var = 88
```
  一般来说，在fork之后是父进程先执行还是子进程先执行是不确定的。这取决于内核所使用的调度算法。如果要求父子进程之间相互同步，则要求某种形式的进程间通信。上面程序中，父进程使自己休眠2秒钟，以使子进程先执行。但并不保证2秒钟已经足够，在后面8.8节说明竞争条件时，还将谈及这一问题及其他类型的同步方法。在10.6节中，在fork之后将用信号使父子进程同步。
  注意，上面程序中的fork与I/O函数之间的关系。回忆第三章中所述，write函数是不带缓存的。因为在fork之前调用write, 所以其数据洗到标准输出一次。但是，标准I/O库是带缓存的。当以交互方式运行该程序时，只得到printf输出行一次，其原因是标准输出缓存由新行符刷新。但是当标准输出重定向到一个文件时，该行数据仍在缓存中，然后在父进程数据空间复制到子进程的过程中时，该缓存数据也被复制到子进程中。于是那时父子进程各自有了带该行内容的缓存。在exit之前的第二个print将其数据添加到现存的缓存中。当每个进程终止时，其缓存中的内容被写到相应的文件中。
  
  文件共享，对上面程序需要注意的另一点是: 在重定向父进程的标准输出时，子进程的标准输出也被重定向。实际上，fork的一个特性是所有由父进程打开的描述符都被复制到子进程中。父子进程每个相同的打开描述符共享一个文件表项。
  
  考虑下述情况，一个进程打开了三个不同文件，它们分别是:标准输入、标准输出和标准出错，在从fork返回时，我们有了如图8-1中所示的安排。
  这种共享文件的方式使父子进程对同一个文件使用了一个文件位移量。考虑下述情况:一个进程fork了一个子进程，然后等待子进程终止。假定，作为普通处理的一部分，父子进程都向标准输出执行写操作。如果父进程使其标准输出重定向(很可能是由shell实现的)，那么子进程写到该标准输出时，它将更新与父进程共享的该文件的位移量。在我们所考虑的例子中，当父进程等待子进程时，子进程写到标准输出；而在子进程终止后，父进程也写到标准输出上，并且直到其输出会添加在子进程缩写数据之后。如果父子进程不共享同一文件位移量，这种形式的交互就很难实现。
  
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/fork_fds.png)
  
  如果父子进程写到同一个描述符文件，但又没有任何形式的同步(例如使父进程等待子进程)，那么它们的输出就会相互混合(假定所用的描述符是在fork之前打开的)。虽然这种情况是有可能发生的，但这并不是常用的操作方式。
  在fork之后处理文件描述符有两种常见的情况:
  1. 父进程等待子进程完成。这种情况下，父进程无需对其描述符做任何处理。当子进程终止后，它曾进行过读写操作的任一共享描述符的文件位移量已经做了相应更新。
  2. 父子进程各自执行不同的程序段。在这种情况下，在fork之后，父子进程各自关闭它们不需要使用的文件描述符，并且不干扰对方使用的文件描述符。这种方法是网络服务进程中进场使用的。
  除了打开文件之外，很多父进程的其他性质也由子进程继承:
  * 实际用户ID, 实际组ID, 有效用户ID,有效组ID
  * 添加组ID
  * 进程组ID
  * 对话期ID
  * 控制终端
  * set-user-ID和set-group-ID标志
  * 当前工作目录
  * 根目录
  * 文件方式创建屏蔽字
  * 信号屏蔽和排列
  * 对任一打开文件描述符的在执行时关闭标志
  * 环境
  * 连接的共享存储段
  * 资源限制
  
  父子进程的区别是:
  * fork的返回值
  * 进程ID
  * 不同的父进程ID
  * 子进程的tms_utime, tms_stime, tms_cutime以及tms_ustime设置为0
  * 父进程设置的锁，子进程不继承
  * 子进程的未决告警被清除
  * 子进程的未决信号集设置为空集。
  
  其中很多特性至今尚未讨论过，我们将在以后几章中对它们进行说明。

  使fork失败的两个主要原因是: 
  * 系统中已经有了太多的进程(通常意味着某个方面出了问题)
  * 该实际用户ID的进程总数超过了系统限制。CHILD_MAX规定了每个实际用户ID在任一时刻可具有的最大进程数。
  
  fork有两种用法:
  1. 一个父进程希望复制自己，使父子进程同时执行不同的代码段。这在网络服务进程中是最常见的--父进程等待委托着的服务请求。当这种请求到达时，父进程调用fork， 使子进程处理此请求。父进程则继续等待下一个服务请求。
  2. 一个进程要执行一个不同的程序。这对shell是常见的情况。在这种情况下，子进程在从fork返回后立即调用exec。
  某些操作系统将2中的两个操作(fork之后执行exec)组合成一个，并称其为spawn. Unix将这两个操作分开，因为在很多场合需要单独使用fork，然后并不跟随exec. 另外，将这两个操作分开，使得子进程在fork和exec之间可以更改自己的属性。例如I/O重定向、用户ID、信号排列等。在第14章有很多这方面的例子。
  
### 8.4 vfork函数
  vfork函数的调用序列和返回值与fork相同，但两者的语义不同。
```
  vfork起源于较早的4BSD虚存版本。在Leffler等[1989]的5.7节中指出:"虽然它是特别有效率的，但是vfork的语义很奇特，
  通常人为它具有结构上的缺陷。"
  
  尽管如此SVR4和4.3+BSD仍支持vfork.
  某些系统具有头文件<vfork.h>, 当调用vfork时，应当包括该头文件。
```
  vfork用于创建一个新进程，而该进程的目的是exec一个新程序。vfork和fork一样都是创建一个子进程，但是它并不将父进程的地址空间完全复制到子进程中，因为子进程会立即调用exec(或exit),于是也就不会存访该地址空间。不过在子进程调用exec或exit之前，它在父进程的空间中运行。这种工作方式在某些Unix的页式虚存实现中提高了效率。
  vfork和fork之间的另一个区别在于:vfork保证子进程先运行，在它调用exec或exit之后，父进程才可能被调度运行。(如果在调用这两个函数之前子进程依赖于父进程的进一步动作，则会导致死锁)。
```
#include "apue.h"

#include <sys/types.h>

int glob = 6;       /** external variable in initialized data **/
char buf[] = "a write to stdout\n";

int
main(void)
{
    int var; /** automatic variable on the stack **/
    pid_t pid;


    var = 88; 
    if(write(STDOUT_FILENO, buf, sizeof(buf) - 1) != sizeof(buf) - 1)
        err_sys("write error");

    printf("before fork\n");     /** we don't flush stdout **/
    if((pid = vfork()) < 0)
        err_sys("vfork error");
    else if(pid == 0) { /** child process **/
        glob++;         /** modify variables **/
        var++;
        _exit(0);
    } else {
        sleep(2);
    }   

    printf("pid = %d, glob = %d, var = %d\n", getpid(), glob, var);
    exit(0);
}
```
  运行程序得到：
```
bogon:process apple$ ./vfork 
a write to stdout
before fork
pid = 78116, glob = 7, var = 89
```
  子进程对变量glob和var做增1操作，结果改变了父进程中的变量值。因为子进程在父进程的地址空间中运行，所以这并不惊讶。但是其作用的确与fork不同。
  注意，上面子进程调用了_exit(0), 而不是exit(0). _exit并不执行标准I/O缓存的刷新操作。如果用exit而不是_exit，则程序不会输出最后一行。(抱歉，在我的macbook上面结果都一样，不知道why?)
  从中可见，父进程printf的输出消失了。其原因是子进程调用了exit, 它刷新关闭了所有标准I/O流，这包括标准输出。虽然这是由子进程执行的，但却是在父进程的地址空间中进行的，所以所有受到影响的标准I/O FILE对象都是在父进程中的。当父进程调用printf时，标准输出已被关闭了，于是printf返回-1.
  

### 8.5 exit函数
  前面章节我们说过，进程有三种正常终止法及两种异常终止法。
  1. 正常终止
  <ol>
    <li>在main函数内执行return语句。这等效于调用exit.</li>
    <li>调用exit函数。其操作包括调用各终止处理程序(终止处理程序在调用atexit时登录)，然后关闭所有标准I/O流等。因为ANSI C并不处理文件描述符、多进程(父子进程)以及作业控制，所以这一定义对Unix系统而言是不完整的。</li>
    <li>调用_exit系统调用函数。此函数由exit调用，他处理Unix特定细节。</li>
  </ol>
  2. 异常终止
  <ol>
    <li>调用abort. 它产生SIGABRT信号，所以是下一种异常终止的一个特例</li>
    <li>当进程接收到某个信号时。进程本身(例如调用abort)、其他进程和内核都能产生传送到某一进程的信号。例如，进程越出其地址空间访问存储单元，或者除以0，内核就会为该进程产生相应的信号。</li>
  </ol>

  不管进程如何终止，最后都会执行内核中的同一段代码。这段代码为相应进程关闭所有打开文件描述符，释放它所使用的存储器等。
  对上述任意一种终止情形，我们都希望终止进程能够通知其父进程它是如何终止的。对于exit和_exit，这是依靠传递给它们的退出状态(exit status)参数来实现的。在异常终止情况，内核(不是进程本身)产生一个指示其异常终止原因的终止状态(termination status). 在任意一种情况下， 该终止进程的父进程都能用wait或waitpid函数取得其终止状态。
  
  注意，这里使用了退出状态(它是传向exit或_exit的参数，或main的返回值)和终止状态两个术语，以表示有所区别。在最后调用_exit时，内核将其退出状态转换成终止状态。下一节我们会介绍父进程检查子进程的终止状态的不同方法。如果子进程正常终止，则父进程可以获得子进程的退出状态。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/c_startup_terminate.png)
  
  在说明fork函数时，一定是一个父进程生成一个子进程。上面又说明了子进程将其终止状态返回给父进程。但是如果父进程在子进程之前终止，则将如何呢? 其回答是对于其父进程已经终止的所有进程，它们的父进程都改变为init进程。我们称这些进程由init进程领养。其操作过程大致是:一个进程终止时，内核逐个检查所有活动进程，以判断它是否是正要终止的进程的子进程，如果是，则该进程的父进程ID就更改为1(init的进程ID).这种处理方法保证了每个进程有一个父进程。
  
  另一个我们关心的情况是如果子进程在父进程之前终止，那么父进程又如何能在做相应检查时得到子进程的终止状态呢? 对此问题的回答是内核为每个终止子进程保存了一定量的信息，所以当终止进程的父进程调用wait或waitpid时，可以得到有关信息。这种信息至少包括进程ID, 该进程的终止状态，以及该进程使用的CPU时间总量。内核可以释放终止进程所使用的所有存储器，关闭其所有打开文件。在Unix术语中，一个已经终止、但是其父进程尚未对其进行善后处理(获取终止子进程的有关信息、释放它仍占有的资源)的进程称为僵尸进程(zombie).ps(1)命令将僵死进程的状态打印为Z. 如果编写一个长期运行的程序，它fork了很多子进程，那么除非父进程等待取得子进程的终止状态，否则这些子进程就会变成僵死进程。
  
  最后需要考虑的一个问题是: 一个由init进程领养的进程终止时会发生什么?它会不会变成一个僵死进程?对此问题的回答是"否", 因为init被编写称只要有一个子进程终止，init就会调用一个wait函数取得其终止状态。这样也就防止了在系统中有很多僵死进程。当提及"一个init的子进程"时，这指的是init直接产生的进程(例如getty进程)，或者是其父进程已终止，由init领养的进程。
  
  

### 8.6 wait和waitpid函数
  当一个进程正常或异常终止时，内核就向其父进程发送SIGCHLD信号。因为子进程终止是个异步事件(这可以在父进程运行的任何时候发生)，所以这种信号也是内核向父进程发的异步通知。父进程可以忽略该信号，或者提供一个信号发生时即被调用执行的函数(信号处理程序)。对于这种信号的系统默认动作是忽略它。第10章将说明这些选择项。现在需要知道的是调用wait或waitpid的进程可能会:
  * 阻塞(如果其所有子进程都还在运行)
  * 带子进程的终止状态立即返回(如果一个子进程已终止，正等待父进程存取其终止状态)。
  * 出错立即返回(如果它没有任何子进程)。
  
  如果进程由于接到SIGCHLD信号而调用wait，则可期望wait会立即返回。但是如果在一个任一时刻调用wait, 则进程可能会阻塞。

```
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int *statloc, int options);
```
  这两个函数的区别是:
  * 在一个子进程终止前，wait使其调用者阻塞，而waitpid有一选择项，可使得调用者不阻塞。
  * waitpid并不等待第一个终止的子进程--它有若干个选择项，可以控制它所等待的进程。
  
  如果一个子进程已经终止，是一个僵死进程，则wait立即返回并取得该子进程的状态，否则wait使其调用者阻塞直到一个子进程终止。如调用者阻塞而且它有多个子进程，则在其中的一个子进程终止时，wait就立即返回。因为wait返回终止子进程的进程ID, 所以它总能了解是哪一个子进程终止了。
  这两个函数的参数statloc是一个整型指针。如果statloc不是一个空指针，则终止进程的终止状态就存放在它所指向的单元内。 如果不关心终止状态，则可将参数指定为空指针。
  依据传统，这两个函数返回的整型状态字是由实现定义的。其中某些位表示退出状态(正常返回)，其他位则指示信号编号(异常返回)，有一位指示是否产生了一个core文件等等。POSIX.1规定终止状态用定义在<sys/wait.h>中的各个宏来查看。有三个互斥的宏可用来取得进程终止的原因，它们的名字都以WIF开始。基于这三个宏中哪一个值是真，就可选用其他宏来取得终止状态、信号编号等。这些都在表8-1中给出。在8-9节讨论作业控制时，将说明如何停止一个进程。
  * WIFEXITED(status): 若为正常终止子进程返回的状态，则为真。对于这种情况可执行WEXITSTATUS(status)取子进程传递给exit或_exit参数的低8位。
  * WIFSIGNALED(status): 若为异常终止子进程返回的状态，则为真(接到一个不捕捉的信号)。对于这中情况可执行WTERMSIG(status)取得子进程终止的信号编号。另外SVR4和4.3+BSD(但是，非POSIX.1)定义宏:WCOREDUMP(status)，若已产生终止进程的core文件，则它返回真。
  * WIFSTOPPED(status): 若为当前暂停子进程的返回状态，则为真。对于这种情况，可执行WSTOPSIG(status)取使子进程暂停的信号编号。


```
/** prexit.c **/
#include "apue.h"
#include "prexit.h"

void
pr_exit(int status)
{
    if (WIFEXITED(status))
        printf("normal termination, exit status = %d\n",
                WEXITSTATUS(status));
    else if (WIFSIGNALED(status))
        printf("abnormal termination, signal number = %d%s\n",
                WTERMSIG(status),
#ifdef    WCOREDUMP
                WCOREDUMP(status) ? " (core file generated)" : "");
#else
                "");
#endif
    else if (WIFSTOPPED(status))
        printf("child stopped, signal number = %d\n",
                WSTOPSIG(status));
}

/** pstatus.c **/
#include <sys/types.h>
#include <sys/wait.h>

#include "apue.h"

int
main(void)
{
    pid_t pid;
    int status;

    if((pid = fork()) < 0)
        err_sys("fork error");
    else if(pid == 0) /** child process **/
        exit(7);

    if(wait(&status) != pid) /** wait for child **/
        err_sys("wait error");
    pr_exit(status);


    if((pid = fork()) < 0)
        err_sys("fork error");
    else if(pid == 0) /** child process **/
        abort();     /** generate SIGABRT **/


    if(wait(&status) != pid)
        err_sys("wait error");
    pr_exit(status);

    if((pid = fork()) < 0)
        err_sys("fork error");
    else if(pid == 0) /** child process **/
        status /= 0;     /** divide by 0 generate SIGFPE **/

    if(wait(&status) != pid) /** wait for child **/
        err_sys("wait error");
    pr_exit(status);

    exit(0);
}
```
  编译并运行结果如下:
```
bogon:process apple$ ./pstatus 
normal termination, exit status = 7
abnormal termination, signal number = 6
abnormal termination, signal number = 8
```
  不幸的是，没有一种可移植的方法将WTERMSIG得到的信号编号映射为说明性的名字。我们必须查看<signal.h>头文件才能直到SIGABRT的值是6，SIGFPE的值是8.
  正如前面所述，如果一个进程有几个子进程，那么只要有一个子进程终止，wait就返回。如果要等待一个指定的进程终止(如果直到要等待的进程的ID),那么该如何做呢? 在早期的Unix版本中，必须调用wait, 然后将其返回的进程ID和所要期望的进程ID进行比较。如果终止进程不是所期望的，则将该进程ID和终止状态保存起来，然后再次调用wait. 反复这样做直到所期望的进程终止。下一次又想等待一个特定进程时，先查看已终止的进程表，若其中已有要等待的进程，则取有关信息，否则调用wait. 其实，我们需要的是等待一个特定进程的函数。POSIX.1定义了waitpid函数以提取这种功能(以及其他的一些功能)。
  
  对于waitpid的pid参数的解释与其值有关:
  * pid == -1: 等待任一子进程。于是在这一方面waitpid与wait等效。
  * pid > 0: 等待其进程ID与pid相等的子进程。
  * pid == 0: 等待其组ID等于调用进程组ID的任一子进程。
  * pid < -1: 等待其组ID等于pid的绝对值的任一子进程。
  
  waitpid返回终止子进程的进程ID, 而该子进程的终止状态则通过statloc返回。 对于wait， 其唯一的出错是调用进程没有子进程(函数调用被一个信号中断时，也可能返回另一种出错。第10章进行讨论。)但是对于waitpid, 如果指定的进程或进程组不存在，或者调用进程没有子进程都能出错。

  options参数使我们能进一步控制waitpid的操作。此参数或者是0，或者是表8-2中常数的逐位或运算。
  * WNOHANG: 若由pid指定的子进程并不立即可用，则waitpid不阻塞，此时其返回值为0.
  * WUNTRACED: 若某实现支持作业控制，则由pid指定的任一进程状态已暂停，且其状态自暂停以来还未报告过，则返回其状态。WIFSTOPPED宏确定返回值是否对应一个暂停子进程。
```
SVR4支持两个附加的非标准的options常数。WNOWAIT使系统将其终止状态已由waitpid返回的进程保持在等待状态，于是该进程
就可被再次等待。对于WCONTINUED， 返回由pid指定的某一子进程的状态，该子进程已被继续，其状态尚未报告过。
```
  waitpid函数提供了wait函数没有提供的三个功能:
  1. waitpid等待一个特定的进程(而wait则返回任一终止子进程的状态)。在谈论popen函数时会再说明这一功能。
  2. waitpid提供了一个wait的非阻塞版本。有时希望取得一个子进程的状态，但不想阻塞。
  3. waitpid支持作业控制(以WUNTRACED选择项)
  
  实例:
  回忆下8.5节中关于僵死进程的讨论。如果一个进程要fork一个子进程，但不要求它等待子进程终止，也不希望子进程处于僵死状态直到父进程终止，实现这一点要求的诀窍是调用fork两次. 下面程序实现了这点。
  在第二个子进程中调用sleep以保证在打印父进程ID时第一个子进程已经终止。在fork之后，父子进程都可以继续执行--我们无法预知哪一个会先执行。如果不使第二个子进程睡眠，则在fork之后，它可能比其父进程先执行，于是它打印的父进程ID将是创建它的父进程，而不是init进程(进程ID 1).
```
#include <sys/types.h>
#include <sys/wait.h>

#include "apue.h"

int
main(void)
{
    pid_t pid;

    if((pid = fork()) < 0)
        err_sys("fork error");

    else if(pid == 0) {
        if((pid = fork()) < 0)
            err_sys("fork error");
        else if(pid > 0)
            exit(0);
        sleep(2);
        printf("second child, parent pid = %d\n", getppid());
        exit(0);
    }   

    if(waitpid(pid, NULL, 0) != pid)
        err_sys("waitpid error");

    exit(0);
}
```
  编译运行，执行结果如下:
```
bogon:process apple$ ./nozombie 
bogon:process apple$ second child, parent pid = 1
```
  注意，当原先的进程(也就是exec本程序的进程)终止时，shell打印其提示符，这在第二个子进程打印其父进程ID之前。
  
  

### 8.7 waitid函数
  这个单独的Unix规范包含一个额外的函数来检索进程退出状态。waitid函数类似于waitpid函数，但是提供了额外的灵活性。
```
#include <sys/wait.h>
int waitid(idtype_t idtype, id_t, siginfo_t *infop, int options);
  return: 0 if OK, -1 on error
```
  和waitpid类似，waitid允许进程指定要wait的是哪个子进程。它不是使用单个参数将进程ID和进程组ID编码在一个信息中，而是采用了两个参数。id参数是基于idtype解释的。 支持的idtype有如下几种:
  * P_PID: wait特定的进程id, 包含要wait的子进程的进程ID
  * P_PGID: wait特定进程组的任意子进程: id包含要wait的子进程的进程组ID
  * P_ALL: wait任意子进程: id参数被忽略掉。
  
  options参数是下列标志的位或结果。这些标志表明调用者感兴趣哪些状态改变。
  * WCONTINUED: 等待之前已经stopped并已经继续的进程，并且这些状态还没有报告过。
  * WEXITED: wait哪些已经exited的进程
  * WNOHANG: 如果没有子进程exit状态改变可用，立即返回，而不是阻塞。
  * WSTOPPED: wait已经stopped的进程，并且这些进程状态还没有报告过。
  
  至少需要在options参数里边提供WCONTINUED, WEXITED或WSTOPPED。
  infop参数是一个指向siginfo的结构。 这个结构包含关于信号产生导致子进程状态改变的详细信息。siginfo结构会在后面10.14中介绍。

### 8.8 wait3和wait4函数
  大多数Unix系统实现提供了另外两个函数:wait3, wait4. 由于历史原因，这两个变体是从BSD分支派生过来。这两个函数提供的功能比POSIX.1函数wait和waitpid所提供的分别要多一个，这与附加参数rusage有关。该参数要求内核返回由终止进程及其所有子进程使用的资源摘要。
```
#include <sys/types.h>
#include <sys/wait.h>
#include <sys/time.h>
#include <sys/resource.h>

pid_t wait3(int *statloc, int options, struct rusage *rusage);
pid_t wait4(pid_t pid, int *statloc, int options, struct rusage *rusage);
```
  资源信息包括用户CPU时间总量、系统CPU时间总量、缺页次数、接收到信号的次数等。有关细节可以参阅getrusage(2)手册页。这些资源信息只包括终止子进程，并不包括处于停止状态的子进程。

### 8.9 竞态条件
  从本书的目的除法，当多个进程都企图对共享数据进行某种处理，而最后的结果又取决于进程运行的顺序时，则我们认为这发生了竞态条件(race condition). 如果在fork之后的某种逻辑显示或隐式地依赖于在fork之后是父进程线运行还是子进程先运行，那么fork函数就会是竞态条件活跃地孽生地。 通常，我们不能预料哪个进程先运行。即使知道哪一个进程先运行，那么在该进程开始运行之后，所发生的事情也依赖于系统负载以及内核的调度算法。
  
  在程序8.5中，当第二个子进程打印其父进程进程id时，我们看到了一个潜在的竞态条件。如果第二个子进程在第一个子进程之前运行，则其父进程将会是第一个子进程。但是，如果第一个子进程先运行，并有足够的时间到达并执行exit,则第二个子进程的父进程就是init。 即使在程序中调用sleep， 这也不保证什么。如果系统负载很重，那么在第二个子进程从sleep返回时，可能第一个子进程还没有得到机会运行。这种形式的问题很难排除，因为在大部分时间，这种问题并不出现。
  如果一个进程希望等待一个子进程终止，则它必须调用wait函数。如果一个进程要等待其他父进程终止，则可以使用下列形式的循环:
```
while(getppid() != 1)
  sleep(1);
```
  这种形式的循环(称为定期询问polling)的问题是它浪费了CPU时间，因为调用者每隔1秒都被唤醒，然后进行条件测试。
  为了避免竞态条件和定期询问，在多个进程之间需要有某种形式的信号机制。在Unix中可以使用信号机制，在10.16节将说明它的一种用法。各种形式的进程间通信(IPC)也可使用。
  在父子进程的关系中，常常出现下述情况。在fork之后，父子进程都有一些事情要做。例如，父进程可能以子进程ID更新日志文件中的一个记录，而子进程则可能要为父进程创建一个文件。在本例中，要求每个进程在执行完它的一套初始化操作后要通知对方，并且在继续运行之前，要等待另一方面完成其初始化操作。这种情况可以描述如下:
```
#include "apue.h"
TELL_WAIT(); /** set things up for TELL_xxx & WAIT_xxx **/

if((pid = fork()) < 0)
  err_sys("fork error");
else if (pid == 0) { /** child **/
  /** child does whatever is necessary ... **/
  TELL_PARENT(getppid()); /** tell parent we're done **/
  WAIT_PARENT(); /** and wait for parent **/
  
  /** and the child continues on its way ... **/
  exit(0);
}

/** parent does whatever is neccessary ... **/
TELL_CHILD(pid);            /** tell child we're done **/
WAIT_CHILD();       /** and wait for child **/

/** and the parent continues on its way ... **/

exit(0);
```

  假设在头文件apue.h中定义了各个需要使用的变量。五个例程TELL_WAIT、TELL_PARENT、TELL_CHILD、WAIT_PARENT以及WAIT_CHILD，可以是宏，也可以是函数。
  
  在后面一些章节中，会说明实现这些TELL和WAIT例程的不同方法:10.16节中说明用信号的一种实现，程序14-3中说明用流管道的一种实现。下面先看一个使用这五个例程的实例。
```
#include <sys/types.h>
#include "apue.h"

static void charatatime(char *); 

int
main(void)
{
    pid_t pid;
    if((pid = fork()) < 0)
        err_sys("fork error");
    else if(pid == 0) {
        charatatime("output from child\n");
    } else {
        charatatime("output from parent\n");
    }   
    exit(0);
}

static void charatatime(char *str)
{
    char *ptr;
    int c;

    setbuf(stdout, NULL);    /* set unbuffered */
    for(ptr = str; c = *(ptr++); )
        putc(c, stdout);
}
```
  在程序中将标准输出设置为不带缓冲的，于是每个字符输出都需要调用一次write. 本例的目的是为了使内核尽可能多次地在两个进程之间进行切换，以例示静态条件。(如果不这样做，可能也就绝不会出现下面所示地输出。没有看到具有错误地输出并不意味着没有静态条件，这只是意味着在此特定地系统上未能见到它)。 我在macbook上看不到父子进程交错输出单个字符地情况，输出就略过了。
  
  修改程序8-6，使其使用TELL和WAIT函数，于是就形成了8-7.
  
```
#include <sys/types.h>
#include "apue.h"

static void charatatime(char *); 

int
main(void)
{
    pid_t pid;
    TELL_WAIT();

    if((pid = fork()) < 0)
        err_sys("fork error");
    else if(pid == 0) {
        WAIT_PARENT();
        charatatime("output from child\n");
    } else {
        charatatime("output from parent\n");
        TELL_CHILD(pid);
    }   
    exit(0);
}

static void charatatime(char *str)
{
    char *ptr;
    int c;

    setbuf(stdout, NULL);    /* set unbuffered */
    for(ptr = str; c = *(ptr++); )
        putc(c, stdout);
}
```
  关于此例后面再讨论。
  
  

### 8.10 exec函数
  8.3中曾提到fork函数创建子进程后，子进程往往会调用一种exec函数以执行另一个程序。当进程调用一种exec函数时，该进程完全由新进程代换，而新程序从其main开始执行。因为调用exec并不创建新进程，所以前后地进程ID并未改变。exec指示用另一个新程序替换了当前进程地正文、数据、堆和栈段。
  
  有六种不同地exec函数可供使用，它们常常被称为exec函数。这些exec函数都是Unix进程控制原语。用fork可以创建新进程，用exec可以执行新的程序。exit函数和两个wait函数处理终止和等待终止。 这些是我们需要的基本进程控制原语。在后面各节中将使用这些源于构造另外一些如popen和system之类的函数。
  
```
#include <unistd.h>
int execl(const char *pathname, const char *arg0, ... /* (char *)0 */ );
int execv(const char *pathname, char *const argv[]);
int execle(const char *pathname, const char *arg0, ... /* (char *)0, char *const envp[] */ );
int execve(const char *pathname, char *const argv[], char *const envp[]); 
int execlp(const char *filename, const char *arg0, ... /* (char *)0 */ ); 
int execvp(const char *filename, char *const argv[]);
int fexecve(int fd, char *const argv[], char *const envp[]);
                  All seven return: −1 on error, no return on success
```
  这些函数之前的第一个区别是前四个取路径名作为参数，接着两个则取文件名作为参数。当指定filename作为参数时:
  * 如果filename中包含/，则就将其视为路径名。
  * 否则就按PATH环境变量，在有关目录中搜索可执行文件。
  
  如果execlp和execvp中的任意一个使用路径前缀找到了一个可执行文件，但是该文件不是由连接编辑程序产生的机器可执行程序代码文件， 则就认为该文件是一个shell脚本，于是试着调用/bin/sh， 并以该filename作为shell的输入。

  第二个区别是参数表的传递有关(l表示列表(list), v表示矢量(vector). 函数execl和execlp和execle要求将新程序的每个命令行参数都说明为一个单独的参数。这种参数以空指针结尾。对另外三个函数(execv, execvp和execve)， 则应先构造一个指向各参数的指针数组，然后将该数组地址作为这三个函数的参数。
  
  最后一个区别就是，带e的，可以传递一个指向环境字符串指针数组的指针。
  
  具体的就不详细介绍， 前面有一篇关于exec的文章，可参照前面文章。
  
  
  前面曾提到在执行exec后，进程ID没有发生改变。除此之外，执行新程序的进程还保持了原进程的如下特征:
  * 进程ID和父进程ID
  * 实际用户ID和实际组ID
  * 添加组ID
  * 进程组ID
  * 对话期ID
  * 控制终端
  * 闹钟尚余留的时间
  * 当前工作目录
  * 根目录
  * 文件方式创建屏蔽字
  * 文件锁
  * 进程信号屏蔽
  * 未决信号
  * 资源限制
  * tms_utime, tms_stime, tms_cutime和tms_ustime值。
  
  对打开文件的处理与每个描述符的exec关闭标志值有关。进程中每个打开的描述符都有一个exec关闭标志。若此标志设置，则在执行exec时关闭该文件描述符，否则该描述符仍打开。除非特地用fcntl设置了该标志，否则系统的默认操作是在exec后仍保持这种描述符打开。

  POSIX.1明确要求在exec时关闭打开目录流。这通常是由opendir函数实现的，它调用fcntl函数为对应于打开目录流的描述符设置exec关闭标志。
  
  注意，在exec前后实际用户ID和实际组ID保持不变，而有效ID是否改变则取决于所执行程序的文件的set-user-ID和set-group-ID位是否设置。如果新程序的set-user-ID已设置，则有效用户ID变成程序文件的所有者的ID, 否则有效用户ID不变。对组ID的处理方式与此相同。
  
  很多Unix实现中，这六个函数中有一个execve是系统内核的系统调用。另外5个是库函数，它们最终都要调用系统调用。者六个函数的关系如下。 在这种安排中，库函数execlp和execvp使用PATH环境变量查找第一个包含名为filename的可执行文件的路径名前缀。
  
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/execs_relations.png)
```
#include <sys/types.h>
#include <sys/wait.h>

#include "apue.h"

char *env_init[] = {"USER=unknow", "PATH=/tmp", NULL};
int
main(void)
{
    pid_t pid;
    if((pid = fork()) < 0)
        err_sys("fork error");
    else if(pid == 0) {
        if(execle("/bin/ls", "ls", "-la", (char *)0, env_init) < 0)
            err_sys("execle error");
    }   

    if(waitpid(pid, NULL, 0) < 0)
        err_sys("wait error");

    if((pid = fork()) < 0)
        err_sys("fork error");
    else if(pid == 0) {
        if(execlp("ls", "ls", "-la", (char *)0) < 0)
            err_sys("execlp error");
    }   

    exit(0);
}
```
  在该程序中，先调用execle, 他要求一个路径名和一个特定的环境。下一个调用的是execlp他用一个文件名，并将调用者的环境传给新程序。execlp在这里我们能够工作的原因是因为目录/bin是PATH的一部分。注意，我们将第一个参数设置为路径名的文件名分量。某些shell将此参数设置为完整的路径名。
  
  

### 8.11 改变用户ID和组ID
  可以用setudi函数设置实际用户ID和有效用户ID. 与此类似，可以用setgid函数设置实际组ID和有效组ID.
```
#include <sys/types.h>
#include <unistd.h>

int setuid(uid_t uid);
int setgid(gid_t gid);
```
  关于谁能更改ID有若干规则。现在先考虑有关改变用户ID的规则(在这里关于用户ID所说明的一切都适用于组ID).
  1. 若进程具有超级用户特权，则setuid函数将实际用户ID、有效用户ID、以及保存的set-user-ID设置为uid.
  2. 若进程没有超级用户特权，但是uid等于实际用户ID或保存的set-user-ID, 则setuid直降有效用户ID设置为uid.不改变实际用户ID和保存的set-user-ID.
  3. 如果上述两个条件都不成立，则errno设置为EPERM, 并返回出错。
  在这里假定_POSIX_SAVED_IDS为真。如果没有提供这种功能，则上面所说的关于保存的set-user-ID部分都无效。

  关于内核所维护的三个用户ID,还需要注意下列几点:
  1. 只有超级用户进程可以更改实际用户ID. 通常，实际用户ID是在用户登录时，由login(1)程序设定的，而且决不会改变它。因为login是一个超级用户进程，当它调用setuid时，设置所有三个用户ID.
  2. 仅当对程序文件设置了set-user-ID位时，exec函数设置有效用户ID. 如果set-user-ID位没有设置，则exec函数不会改变有效用户ID，则将其维持为原先值。任何时候都可以调用setuid，将有效用户ID设置为实际用户ID或保存的set-user-ID. 自然，不能将有效用户ID设置为任一随机值。
  3. 保存的set-user-ID是由exec从有效用户ID复制的。在exec按文件用户ID设置了有效用户ID后，即进行这种复制，并将此副本保存起来。
  

### 8.12 解释器文件

### 8.13 system函数

### 8.14 进程会计

### 8.15 用户标识符

### 8.16 进程调度

### 8.17 进程时间

### 8.18 总结
