## 第14章 高级I/O

### 14.1 引言
  本章涵盖众多概念和函数，我们把它们统统都放到高级I/O下讨论:非阻塞I/O、记录锁、I/O多路转接(select和epoll函数)、异步I/O、readv和writev函数以及存储映射I/O(mmap)。第15章和第17章的进程间通信以及以后各章中的很多实例都要使用本章所秒数的概念和函数。

### 14.2 非阻塞I/O
  10.5节中曾将系统调用分成两类:低速系统调用和其他。低速系统调用是可能会使进程永远阻塞的一类系统调用:
  * 如某些文件类型(例如管道、终端设备和网络设备)的数据并不存在，读操作可能会使调用者永远阻塞。
  * 如果数据不能被相同的文件类型立即接收(例如管道中无空间、网络流控制)，写操作可能会使调用者永远阻塞。
  * 在某种条件发生之前打开某些文件类型可能会发生阻塞(如要打开一个终端设备，需要先等待与之连接的调制解调器应答，又如若以只写模式打开FIFO ,那么在没有其它进程已用读模式打开该FIFO时也要等待。)
  * 对已加上强制性记录锁的文件进行读写
  * 某些ioctl
  * 某些进程间通信函数
  
  我们也曾说过，虽然读写磁盘文件会暂时阻塞调用者，但并不能将与磁盘I/O有关的系统调用视为“低速”。
  
  非阻塞I/O使我们可以发出open, read, write这样的I/O操作，并使这些操作不会永远阻塞。如果这种操作不能完成，则调用立即出错返回，表示该操作如继续执行将阻塞。

  对于一个给定的描述符，有两种为其指定非阻塞I/O的方法。
  1. 如果调用open获得描述符，则可指定O_NONBLOCK标志。
  2. 对于已经打开的一个描述符，则可调用fcntl，由该函数打开O_NONBLOCK文件状态标志。
```
#include "apue.h"
#include <fcntl.h>

void set_fl(int fd, int flags)
{
  int val;
  
  if((val = fcntl(fd, F_GETFL, 0)) < 0)
    err_sys("fcntl F_GETFL error");
    
  val |= flags;
  
  if(fcntl(fd, F_SETFL, val) < 0)
    err_sys("fcntl F_SETFL error");
}

void clear_fl(int fd, int flags)
{
  int val;
  
  if((val = fcntl(fd, F_GETFL, 0)) < 0)
    err_sys("fcntl F_GETFL error");
    
  val &= flags;
  
  if(fcntl(fd, F_SETFL, val) < 0)
    err_sys("fcntl F_SETFL error");
}
```
  上面函数set_fl可以用来给一个描述符打开任一文件状态标志。
  
  实例:
  程序14-1是一个非阻塞I/O的实例，它从标准输入读500 000字节，并试图将它们写到标准输出上。该程序现将标准输出设置为非阻塞的，然后for循环进行输出，每次写的结果都在标准出错上打印。其中set_fl,clear_fl分别是为给定描述符设置或清除文件描述符特定标志位的两个自定义函数。
```
#include "apue.h"

#include <errno.h>
#include <fcntl.h>

char buf[500000];

int main(void)
{
    int ntowrite, nwrite;
    char *ptr;

    ntowrite = read(STDIN_FILENO, buf, sizeof(buf));
    fprintf(stderr, "read %d bytes\n", ntowrite);

    set_fl(STDOUT_FILENO, O_NONBLOCK); /** set nonblocking **/
    for(ptr = buf; ntowrite > 0;) {
        errno = 0;
        nwrite = write(STDOUT_FILENO, ptr, ntowrite);
        fprintf(stderr, "nwrite = %d, errno = %d\n", nwrite, errno);

        if(nwrite > 0) {
            ptr += nwrite;
            ntowrite -= nwrite;
        }   
    }   

    clear_fl(STDOUT_FILENO, O_NONBLOCK); /** clear nonblocking **/
    exit(0);
}
```
  若标准输出是普通文件，则可以期望write只执行一次。
```
bogon:advio apple$ ls -l /ect/services # 打印文件长度
-rw-r--r--  1 root  wheel  677972  9 10  2014 /etc/services

bogon:advio apple$ ./nonblock < /etc/services > temp.file 
read 500000 bytes
nwrite = 500000, errno = 0
```
  但是，若标准输出为终端，则期望write有时返回小于500 000的一个数字，有时返回错误。下面是运行的结果:
```
bogon:advio apple$ ./nonblock < /etc/services 2>stderr.out
这里产生大量输出

然后查看stderr.out文件内容:
bogon:advio apple$ cat stderr.out
bogon:advio apple$ cat stderr.out 
read 500000 bytes
nwrite = 999, errno = 0
nwrite = -1, errno = 35
nwrite = -1, errno = 35
nwrite = -1, errno = 35
nwrite = -1, errno = 35
nwrite = -1, errno = 35
...
nwrite = -1, errno = 35
nwrite = 1005, errno = 0
nwrite = 1005, errno = 0
nwrite = 1005, errno = 0
nwrite = 1008, errno = 0
nwrite = 349, errno = 0
```
  该系统上，errno35对应的是EAGAIN。终端驱动程序一次能接受的数量随系统而变化。具体结果还会因登录系统时所使用的方式不同而不同:在系统控制台上登录、在硬件接线终端上登录或用伪终端在网络上连接上登录。如果你在终端上运行一个窗口系统，那么也是经由伪终端设备与系统交互。
  
  在此实例中，程序发出了600多个write调用，但是只有400多个真正输出了数据，其余的都只返回了错误。这种形式的循环称为轮询。在多用户系统上它浪费了CPU时间。在14.4节我们将介绍非阻塞描述符的I/O多路转接，这是一种进行这种操作的更加有效的方法。
  
  有时，可以将应用程序设计称使用多线程的，从而避免使用非阻塞I/O。如若我们能在其他线程中继续进行，则可以允许单个线程在I/O调用中阻塞。这种方法有时能简化应用程序的设计，但是，线程间同步的开销有时却可能增加复杂性，于是导致得不偿失的后果。

### 14.3 记录锁
  当两个人同时编辑一个文件时，其后果将如何呢? 在很多Unix系统中，该文件的最后状态取决于写该文件的最后一个进程。但是对于有些应用程序，例如数据库，有时进程需要确保它正在单独写一个文件。为了向进程提供这种功能，商用Unix提供了记录锁机制。(第20章包含了使用记录锁的数据库函数库。)
  
  记录锁(read locking)的功能是: 当第一个进程正在读或修改文件的某个部分时，使用记录锁可以防止其他进程修改同一文件区。对于Unix，记录这个词是一种误用，因为Unix根本没有使用文件记录这种概念。一个更适合的术语可能是区域锁，因为它锁定的只是文件的一个区域(也可能是整个文件)。

#### 14.3.1 历史
  对早期Unix系统的其中一个批评是它们不能用来进行数据库系统，其原因是这些系统不支持对部分文件加锁。在Unix系统寻找进入商用计算机的途径时，很多系统开发小组已各种不同方式增加了对记录锁的支持。
  早期的伯克利版本只支持flock函数。该函数只能对整个文件加锁，不能对文件中的一部分加锁。
  SVR3通过fcntl函数增加了记录锁功能。在此基础上构造了lockf函数，它提供了一个简化的接口。这些函数允许调用者对一个文件中任意字节的区域加锁，长至整个文件，短至文件中的一个字节。
  POSIX.1标准的基础是fcntl方法。图14-2列出了各种系统提供的不同形式的记录锁。注意，Single Unix Specification在其XSI扩展中包括了lockf.
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/flock_support.png)
  本章最后将说明建议性锁和强制性锁的区别。本书只介绍POSIX.1的fcntl锁。
```
记录锁是1980年由John Bass最早加到V7上的。内核中相应的系统调用入口表 项是名为locking的函数。此函数提供了强制性记录锁功能,它被用在很多制造商的 系统III版本中。Xenix系统采用了此函数,SVR4在Xenix兼容库中仍旧支持该函数。
SVR2是系统V中第一个支持fcntl风格记录锁的版本( 1984年)。
```
#### 14.3.2 fcntl记录锁
  3.14节中已经给出了fcntl函数的原型。为了述说方便，这里再次重复一次:
```
#include <fcntl.h>
int fcntl(int fd, int cmd, .../* struct flock *flockptr */);
          返回值: 若成功，依赖于cmd, 否则返回-1
```
  对于记录锁，cmd是F_GETLK, F_SETLK或F_SETLKW. 第三个参数(我们将调用flockptr)是一个指向flock的结构指针。
```
struct flock{
  short l_type; /** F_RDLCK, F_WRLCK, Or F_UNLCK **/
  short l_whence; /** SEEK_SET, SEEK_CUR, or SEEK_END **/
  off_t l_start; /** offset in bytes, relative to l_whence **/
  off_t l_len; /** length, in bytes, o means lock to EOF; **/
  pid_t l_pid; /** returned with F_GETLK **/
}
```
  对于flock结构说明如下:
  * 所希望的锁类型:F_RDLCK(共享读锁)、F_WRLCK(独占性写锁)、F_UNLCK(解锁一个区域)
  * 要加锁或解锁的起始字节偏移量(l_start和l_whence).
  * 区域的字节长度(l_len)
  * 进程的ID(l_pid)持有的锁能阻塞当前进程(仅由F_GETLK返回)。
  
  关于加锁或解锁区域的说明还要注意下列几项规则。
  * 指定区域起始偏移量的两个元素与lseek函数中最后两个参数类似。l_whence可选用的值是SEEK_SET, SEEK_CUR, 或SEEK_END。
  * 锁可以在当前文件尾端处开始或者越过尾端处开始，但是不能在文件起始位置之前开始。
  * 如若l_len为0，则表示锁的范围可以扩展到最大可能偏移量。这意味着不管向该文件中追加写了多少数据，它们都可以处于锁的范围内(不必猜测会有多少字节被追加写到文件之后)，而且起始位置可以是文件中的任意一个位置。
  * 为了对整个文件加锁，我们设置l_start和l_whence指向文件的起始位置，并且指定长度l_len为0.(有多种方法可以指定文件起始处，但常用的方法是将l_start指定为0，l_whence指定为SEEK_SET).
  
  上面提到了两种类型的锁:共享读锁(l_type为L_RDCLK)和独占性写锁(l_type为L_WRCLK).基本规则是:任意多个进程在一个给定的字节上可以有一把共享的读锁，但是在一个给定字节上只能有一个进程有一把独占写锁。进一步而言，如果在一个给定字节上已经有一把或多把读锁，则不能在该字节上再加写锁；如果再一个自街上已经有一把独占性写锁，则不能再对它加任何读锁。在图14-3中示范了这些兼容性规则。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/flock_comp.png)
  
  上面说明的兼容性规则适用于不同进程提出的锁请求，并不适用于单个进程提出的多个锁请求。如果一个进程对一个文件区已经有了一把锁，后来该进程又企图在同一个文件区间再加一把锁，那么新锁将替换已有锁。因此，若一进程在某文件的16～32字节区间有一把写锁，然后又试图在16～32字节区间加一把读锁，那么该请求将成功执行，原来的写锁会被替换为读锁。

  加读锁时，该描述符必须是读打开。加写锁是，该描述符必须是写打开的。
  
  下面说明以下fcntl函数的3种命令。
  * F_GETLK: 决定由flockptr锁描述的锁是否被另外一把锁排斥(阻塞)。如果存在一把锁，它阻止创建由flockptr所描述的锁，则该现存的锁的信息将重写flockptr指向的信息。如果不存在这种情况，则除了将l_type设置为F_UNLCK之外，flockptr所指向结构中的其他信息保持不变。
  * F_SETLK: 设置由flockptr所描述的锁。如果我们试图获得一把读锁(l_type为F_RDLCK)或写锁(l_type为F_WRLCK)，而兼容性规则阻止系统给我们这把锁，那么fcntl会立即出错返回，此时errno设置为EACCES或EAGAIN. `虽然POSIX.1允许实现返回这两种出错代码中的任何一种，但本书说明的4中实现在锁请求不能得到满足时，都返回EAGAIN.`
  * F_SETLKW: 此命令也用来清除由flockptr指定的锁(l_type为F_UNLCK). 这个命令是F_SETLK的阻塞版本(命名中的W表示wait).如果所请求的读锁或写锁因另一个进程当前已经对所请求区域的某部分进行了加锁而不能被授予，那么调用进程会被设置为休眠。如果请求创建的锁已经可用，或者休眠由信号中断，则该进程被唤醒。
  应当了解，用F_GETLK测试能否建立一把锁，然后用F_SETLK或F_SETLKW企图建立那把锁，这两者不是一个原子操作。因此不能保证在这两次fcntl调用之间不会有另一进程插入并建立一把相同的锁。如果不希望在等待锁变为可用时产生阻塞，则必须处理由F_SETLK返回的可能的出错。


>注意，POSIX.1并没有说明在下列情况下将发生什么:一个进程在某个文件的一个区间上设置了一把读锁，第二个进程在试图对同一文件加一把写锁时阻塞，然后第三个进程则测试在同一文件区间上得到另一把锁。如果第三个进程只是因为读区间已经有一把读锁，而被允许在该区间防止另一把读锁，那么这种实现就可能会使希望加写锁的进程饿死。同时，当对同一区间加另一把读锁的请求到达时，提出加写锁而阻塞的进程需等待的时间延长了。如果加读锁的请求来得很频繁，使得该文件区间始终存在一把或几把读锁，那么预加写锁得进程就将等待很长时间。

  在设置或释放文件上的一把锁时，系统按要求组合或分裂相邻区。例如，若第100～199字节时加锁的区，需要解锁第150字节，则内核将维持两把锁，一把用于第100～149字节，另一把用于第151～199字节。下图说明了这种情况的字节范围锁。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/filetypes_lock_split.png)
  
  假定我们又对150字节加锁，那么系统将会再把三个相邻的加锁区合并成一个区(第100～199字节). 其结果如上图的上图，又跟之前的一样了。
  
  实例:请求和释放一把锁
> 为了避免每次分配flock结构，然后又填入各项信息，可用下面的函数lock_reg来处理所有细节。

```
  a
```
  

### 14.4 I/O复用

### 14.5 select和pselect函数

### 14.6 poll函数

### 14.7 异步I/O

### 14.8 System V异步I/O

### 14.9 BSD异步I/O

### 14.10 POSIX异步I/O

### 14.11 readv和writev函数

### 14.12 readn和writen函数

### 14.13 内存映射I/O

### 14.14 总结
