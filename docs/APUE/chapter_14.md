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
#include "apue.h"
#include <fcntl.h>

int
lock_reg(int fd, int cmd, int type, off_t offset, int whence, off_t len)
{
    struct flock lock;
    lock.l_type = type; /** F_RDLCK, F_WRLCK, F_UNLCK **/
    lock.l_start = offset; /** byte offset, relative to l_whence **/
    lock.l_whence = whence; /** SEEK_SET, SEEK_CUR, SEEK_END **/
    lock.l_len = len;   /** bytes (0 means to EOF) **/

    return (fcntl(fd, cmd, &lock));
}
```
  因为大多数锁调用是加锁或解锁一个文件区域(命令F_GETLK很少使用)，故而通常使用下面5个宏中的一个，这5个宏都定义到apue.h中。
```
#define read_lock(fd, offset, whence, len) \
    lock_reg((fd), F_SETLK, F_RDLCK, (offset), (whence), (len))

#define readw_lock(fd, offset, whence, len) \
    lock_reg((fd), F_SETLKW, F_RDLCK, (offset), (whence), (len))

#define write_lock(fd, offset, whence, len) \
    lock_reg((fd), F_SETLK, F_WRLCK, (offset), (whence), (len))

#define writew_lock(fd, offset, whence, len) \
    lock_reg((fd), F_SETLKW, F_WRLCK, (offset), (whence), (len))

#define un_lock(fd, offset, whence, len) \
    lock_reg((fd), F_SETLK, F_UNLCK, (offset), (whence), (len))
```
  我们有目的地用与lseek函数同样地顺序定义了这些宏中的前三个参数。
  
  实例: 测试一把锁
  下面定义了一个函数lock_test,用于测试一把锁。
```
pid_t
lock_test(int fd, int type, off_t offset, int whence, off_t len)
{
    struct flock lock;

    lock.l_type = type; /** F_RDLCK, F_WRLCK, F_UNLCK **/
    lock.l_start = offset; /** byte offset, relative to l_whence **/
    lock.l_whence = whence; /** SEEK_SET, SEEK_CUR, SEEK_END **/
    lock.l_len = len;   /** bytes (0 means to EOF) **/

    if(fcntl(fd, F_GETLK, &lock) < 0)
        err_sys("fcntl error");

    if(lock.l_type == F_UNLCK)
        return (0); /** false, region isn't locked by annother proc **/

    return (lock.l_pid); /** true, return pid of lock owner **/
}
```
  如果存在一把锁，它阻塞由参数指定的锁请求，则此函数返回持有这把现有锁的进程的进程ID, 否则此函数返回0。通常用下面两个宏来调用此函数。
```
#define is_read_lockable(fd, offset, whence, len) \
    (lock_test((fd), F_RDLCK, (offset), (whence), (len)) == 0)

#define is_write_lockable(fd, offset, whence, len) \
    (lock_test((fd), F_WRLCK, (offset), (whence), (len)) == 0)
```
  注意，进程不能使用lock_test函数测试它自己是否再文件的某一部分持有一把锁。F_GETLK命令的定义说明，返回信息指示是否有现有的锁阻止调用进程设置它自己的锁。因为F_SETLK和F_SETLKW命令总是替换调用进程现有的锁(若已存在)，所以调用进程绝不会阻塞再自己持有的锁上，于是，F_GETLK命令决不会报告调用进程自己持有的锁。

  实例:死锁
> 如果两个进程相互等待对方持有并且不释放(锁定)的资源时，则这两个进程就处于死锁状态。如果一个进程已经控制了文件中的一个加锁区域，然后它又试图对另一个进程控制的区域加锁，那么它就会休眠，在这种情况下，又发生死锁的可能性。 下面程序给出了一个死锁的例子。子进程对第0字节加锁，父进程对第1字节进行加锁。然后，它们中的每一个又试图对对方已经加锁的字节加锁。在该程序中使用了前面介绍的父子进程同步例程(TELL_xxx)和(WAIT_xxx),以便每个进程都能够等待另一个进程获得它设置的第一把锁。

```
#include "apue.h"
#include <fcntl.h>

static void
lockabyte(const char *name, int fd, off_t offset)
{
    if(writew_lock(fd, offset, SEEK_SET, 1) < 0)
        err_sys("%s: writew_lock error", name);

    printf("%s: got the lock, byte %lld\n", name, (long long)offset);
}


int
main(void)
{
    int fd; 
    pid_t pid;

    if((fd = creat("templock", FILE_MODE)) < 0)
        err_sys("creat error");

    if(write(fd, "ab", 2) != 2)
        err_sys("write error");

    TELL_WAIT();
    if((pid = fork()) < 0) {
        err_sys("fork error");
    } else if(pid == 0) { /** child process **/
        lockabyte("child", fd, 0); 
        TELL_PARENT(getppid());
        WAIT_PARENT();
        lockabyte("child", fd, 1); 
    } else {
        lockabyte("parent", fd, 1); 
        TELL_CHILD(pid);
        WAIT_CHILD();

        lockabyte("parent", fd, 0); 
    }   

    exit(0);
}
```
  编译并运行，结果如下，
```
bogon:advio apple$ ./deadlock 
parent: got the lock, byte 1
child: got the lock, byte 0
parent: writew_lock error: Resource deadlock avoided
child: got the lock, byte 1
```
  检测到死锁时，内核必须选择一个进程接受出错返回。在本例中，选择了父进程，但这是一个实现细节。在某些系统上，子进程总是接到出错信息，在另一些系统上，父进程总是接到出错信息。在某些系统上，当试图使用多把锁时，有时是子进程接到出错信息，有时是父进程接到出错信息。
  
#### 14.3.3 锁的隐含继承和释放
  关于记录锁的自动继承和释放有3条规则。
  1. 锁与进程、文件两者相关联。这有两重含义: 第一很明显，当一个进程终止时，它所建立的锁全部释放；第二重意思就不很明显，任何时候关闭一个描述符时，则该进程通过这一描述符可以存访的文件上的任何一把锁都被释放(这些锁都是该进程设置的)。这意味着，如果执行下列四步:`fd1 = open(pathname, ...);read_lock(fd1, ...); fd2 = dup(fd1); close(fd2);` 则在close(fd2)后，在fd1上设置的锁被释放。如果将dup代换为open,其效果也一样:`fd1 = open(pathname, ...); read_lock(fd1, ...); fd2 = open(pathname, ...); close(fd2);`
  2. 由fork产生的子进程不继承父进程所设置的锁。这意味着，若一个进程得到一把锁，然后调用fork, 那么对于父进程获得的锁而言，子进程视为另一个进程.对于通过fork从父进程处继承过来的描述符，子进程需要调用fcntl获得它自己的锁。这个约束时有道理的，因为锁的作用时阻止多个进程同时写一个文件。如果子进程通过fork继承父进程的锁，则父进程和子进程都可以同时写一个文件。
  3. 在执行exec后，新程序可以继承原执行程序的锁。但是注意，如果对一个文件描述符设置了执行时关闭标志，那么当作为exec的一部分关闭该文件描述符时，将释放相应文件的所有锁。
  
#### 14.3.4 FreeBSD实现
  先简要的观察FreeBSD实现中使用的数据结构。这会帮助我们进一步理解记录锁的自动继承和释放的第一条规则:锁与进程和文件两者相关联。
  考虑一个进程，它执行下列语句(忽略出错返回):
```
fd1 = open(pathname, ...);
write_lock(fd1, 0, SEEK_SET, 1); /* parent write locks byte 0 */
if(fork() > 0) { /* parent process */
  fd2 = dup(fd1);
  fd3 = open(pathname, ...);
  pause();
} else {
  read_lock(fd1, 1, SEEK_SET, 1); /* child read locks byte 1 */
  pause();
}
```
  下图显示了父进程和子进程暂停(执行pause())后的数据结构情况：
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/freebsd_lock_data_structure.png)
  
  前面已经给出了open、fork以及dup调用后数据结构(见图3-9和图8-2).有了记录锁后，在原来的这些图上新增了lockf结构，它们由i节点结构开始相互链接起来。每隔lockf结构描述了一个给定进程的一个加锁区域(由偏移量和长度定义的)。图中显示了两个lockf结构，一个是父进程调用write_lock形成的，另一个则是由子进程调用read_lock形成的。每一个结构都包含了相应的进程ID.
  
  实例: 下面程序中，我们了解到，守护进程可用一把文件锁来保证只有该守护进程的唯一副本在运行。
```
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>
#include <syslog.h>
#include <string.h>
#include <errno.h>
#include <stdio.h>
#include <sys/stat.h>

#define LOCKFILE "/var/run/deamon.pid"
#define LOCKMODE (S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH)

extern int lockfile(int);

int already_running(void)
{
    int fd; 
    char buf[16];

    fd = open(LOCKFILE, O_RDWR|O_CREAT, LOCKMODE);

    if(fd < 0) {
        syslog(LOG_ERR, "can't open %s: %s", LOCKFILE, strerror(errno));
        exit(1);
    }   

    if(lockfile(fd) < 0) {
        if(errno == EACCES || errno == EAGAIN) {
            close(fd);
            return (1);
        }   
        syslog(LOG_ERR, "can't open %s: %s", LOCKFILE, strerror(errno));
        exit(1);
    }   

    ftruncate(fd, 0); 
    sprintf(buf, "%ld", (long)getpid());
    write(fd, buf, strlen(buf) + 1); 
    return (0);
}

int
lockfile(int fd) 
{
    struct flock fl; 
    fl.l_type = F_WRLCK;
    fl.l_start = 0;
    fl.l_whence = SEEK_SET;
    fl.l_len = 0;
    return (fcntl(fd, F_SETLK, &fl));
}
```

  另一种方法是使用write_lock来定义lockfile:
  ```#define lockfile(fd) write_lock((fd), 0, SEEK_SET, 0)```.
  
#### 14.3.5 在文件尾端加锁
  在相对于文件尾端的字节范围加锁或解锁时需要特别小心。大多数实现按照l_whence的SEEK_CUR或SEEK_END值，用l_start以及文件当前位置或当前长度得到绝对文件偏移量。但是，常常需要相对文件的当前长度指定一把锁，但又不能用fstat来得到当前文件长度，因为我们在该文件上没有锁。(在fstat和锁调用之间，可能会有另一个进程改变该文件的长度。)
  考虑以下代码序列:
```
write_lock(fd, 0, SEEK_END, 0);
write(fd, buf, 1);
unlock(fd, 0, SEEK_END);
write(fd, buf, 1);
```
  该代码序列所做的可能并不是你期望的。它得到一把写锁，该写锁从当前文件尾端起，包括以后可能追加写到该文件的任何数据。假定，该文件偏移量处于文件尾端时，执行第一个write, 这个操作将文件延伸了1个字节，而该字节将被加锁。跟随其后的是解锁操作，其作用是对以后追加写到文件上的数据不再加锁。但在其之前刚追加的一个字节则保留加锁状态。当执行第二个写时，文件尾端又延伸了1个字节，但该字节并未加锁。由此代码序列造成的文件锁状态如下图所示:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/file_lock_at_eof.png)
  
  当对文件的一部分加锁，内核将指定的偏移量变换成绝对文件偏移量。另外，除了指定一个绝对偏移量(SEEK_SET)之外，fcntl还允许我们相对文件中的某个点指定该偏移量，这个点是指当前偏移量(SEEK_CUR)或文件尾端(SEEK_END).当前偏移量和文件尾端可能会不断变化，而这种变化又不应该影响现有锁的状态，所以内核必须独立于当前文件偏移量或文件尾端而记住锁。
  如果相解除锁中包括第一次write缩写的1个字节，那么应该指定长度为-1. 负的长度表示在指定偏移量之前的字节数。
  
#### 14.3.6 建议性锁和强制性锁
  考虑数据库访问例程库。如果该库中所有函数都以一致的方法处理记录锁，则称使用这些函数访问数据库的进程集为合作进程(cooperating process)。如果这些函数是唯一地用来访问数据库地函数，那么它们使用建议性锁是可行的。但是建议性锁并不能阻止对数据库文件有写权限的任何其他进程写这个数据库文件。不使用数据库访问例程协同一致的方法来访问数据库的进程是非合作进程。
  
  强制性锁会让内核检查每一个open, read和write，验证调用进程是否违背了正在访问的文件上的某一把锁。强制性锁有时也称为强迫方式锁(enforcement-mode locking).
> 从图14-2可以看到，Linux 3.2.0和Solaris 10提供强制性记录锁，而FreeBSD和Mac OS X 10.6.8则不提供。强制性记录锁不是Single Unix Specification的组成部分。在Linux中，如果用户想要使用强制性锁，则需要在各个文件系统基础上用mount命令-o mand选项来打开该机制。

  对一个特定文件打开起设置组ID位，关闭其组执行位便开启了对该文件的强制性锁机制. 因为当组执行位被关闭时，设置组ID位不再有意义，所以SVR3的设计者借用两者的这种组合来指定对一个文件的锁是强制性的而非建议性的。
  
  如果一个进程试图读(read)或写(write)一个强制性锁起作用的文件，而欲读写的部分又由其他进程加上了锁，此时会发生什么呢?对于这一问题的回答取决于3方面的因素: 操作类型(read或write)、其他进程持有锁的类型(读锁或写锁)以及read或write的描述符是阻塞的还是非阻塞的。下图列出了8种可能性:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/man_lock_relation_to_other_rw.png)
  
  除了上图的read和write函数，另一个进程持有的强制性锁也会对open函数产生影响。通常，即使正在打开的文件具有强制性锁，该open也会成功。随后的read或write依存于上图的规则。但是，如果欲打开的文件具有强制性记录锁(读锁或写锁)，而且open调用中的标志为O_TRUNC或O_CREAT，则不论是否指定O_NONBLOCK, open都立即出错返回，errno设置为EAGAIN.
  
> 只有Solaris对O_CREAT标志处理为出错。当打开一个具有强制性锁的文件时，Linux允许指定O_CREAT标志。对O_TRUNC标志产生open出错是有意义的，因为对于一个文件来讲，若另一个进程持有它的读锁或写锁，那么他就不能被截短为0.但是对O_CREAT标志在返回时设置出错就没有什么意义了，因为该标志表示，只有在该文件不存在时才创建，但由于另一个进程持有该文件的记录锁，所以该文件肯定是存在的。

  这种open的锁冲突处理方式可能会导致令人惊异的结果。在开发本节习题的时候，我们曾编写过一个测试程序，它打开一个文件(其模式指定为强制性锁)，对该文件整体设置一把读锁，然后休眠一段时间。(回忆上图读锁应当阻止其他进程写文件)在这段休眠时间内，用某些典型的Unix系统程序和操作符对该文件进行处理，发现下列情况:
  * 可用ed编辑器对该文件进行编辑操作，而且编辑结果可以协会磁盘！强制性记录锁根本不起作用。用某些Unix系统版本提供的系统调用跟踪特性，对ed操作进行跟踪分析发现，ed将内容写到一个临时文件中，然后删除原文件，最后将临时文件改名为原文件名。强制性锁机制对unlink函数没有影响，于是这一切发生了。
  * 不能用vi编辑器编辑该文件。vi可以读该文件内容，但是如果试图将新的数据写到该文件中，就会出错返回EAGAIN. 如果试图将新数据追加写到该文件中，则write阻塞。vi的这种行为与我们希望的一样。
  * 使用Korn shell的>和>>操作符重写或追加该文件，会产生出错信息"cannot create".
  * 在Bourne shell下使用>也会出错，但是使用>>操作符时只阻塞，在解除强制性锁后会继续处理。(这两种shell在执行追加写操作时之所以会产生差异，时因为Korn shell以O_CREAT和O_APPEND标志打开文件，而上面已提及O_CREAT会产生出错返回。但是Bourne shell在该文件已经存在时并不指定O_CREAT，所以open成功，而下一个write则阻塞。)
  
  产生的结果随所用操作系统版本不同而不同。从这样一个习题可中可见，在使用强制性锁时还需要警惕。从ed实例可以看出，强制性锁时可以设法避开的。

  一个恶意的用户可以使用强制性记录锁，对大家都可读的文件加一把写锁，这样就能阻止任何人写该文件(当然，该文件应当时强制性锁机制起作用的，这可能要求该用户能够更改该文件的权限位)。考虑一个数据库文件，它是大家都可读的，并且时强制性锁机制起作用的。如果一个恶意用户要对整个文件持有一把读锁，其他进程就不能再写该文件了。
  
  实例: 写一个程序确定系统是否支持强制性锁。
```
#include "apue.h"
#include <errno.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/wait.h>

int
main(int argc, char **argv)
{
    int fd;
    pid_t pid;
    char buf[15];
    struct stat statbuf;

    if(argc != 2) {
        fprintf(stderr, "usage: %s filename\n", argv[0]);
        exit(1);
    }   

    if((fd = open(argv[1], O_RDWR | O_CREAT | O_TRUNC, FILE_MODE)) < 0)
        err_sys("open error");

    if(write(fd, "abcdef", 6) != 6)
        err_sys("write error");

    /** turn on set-group-ID and turn off group-execute **/
    if(fstat(fd, &statbuf) < 0)
        err_sys("fstat error");

    if(fchmod(fd, (statbuf.st_mode & ~S_IXGRP) | S_ISGID) < 0)
        err_sys("fchmod error");

    TELL_WAIT();
    if((pid = fork()) < 0) {
        err_sys("fork error");
    } else if (pid > 0) { /** parent process **/
        /** write lock entire file **/
        if(write_lock(fd, 0, SEEK_SET, 0) < 0)
            err_sys("write_lock error");

        TELL_CHILD(pid);

        if(waitpid(pid, NULL, 0) < 0)
            err_sys("waitpid error");
    } else {
        WAIT_PARENT(); /** wait for parent to set lock **/
        set_fl(fd, O_NONBLOCK);

        /** first let's see what error we get if region is locked **/
        if(read_lock(fd, 0, SEEK_SET, 0) != -1) /** no wait **/
            err_sys("child: read_lock succeeded");
        printf("read_lock of already-locked region returns %d\n", errno);

        /** now try to read the mandatory locked file **/
        if(lseek(fd, 0, SEEK_SET) == -1)
            err_sys("lseek error");
        if(read(fd, buf, 2) < 0)
            err_ret("read failed (mandatory locking works)");
        else
            printf("read OK (no mandatory locking), buf = %2.2s\n", buf);
    }

    exit(0);
}
```

  此程序首先创建一个文件，并使强制性锁机制对其起作用。然后程序分出一个父进程和子进程。父进程对整个文件设置一把写锁，子进程则先将该文件描述符设置位非阻塞的，然后企图对该文件设置一把读锁，我们期望这会出错返回，并希望看到系统返回是EACCES或EAGAIN.接着子进程将文件读写位置调整到文件开头，并试图读该文件。如果系统提供强制性锁机制，则read应该返回EACCES或EAGAIN(因为该描述符是非阻塞的)，否则read返回所读的数据。在Solaris 10上运行，是可以看到如下结果(支持强制性锁机制)：
  
```
$ ./a.out temp.lock
read_lock of already-locked region returns 11
read failed (mandatory locking works): Resource temporarily unavailable
```

  查看系统头文件或intro(2)手册页，可以看到errno11对应于EAGAIN. 若在FreeBSD或mac上可以得到下面结果:
  
```
bogon:advio apple$ ./tmplock temp.lock
read_lock of already-locked region returns 35
read OK (no mandatory locking), buf = ab
```

  其中errno值为35对应于EAGAIN. 该系统不支持强制性锁。
  
  让我们回到本节的第一个问题:当两个人同时编辑同一个文件将会怎样呢?一般的UNIX文本编辑器并不使用记录锁,所以对此问题的回答仍然是:该文件的最后结果取决于写该文件的最后一个进程。
  (4.3+BSD的vi编辑器确实有一个编译选择项使运行时建议性记录锁起作用,但是这一选择项并不是默认可用的。)即使我们在一个编辑器,例如vi中使用了建议性锁,可是这把锁并不能阻止其他用户使用另一个没有使用建议性记录锁的编辑器。

  若系统提供强制性记录锁,那么可以修改常用的编辑器(如果有该编辑器的源代码)。如没有该编辑器的源代码,那么可以试一试下述方法。编写一个vi的前端程序。该程序立即调用fork,然后父进程等待子进程终止,子进程打开在命令行中指定的文件,使强制性锁起作用,对整个文件设置一把写锁,然后运行vi。在vi运行时,该文件是加了写锁的,所以其他用户不能修改它。当vi结束时,父进程从wait返回,此时自编的前端程序也就结束。
  本例中假定锁能跨越exec,这正是前面所说的SVR4的情况(SVR4是提供强制性锁的唯一系统)。这种类型的前端程序是可以编写的,但却往往不能起作用。问题出在大多数编辑器(至少是vi和ed)读它们的输入文件,然后关闭它。只要引用被编辑文件的描述符关闭了,那么加在该文件上的锁就被释放了。这意味着,在编辑器读了该文件的内容,然后关闭了它,那么锁也就不存在了。前端程序中没有任何方法可以阻止这一点。

### 14.4 I/O复用
  当从一个描述符读，然后又写到另一个描述符，可以在下列形式的循环中使用阻塞I/O:
```
while((n = read(STDIN_FILENO, buf, BUFFSIZ)) > 0)
  if(write(STDOUT_FILENO, buf, n) != n)
    err_sys("write error");
```

  这种形式的阻塞I/O随处可见。但是如果必须从两个描述符读，又将如何呢? 在这种情况下，我们不能在任一个描述符上进行阻塞读(read)，否则可能会因为被阻塞在一个描述符的读操作上导致另一个描述符即使有数据也无法处理。所以为了处理这种情况需要另一种不同的技术。
  让我们观察telnet(1)命令的结构。该程序从终端(标准输入)读，将所得数据写到网络连接上同时从网络连接读，将所得数据写到终端上(标准输出)。在网络连接的另一端，telnetd守护进程读用户键入的命令，并将所读到的送给shell,这如同用户登录到远程机器上一样。telnetd守护进程将执行用户键入命令而产生的输出通过telnet命令送回给用户，并显示在用户终端上。下图显示了这种工作情景:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/overview_telnet.png)
  
  telnet进程有两个输入，两个输出。我们不能对两个输入中的任何一个使用阻塞read,因为我们不知道到底哪一个输入会得到数据。
  
  处理这种特殊问题的一种方法是:将一个进程变成两个进程(用fork),每个进程处理一跳数据通路。下图显示了这种安排。System V的uucp通信包提供了cu(1)命令，其结构与此相似。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/telnet_with_2processes.png)
  
  如果使用两个进程，则可使每个进程都执行阻塞read. 但是这样也产生了问题: 操作什么时候终止?如果子进程接收到文件结束符(telnetd守护进程使网络连接断开)，那么该子进程终止，然后父进程收到SIGCHLD信号。但是如果父进程终止(用户在终端上键入了文件结束符)，那么父进程应通知子进程停止。为此可以使用一个信号(如SIGUSR1), 但这使得程序变得更加复杂。
  
  我们可以不使用两个进程，而是用一个进程中的两个线程。虽然这避免了终止的复杂性，但却要处理两个线程之间的同步，在复杂性方面这可能会得不偿失。
  
  另一个方法，仍旧是使用一个进程执行该程序，但使用非阻塞I/O读取数据。其基本思想是: 将两个输入描述符都设置为非阻塞的，对第一个描述符发一个read.如果该输入上有数据，则读数据并处理它。如果无数据可读，则该调用立即返回。然后对第二个描述符做同样的处理。在此之后，等待一定的时间(可能是若干秒)，然后再尝试从第一个描述符读。这种形式的循环称为轮询。这种方法的不足之处是浪费CPU时间。大多数时间实际上是无数据可读，因此执行read系统调用浪费了时间。在每次循环后要等待多长时间在执行下一轮循环也很难确定。虽然轮询技术在支持非阻塞I/O的所有系统上都可用，但是在多任务系统中应当避免使用这种方法。
  
  还有一种技术称为异步I/O(asynchronous I/O).利用这种技术，进程告诉内核: 当描述符准备好可以进行I/O时，用一个信号通知它。这种技术有两个问题: 首先，尽管一些系统提供了各自首先形式的异步I/O，但POSIX采纳了另外一套标准化接口，所以可移植性成为一个问题(以前POSIX异步I/O是Single Unix Specification中是可选设施，但现在，这些接口在SUSv4中是必须的)。System V提供了SIGPOLL信号来支持受限形式的异步I/O, 但是仅当描述符引用STREAMS设备时，此信号才起作用。BSD有一个类似的信号SIGIO, 但也有类似的限制: 仅当描述符引用终端设备或网络时它才能起作用。
  
  这种技术的第二个问题是，这种信号对每个进程而言只有一个(SIGPOLL或SIGIO)。如果使该信号对两个描述符都起作用(在我们讨论的实例中，从两个描述符读)，那么进程在接到此信号时将无法判断时哪一个描述符准备好了。尽管POSIX.1异步I/O接口允许选择哪个信号作为通知，但能用的信号数量仍远小于潜在的打开文件描述符的数量。为了确定时哪一个描述符准备好了，仍需将这两个描述符都设置为非阻塞的，并顺序尝试执行I/O. 我们将在14.5中讨论异步I/O.
  
  一个比较好的技术是使用I/O多路复用(I/O multiplexing). 为了使用这种技术，先构造一些我们感兴趣的描述符(通常不止一个)的列表，然后调用一个函数，直到这些描述符中的一个已准备好进行I/O时，该函数才返回。poll, pselect和select这3个函数使我们能够执行I/O多路转接。在从这些函数返回时，进程会被告知哪些描述符已经准备好进行I/O.

> POSIX指定，为了在程序中使用select, 必须包含<sys/select.h>. 但较老的系统还需要包括<sys/types.h>, <sys/time.h>和<unistd.h>. 查看select手册可以弄清楚你的系统都支持什么。
> I/O多路转接在4.2BSD中使用select函数提供的。虽然该函数主要用于终端I/O和网络I/O, 但它对其他描述符同样起作用的。SVR3在增加STREAMS机制时增加了poll函数。但在SVR4之前，poll只对STREAMS设备起作用。SVR4支持对任一描述符其作用的poll.

#### 14.4.1 select和pselect函数
  在所有POSIX兼容的平台上，select函数使我们可以执行I/O多路转接。传给select的参数告诉内核:
  * 我们关心的描述符
  * 对于每个描述符我们所关心的条件(是否想从一个给定的描述符读，是否想写一个给定的描述符，是否关心一个给定描述符的异常条件)
  * 愿意等待多长时间(可以永远等待、等待一个固定的时间或者根本不等待)
  
  从select返回时，内核告诉我们:
  * 已经准备好的描述符总数量
  * 对于读写或异常这三个条件中的每一个，哪些描述符已准备好。
  
  使用这种返回信息，就可调用相应的I/O函数(一般是read或write), 并且却只该函数不会阻塞。
```
#include <sys/select.h>
int select(int maxfdp1, fd_set *restrict readfds,
fd_set *restrict writefds, fd_set *restrict exceptfds, struct timeval *restrict tvptr);
              Returns: count of ready descriptors, 0 on timeout, −1 on error
```
  先来说明最后一个参数，它指定愿意等待的时间长度，单位为秒和微秒。有以下三种情况:
  * tvptr == NULL: 永远等待。如果捕捉到一个信号则终端此无限期等待。当所指定的描述符中的一个已准备好或捕捉到一个信号则返回。如果捕捉到一个信号，则select返回-1, errno设置为EINTR.
  * tvptr->tv_sec == 0 && tvptr->tv_usec == 0: 根本不等待。测试所有指定的描述符并立即返回。这时轮询系统找到多个描述符状态而不阻塞select函数的方法。
  * tvptr->tv_sec != 0 || tvptr->tv_usec != 0: 等待指定的秒数或微秒数。当指定的描述符之一已准备好，或当指定的时间值已经超过时立即返回。如果在超过到期时还没有一个描述符准备好，则返回值为0.(如果系统不提供微秒级的精度，则tvptr->tv_usec取汁到最近的支持值。)与第一种情况一样，这种等待可被捕捉到的信号中断。

> POSIX.1允许实现修改timeval结构中的值，所以在select返回后，你不能指望该结构仍旧保持调用select之前它所包含的值。FreeBSD8.0, Mac OS X 10.6.8和Solaris 10都保持该结构中的值不变。但是，若在超过时间尚未到期时，select就返回，那么Linux3.2.0将用剩余时间值更新该结构。

  中间3个参数readfds, writefds, exceptfds是指向描述符集的指针。这3个描述符集说明了我们关心的可读、可写或处于异常条件的描述符集合。每个描述符存储在一个fd_set数据类型中。这个数据类型是由实现选择的，它可以为每一个可能的描述符保持一位。我们可以认为它只是一个很大的字节数组，如下图:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/multiplexing_select_fds.png)
  
  对于fd_set数据类型，唯一可以进行的处理是: 分配一个这中类型的变量，将这种类型的一个变量值赋给同样类型的另一个变量，或对这种类型的变量使用下列4个函数中的一个。
```
#include <sys/select.h>
int FD_ISSET(int fd, fd_set *fdset);
void FD_CLR(int fd, fd_set *fdset);
void FD_SET(int fd, fd_set *fdset);
void FD_ZERO(fd_set *fdset);
```
  这些接口可以实现宏或函数。调用FD_ZERO将一个fd_set变量的所有位设置为0. 要开启描述符集中的一位，可以调用FD_SET. 调用FD_CLR可以清除一位。最后可以调用FD_ISSET测试描述符集合中的一个指定位是否打开。
  
  在声明了一个描述符集合后，必须用FD_ZERO将这个描述符集置为0，然后在其中设置关心的各个描述符的位。具体操作如下:
```
fdset rset;
int fd;
FD_ZERO(rset);
FD_SET(fd, &rset);
FD_SET(STDIN_FILENO, &rset);
```
  从select返回时，可以用FD_ISSET测试该集中的一个给定位是否仍处于打开状态:
```
if(FD_ISSET(fd, &rset)) {
  ...
}
```
  select的中间三个参数中的任一个可以是空集，这表示对相应条件并不关心。如果所有3个指针都是NULL, 则select提供了比sleep更精准的定时器。(回忆一下10.19， sleep等待整数秒，而select的等待时间则可以小于1秒，其实际精度取决于系统时钟。)
  
  select的第一个参数maxfdp1的意思是最大文件描述符编号加1. 考虑所有3个描述符集，在3个描述符集中找出最大描述符编号值，然后加1，这就是第一个参数值。也可将第一个参数设置为FD_SETSIZE, 这是<sys/select.h>中的一个常数，它指定最大描述符(经常是1024)，但是绝大多数应用程序而言，此值太大了。确实，大多数应用程序只使用3-10个描述符(某些应用程序需要更多的描述符，但这种Unix程序并不典型)。通过指定我们所关注的最大描述符，内核就只需要在此范围内寻找打开的位，而不比在3个描述符集中的数百个没有使用的位内搜索。
  
  例如，下图所示的描述符集情况就好像是执行了下述操作:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/select_sample_fdsets.png)
```
fdset readset, writeset;
FD_ZERO(&readset);
FD_ZERO(&writeset);

FD_SET(0, &readset);
FD_SET(3, &readset);

FD_SET(1, &writeset);
FD_SET(2, &writeset);

select(4, &readset, &writeset, NULL, NULL);
```
  因为描述符从0开始，所以要在最大描述符编号值上加1.第一个参数实际上是要检查的描述符数(从描述符0开始)。
  select有3个可能的返回值:
  1. 返回值-1表示出错。 这是可能发生的，例如，在所指定的描述符一个都没准备好时捕捉到一个信号。在此种情况下，一个描述符集都不修改。
  2. 返回值0表示没有描述符准备好。若指定的描述符一个都没准备好，指定的时间就过了，那么就会发生这种情况。此时，所有描述符集合都被置0.
  3. 一个正返回值说明了已经准备好的描述符数。该值是3个描述符集中已经准备好的描述符数之和，所以如果同一描述符已经准备好读写，那么在返回值中会对其计数两次。这这种情况下，3个描述符集中仍旧打开的位对应于已经准备好的描述符。
  
  对于准备好的含义需要作进一步具体的说明:
  * 若对读集(readfds)中的一个描述符进行的read操作不会阻塞，则认为此描述符是准备好的。
  * 若对写集(writefds)中的一个描述符进行的write操作不会阻塞，则认为此描述符是准备好的。
  * 若对异常条件集(exceptfds)中的一个描述符有一个未决异常条件，则认为此描述符是准备好的。现在，异常条件包括: 在网络连接上到达带外的数据，或者在处于数据包模式的伪终端上发生了某些条件.
  * 对于读写和异常条件，普通文件的文件描述符总是返回准备好。
  
  一个描述符阻塞与否并不影响select是否阻塞，理解这一点很重要。也就是说，如果希望读一个非阻塞描述符，并且以超时值位5秒调用select，则select最多阻塞5秒。相类似，如果指定一个无限的超时值，则在该描述符数据准备好，或捕捉到一个信号之前，select会一直阻塞。

  如果在一个描述符上碰到了文件尾端，则select会认为该描述符是可读的。然后调用read，它返回0， 这是Unix系统只是到达文件尾端的方法。(很多人错误的认为，当到达文件尾端时，select也会只是一个异常条件)。
  
  POSIX.1也定义了一个select的变体，称为pselect.
```
#include <sys/select.h>
int pselect(int maxfdp1, fd_set *restrict readfds,
fd_set *restrict writefds, fd_set *restrict exceptfds, const struct timespec *restrict tsptr,
const sigset_t *restrict sigmask);

  返回值: 准备就绪的描述符数目；若超时，返回0；若出错，返回－1.
```
  除下列几点，pselect和select相同。
  * select的超时值用timeval结构指定，但pselect使用timespec结构。timespec结构以秒和纳秒表示超时值，而非秒和微秒。如果平台支持这样的时间精度，那么timespec就能提供更精准的超时时间。
  * pselect的超时值被声明为const, 这保证了调用pselect不会改变此值
  * pselect可使用可选信号屏蔽字。若sigmask为NULL，那么在与信号有关的方面，pselect的运行状况和select相同。否则，sigmask指向一个信号屏蔽字，在调用pselect时，以原子操作的方式安装该信号屏蔽字。在返回时，恢复以前的信号屏蔽字。
  
  
#### 14.4.2 poll函数
  poll函数类似于select,但是程序员接口有所不同。虽然poll函数是System V引入来支持STRAEMS子系统的，但是poll函数可用于任何类型的文件描述符。
```
#include <poll.h>
int poll(struct pollfd fdarray[], nfds_t nfds, int timeout);
    Returns: count of ready descriptors, 0 on timeout, −1 on error
```
  与select不同，poll不是为每个条件(可读、可写和异常条件)构造一个描述符集。而是构造一个pollfd结构的数组，每个数据元素指定一个描述符编号以及我们对该描述符感兴趣的条件。
```
struct pollfd{
  int fd; /** 要检查的文件描述符，或者使用<0的值忽略要检查的描述符 **/
  short events; /** 感兴趣的fd的事件集合 **/
  short revents; /** 发生在fd上面的事件集合 **/
}
```
  fdarray数组中的元素由nfds指定。
> 由于历史原因，在如何声明nfds参数方面有几种不同的方式。SVR3将nfds的类型指定为unsigned long, 这似乎太大了。在SVR4手册中, poll原型的第二个参数数据类型为size_t. 但在<poll.h>包含的实际原型中，第二个参数的数据类型仍指定为unsigned long. Single Unix Specification定义了新类型nfds_t, 该类型允许实现选择对其合适的类型并且隐藏了应用细节。注意，因为返回值表示数组中满足事件的项数，所该中类型必须大得足以保存一个整数。

> 对应于SVR4的SVID上显示，poll的第一个参数是struct pollfd fdarray[], 而SVR4手册页上则显示该参数为struct pollfd *fdarray. 在C语言中，这两种声明是等价的。我们使用第一种声明是为了重申fdarray指向的是一个结构数组，而不是指向单个结构的指针。

  应将每个数组元素的events成员设置为图14-17中所示值的一个或几个，通过这些值告诉内核我们关心的是每个描述符的哪些事件。返回时，revents成员由内核设置，用于说明每个描述符发生了哪些事件。(注意，poll没有更改events成员。这与select不同，select修改其canshu以指示哪一个描述符已经准备好了。)
  
  图14-17中的前4行测试的是可读性，接下来的3行测试的是可写性，最后3行测试的是异常条件。最后3行是由内核在返回时设置的。即使在events字段中没有指定这3个值，如果相应条件发生，在revents中也会返回它们。
  
> 有些poll事件名字中包含BAND, 它指的是STREAMS当中优先级波段。想要了解关于STREAMS和优先级波段的更多信息，可以查看Rago[1993].
  
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/poll_events_revents_flags.png)

  当一个描述符被挂断(POLLHUP)后，就不能再写该描述符，但是有可能仍然可以从该描述符读取到数据。
  poll的最后一个参数指定的是我们愿意等待多长时间。如同select一样，有3种不同的情形:
  * timeout == -1: 永远等待。
  * timeout == 0: 不等待
  * timeout > 0: 等待timeout毫秒。
  
  理解文件尾端与挂断之间的区别是很重要的。如果我们正从终端输入数据，并键入文件结束符，那么就会打开POLLIN,于是我们就可以读文件结束指示(read返回0). revents中的POLLHUP没有打开。如果正在读调制解调器，并且电话线已经挂断，我们将接到POLLHUP通知。
  与select一样，一个描述符是否阻塞不会影响poll是否阻塞。

  select和poll的可中断性
  中断的系统调用的自动重启是由4.2BSD引入的，但当时select函数是不重启的。这种特性在大多数系统中一直延续下来，即使指定了SA_RESTART选项也是如此。但是在SVR4上，如果指定了SA_RESTART，那么select和poll也是自动重启的。为了将软件移植到SVR4派生的系统上时阻止了之一点，如果信号有可能会中断select或poll, 就要使用signal_intr函数。

### 14.5 异步I/O
  使用上一节说明的select和poll可以实现异步形式的通知。关于描述符的状态，系统并不主动告诉我们任何信息，我们需要进行查询(调用select或poll). 比如在第10章，信号机构提供了一种以异步形式通知某种事件发生的方法。由BSD和System V派生的所有系统都提供了某种形式的异步I/O,使用一个信号(在System V中是SIGPOLL，在BSD中是SIGIO)通知进程，对某个描述符所关心的某个事件已经发生。我们在前面的章节中提到过，这些形式的异步I/O是受限制的: 它们并不能用在所有的文件类型上，而且只能使用一个信号。如果要对一个以上的描述符进行异步I/O, 那么在进程接收到该信号时并不知道这一信号对应于哪一个描述符。
  
  SUSv4中将通用的异步I/O机制从实时扩展部分调整到基本规范部分。这种机制解决了额这些陈旧的异步I/O设施存在的局限性。
  
  在我们了解使用异步I/O的不同方法之前，需要先讨论一下成本。在用异步I/O的时候，要通过选择来灵活处理多个兵法操作，这回使应用程序的设计复杂化。更简单的做法可能时使用多线程，使用同步模型来编写程序，并让这些线程以异步的方式运行。
  使用POSIX异步I/O接口，会带来下列麻烦。
  * 每个异步操作有3处可能产生错误的地方:一处在操作提交的部分，一处操作本身的结果，还有一处在用于决定异步操作状态的函数中。
  * 与POSIX异步I/O接口传统方法相比，它们本身涉及大量的额外设置和处理规则。
  * 从错误中恢复可能会比较困难。举例来说，如果提交了多个异步写操作，其中一个失败了，下一步我们应该怎么做? 如果这些写操作是相关的，那么可能还需要撤销所有成功的写操作。
  

#### 14.5.1 System V异步I/O
  在System V中，异步I/O是STREAMS系统的一部分，它只对STREAMS设备和STREAMS管道起作用。System V的异步I/O信号是SIGPOLL.
  
  为了对一个STREAMS设备启动异步I/O,需要调用ioctl, 将它的第二个参数(request)设置成I_SETSIG.第三个参数是由14-18中的一个或多个常量构成的整型值。这些常量是在<stropts.h>中定义的。
  除了调用ioctl指定产生SIGPOLL信号的条件以外，还应为该信号建立信号处理程序。SIGPOLL默认动作是终止该进程，所以应当在调用ioctl之前建立信号处理函数。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/sigpoll_conds.png)
  
  
#### 14.5.2 BSD异步I/O
  在BSD派生的系统中，异步I/O是信号SIGIO和SIGURG的组合。SIGIO是通用异步I/O信号，SIGURG则只用来通知进程网络连接上的带外数据已经到达。
  为了接收SIGIO信号，需要执行以下3步:
  1. 调用signal或sigaction为SIGIO信号建立信号处理函数。
  2. 以命令F_SETOWN调用fcntl来设置进程ID或进程组ID,用于接收对于该描述符的信号。
  3. 以命令F_SETFL调用fcntl设置O_ASYNC文件状态标志，使在该描述符上可以进行异步I/O.
  
  第三步仅能对指向终端或网络的描述符执行，这是BSD异步I/O设施的一个基本限制。

  对于SIGURG信号，只需要执行1，2步。该信号金队应用支持带外数据的网络连接描述符而产生，如TCP连接。

#### 14.5.3 POSIX异步I/O
  POSIX异步I/O接口为对不同类型的文件进行异步I/O提供了一套一致的方法。这些接口来自实时草案标准，该标准是由Single Unix Specification的可选项。在SUSv4中，这些接口被一直到了基本部分中，所以现在所有平台都被要求支持这些接口。
  
  这些异步I/O接口使用AIO控制块来描述I/O操作。aiocb结构定义了AIO控制块。该结构至少包括下面这些字段(具体的实现可能还包含有额外的字段)
```
/** asynchronous I/O control block **/
struct aiocb{
  int             aio_fildes; /** 文件描述符 **/
  int             aio_offset; /** 文件I/O偏移量 **/
  volatile void  *aio_buf; /** I/O缓存 **/
  size_t          aio_nbytes; /** 要传输的字节数 **/
  int             aio_reqprio; /** 优先级 **/
  struct sigevent aio_sigevent; /** 信号函数 **/
  int             aio_lio_opcode; /** I/O列表的操作 **/
}
```
  aio_fildes字段表示被打开用来读或写的文件描述符。读或写操作从aio_offset指向的偏移量开始。对于读操作，数据会复制到缓冲区中，该缓冲区从aio_buf指定的地址开始。对于写操作，数据会从这个缓冲区中复制出来。aio_nbytes字段包含了要读或写的字节数。
  
  注意，异步I/O操作必须显示的指定偏移量。异步I/O接口并不影响由操作系统维护的文件偏移量。只要不再同一个进程里把异步I/O函数和传统I/O函数混在一起用在同一个文件上，就不会导致什么问题。同时值得注意的是，如果使用异步I/O接口向一个以追加模式(使用O_APPEND)打开的文件中写入数据，AIO控制块中的aio_offset字段会被系统忽略。
  
  其他字段和传统I/O函数中的不一致。应用程序使用aio_reqprio字段为异步I/O请求提示顺序。然而，系统对于该顺序只有有限的控制能力，因此不一定能遵循该提示。aio_lio_opcode字段只能用于基于列表的异步I/O，我们在稍后再讨论它。aio_sigevent字段控制，在I/O条件完成后，如何通知应用程序。这个字段通过sigevent结构来描述。
```
struct sigevent{
  int               sigev_notify; /** 通知类型 **/
  int               sigev_signo; /** 信号编号 **/
  union sigval      sigev_value; /** 通知参数 **/
  void (*sigev_notify_function)(union sigval); /** 通知函数 **/
  pthread_attr_t *sigev_notify_attributes; /** 通知属性 **/
}
```
  sigev_notify: 字段控制通知的类型。取值可能是以下3个中的一个:
  * SIG_NONE: 异步I/O请求完成后，不通知进程
  * SIG_SIGNAL: 异步I/O请求完成后，产生由sigev_signo字段指定的信号。如果应用程序已选择捕捉信号，且在建立信号处理程序的时候指定了SA_SIGINFO标志，那么该信号将被入列(如过实现支持排队信号)。信号处理程序会传送给一个siginfo结构，该结构的si_value字段被设置为sigev_value(如果使用了SA_SIGINFO标志).
  * SIGEV_THREAD: 当异步I/O完成时，由sigev_notify_function字段指定的函数被调用。sigev_value字段被传入作为它的唯一参数。除非sigev_notify_attributes字段被设定为pthread属性结构的地址，且该结构指定了一个另外的线程属性，否则该函数将在分离状态下的一个单独线程中执行。
  
  在进行异步I/O之前需要先初始化AIO控制块，调用aio_read函数来进行异步读操作，或调用aio_write函数来进行异步写操作。
```
#include <aio.h>
int aio_read(struct aiocb *aiocb); 
int aio_write(struct aiocb *aiocb);
    Both return: 0 if OK, −1 on error
```

  当这些函数返回成功时，异步I/O请求便已经被操作系统放入等待处理的队列中了。这些返回值与实际I/O结果没有任何关系。I/O操作在等待时，必须注意确保AIO控制块和数据库缓冲区保持稳定；它们下面对应的内存必须始终是合法的，除非I/O操作完成，否则不能被复用。
  要想强制所有等待中的异步操作不等待而写入持久化的存储中，可以设立一个AIO控制块并调用aio_fsync函数。
```
#include <aio.h>
int aio_fsync(int op, struct aiocb *aiocb);
```
  AIO控制块中的aio_fildes字段制定了其异步写操作被同步的文件。如果op参数设定为O_DSYNC,那么操作执行起来就会像调用了fdatasync一样。否则，如果op参数设定为O_SYNC， 那么操作执行起来就会像调用了fsync一样。
  
  像aio_read和aio_write函数一样，在安排了同步时，aio_fsync操作返回。在异步同步操作完成之前，数据不会被持久化。AIO控制块控制我们如何被通知，就像aio_read和aio_write函数一样。
  
  为了获知一个异步读、写或者同步操作的完成状态，需要调用aio_error函数。
```
#include <aio.h>
int aio_error(const struct aiocb *aiocb);
```
  返回值为下面4种情况中的一种:
  * 0: 异步操作成功完成。需要调用aio_return函数获取操作返回值。
  * -1: 对aio_error的调用失败。这种情况下，errno会告诉我们为什么。
  * EINPROGRESS: 异步读、写或同步操作仍在等待。
  * 其他情况: 其他任何返回值时相关的异步操作失败返回的错误码。
  
  如果异步操作成功，可以调用aio_return函数来获取异步操作的返回值。
```
ssize_t aio_return(const struct aiocb *aiocb);
```
  直到异步操作完成之前，都需要小心不要调用aio_return函数。操作完成之前的结果是未定义的。还需要小心对每个异步操作只调用一次aio_return. 一旦调用了该函数，操作系统就可以释放掉包含了I/O操作返回值的记录。
  
  如果aio_return函数本身失败，会返回-1, 并设置errno. 其他情况下，它将返回异步操作的结果，即会返回read, write或者fsync在被成功调用时可能返回的结果。
  
  执行I/O操作时，如果还有其他事务需要处理而不想被I/O操作阻塞，就可以使用异步I/O.然而，如果在完成了所有事务时，还有异步操作未完成时，可以调用aio_suspend函数来阻塞进程，直到操作完成。
  
```
#include <aio.h>
int aio_suspend(const struct aiocb *const list[], int nent,
const struct timespec *timeout);
        Returns: 0 if OK, −1 on error
```
  aio_suspend可能会返回三种情况中的一种。如果我们被一个信号中断，它将会返回-1，并将errno设置为EINTR. 如果在没有任何I/O操作完成的情况下，阻塞的事件超过了函数中可选的timeout参数所指定的时间限制，那么aio_suspend将返回-1, 并将errno设置为EAGAIN(不想设置任何时间限制的话，可以把空指针传给timeout参数)。如果有任何I/O操作完成，aio_suspend将返回0.如果在我们调用aio_suspend操作时，所有的异步I/O操作都已完成，那么aio_suspend将在不阻塞的情况下直接返回。
  
  list参数时一个指向AIO控制块数组的指针，nent参数表明了数组中的条目数。数组中的空指针会被跳过，其他条目都必须指向已用于初始化异步I/O操作的AIO控制块。
  
  当还有我们不想再完成的等待中的异步I/O操作时，我们可以尝试使用aio_cancel函数来取消它们。
```
#include <aio.h>
int aio_cancel(int fd, struct aiocb *aiocb);
```
  fd参数制定了哪个未完成的异步I/O操作的文件描述符。如果aiocb参数为NULL，系统将会尝试取消所有该文件上未完成的异步I/O操作。其他情况下，系统将尝试取消由AIO控制块描述的单个异步I/O操作。我们之所说系统“尝试”取消操作，是因为无法保证系统能够取消正在进程中的任何操作。
  aio_cancel函数可能会返回以下4个值中的一个:
  * AIO_ALLDONE: 所有操作在尝试取消它们之前已经完成。
  * AIO_CANCELED: 所有要求的操作已被取消
  * AIO_NOTCANCELED: 至少由一个要求的操作没有被取消
  * -1: 对aio_cancel的调用失败，错误码将被存储在errno中。
  
  如果异步I/O操作被成功取消，对相应的AIO控制块调用aio_error函数将会返回错误ECANCELED. 如果操作不能被取消，那么相应的AIO控制块不会因为对aio_cancel的调用而被修改。

  还有一个函数也被包含在异步I/O接口当中，尽管它既能以同步方式来使用，又能以异步方式来使用，这个函数就是lio_listio.该函数提交一系列由一个AIO控制块列表描述的I/O请求。
```
#include <aio.h>
int lio_listio(int mode, struct aiocb *restrict const list[restrict], int nent, struct sigevent *restrict sigev);
```
  mode参数决定了I/O是否真的是异步的。如果该参数被设定为LIO_WAIT, lio_listio函数将在所有由列表指定的I/O操作完成后返回。在这种情况下，sigev参数将被忽略。如果mode参数设定为LIO_NOWAIT，lio_listio函数将在I/O请求入队后立即返回。进程将在所有I/O操作完成后，按照sigev参数指定的，被异步地通知。如果不想被通知，可以把sigev设定为NULL. 注意，每个AIO控制块本身也可能启用了在各自操作完成时的异步通知。被sigev参数指定的异步通知是在此之外另加的，并且只会在所有I/O操作完成后发送。
  
  list参数指向AIO控制块列表，该列表指定了要运行的I/O操作的。nent参数制定了额数组中元素的个数。AIO控制块列表还可以包含NULL指针，这些太哦亩将被忽略。
  
  在每一个AIO控制块中，aio_lio_opcode字段制定了该操作是一个读操作(LIO_READ), 写操作(LIO_WRITE)，还是将被忽略的空操作(LIO_NOP). 读操作会按照对应的AIO控制块被传给了aio_read函数来处理。类似的，写操作会按照对应的AIO控制块被传给了aio_write函数来处理。
  
  实现会限制我们不想实现的异步操作I/O操作的数量。这些都是运行时不变量，其总结如下图。
  
  可以通过sysconf函数并把name参数设置为_SC_IO_LISTIO_MAX来设定AIO_LISTIO_MAX的值。类似的，可以通过调用sysconf并把name设置为_SC_AIO_MAX来设置AIO_MAX的值，通过调用sysconf并把其参数设置为_SC_AIO_PRIO_DELTA_MAX来设定AIO_PRIO_DELTA_MAX的值。
  
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/aio_runtime_const.png)
  
  引入POSIX异步操作I/O接口的初衷时为实时应用提供一种方法，避免在执行I/O操作时阻塞进程。接下来就让我们看一个使用这些接口的例子。
  
  实例：
  虽然我们不会在本文中讨论实时编程，但是因为POSIX异步I/O接口现在是Single Unix Specification的基本部分，所以我们要了解以下怎么使用它们。为了对比异步I/O接口和相应传统I/O接口，我们来研究一个任务，将一个文件从一种格式翻译成另一种格式。
  
  下面的程序是使用20世纪80年代流行的USENET新闻系统中使用的ROT-13算法，翻译文件，该算法原本用于将文本中带有侵犯性的或含有剧透和笑话笑点部分的文本模糊化。该算法将文本中的英文字符a~z和A~Z分别循环向右移13个字母位移，但步改变其他字符。
  
```
#include "apue.h"
#include <ctype.h>
#include <fcntl.h>

#define BSZ 4096

unsigned char buf[BSZ];

unsigned char
translate(unsigned char c)
{
    if(isalpha(c)) {
        if(c >= 'n')
            c -= 13; 
        else if(c >= 'a')
            c += 13; 
        else if(c >= 'N')
            c -= 13; 
        else
            c += 13; 
    }   

    return (c);
}

int main(int argc, char **argv)
{
    int ifd, ofd, i, n, nw; 
    if(argc != 3)
        err_quit("usage: rot13 infile outfile");

    if((ifd = open(argv[1], O_RDONLY)) < 0)
        err_sys("can't open %s", argv[1]);

    if((ofd = open(argv[2], O_RDWR | O_CREAT | O_TRUNC, FILE_MODE)) < 0)
        err_sys("can't create %s", argv[2]);

    while((n = read(ifd, buf, BSZ)) > 0) {
        for(i = 0; i < n; i++)
            buf[i] = translate(buf[i]);

        if((nw = write(ofd, buf, n)) != n) {
            if(nw < 0)
                err_sys("write failed");
            else
                err_quit("short write (%d/%d)", nw, n); 
        }   
    }   

    fsync(ofd);
    exit(0);
}
```
  程序中的I/O部分是很直接的: 从输入文件中读取一个块，翻译之，然后再把这个块写到输出文件中。重复该步骤直到遇到文件尾端,read返回0. 下面是等价的异步I/O函数做同样的任务.
  
```
#include "apue.h"                                                                      
#include <ctype.h>                                                                     
#include <fcntl.h>                                                                     
#include <aio.h>                                                                       
#include <errno.h>                                                                     
#include <sys/stat.h>                                                                  
                                                                                       
#define BSZ 4096                                                                       
#define NBUF 8                                                                         
                                                                                       
enum rwop {                                                                            
    UNUSED = 0,                                                                        
    READ_PENDING = 1,                                                                  
    WRITE_PENDING = 2                                                                  
};                                                                                     
                                                                                       
struct buf {                                                                           
    enum rwop     op;                                                                  
    int           last;                                                                
    struct aiocb  aiocb;                                                               
    unsigned char data[BSZ];                                                           
};                                                                                     
                                                                                       
struct buf bufs[NBUF];                                                                 
                                                                                       
unsigned char                                                                          
translate(unsigned char c)                                                             
{                                                                                      
    /* same as before */                                                               
    if (isalpha(c)) {                                                                  
        if (c >= 'n')                                                                  
            c -= 13;                                                                   
        else if (c >= 'a')                                                             
            c += 13;                                                                   
        else if (c >= 'N')                                                             
            c -= 13;                                                                   
        else                                                                           
            c += 13;                                                                   
    }                                                                                  
    return(c);                                                                         
}                                     

int
main(int argc, char* argv[])
{
    int                    ifd, ofd, i, j, n, err, numop;
    struct stat            sbuf;
    const struct aiocb    *aiolist[NBUF];
    off_t                off = 0;

    if (argc != 3)
        err_quit("usage: rot13 infile outfile");
    if ((ifd = open(argv[1], O_RDONLY)) < 0)
        err_sys("can't open %s", argv[1]);
    if ((ofd = open(argv[2], O_RDWR|O_CREAT|O_TRUNC, FILE_MODE)) < 0)
        err_sys("can't create %s", argv[2]);
    if (fstat(ifd, &sbuf) < 0)
        err_sys("fstat failed");

    /* initialize the buffers */
    for (i = 0; i < NBUF; i++) {
        bufs[i].op = UNUSED;
        bufs[i].aiocb.aio_buf = bufs[i].data;
        bufs[i].aiocb.aio_sigevent.sigev_notify = SIGEV_NONE;
        aiolist[i] = NULL;
    }
    
    
    numop = 0;
    for (;;) {
        for (i = 0; i < NBUF; i++) {
            switch (bufs[i].op) {
            case UNUSED:
                /*
                 * Read from the input file if more data
                 * remains unread.
                 */
                if (off < sbuf.st_size) {
                    bufs[i].op = READ_PENDING;
                    bufs[i].aiocb.aio_fildes = ifd;
                    bufs[i].aiocb.aio_offset = off;
                    off += BSZ;
                    if (off >= sbuf.st_size)
                        bufs[i].last = 1;
                    bufs[i].aiocb.aio_nbytes = BSZ;
                    if (aio_read(&bufs[i].aiocb) < 0)
                        err_sys("aio_read failed");
                    aiolist[i] = &bufs[i].aiocb;
                    numop++;
                }
                break;
            case READ_PENDING:
                if ((err = aio_error(&bufs[i].aiocb)) == EINPROGRESS)
                    continue;
                if (err != 0) {
                    if (err == -1)
                        err_sys("aio_error failed");
                    else
                        err_exit(err, "read failed");
                }

                /*
                 * A read is complete; translate the buffer
                 * and write it.
                 */
                if ((n = aio_return(&bufs[i].aiocb)) < 0)
                    err_sys("aio_return failed");
                if (n != BSZ && !bufs[i].last)
                    err_quit("short read (%d/%d)", n, BSZ);
                for (j = 0; j < n; j++)
                    bufs[i].data[j] = translate(bufs[i].data[j]);
                bufs[i].op = WRITE_PENDING;
                bufs[i].aiocb.aio_fildes = ofd;
                bufs[i].aiocb.aio_nbytes = n;
                if (aio_write(&bufs[i].aiocb) < 0)
                    err_sys("aio_write failed");
                /* retain our spot in aiolist */
                break;
            case WRITE_PENDING:
                if ((err = aio_error(&bufs[i].aiocb)) == EINPROGRESS)
                    continue;
                if (err != 0) {
                    if (err == -1)
                        err_sys("aio_error failed");
                    else
                        err_exit(err, "write failed");
                }

                /*
                 * A write is complete; mark the buffer as unused.
                 */
                if ((n = aio_return(&bufs[i].aiocb)) < 0)
                    err_sys("aio_return failed");
                if (n != bufs[i].aiocb.aio_nbytes)
                    err_quit("short write (%d/%d)", n, BSZ);
                aiolist[i] = NULL;
                bufs[i].op = UNUSED;
                numop--;
                break;
            }
        }
        if (numop == 0) {
            if (off >= sbuf.st_size)
                break;
        } else {
            if (aio_suspend(aiolist, NBUF, NULL) < 0)
                err_sys("aio_suspend failed");
        }
    }

    bufs[0].aiocb.aio_fildes = ofd;
    if (aio_fsync(O_SYNC, &bufs[0].aiocb) < 0)
        err_sys("aio_fsync failed");
    exit(0);
}
```
  我们使用了8个缓冲区，因此可以有最多8个异步I/O请求处于等待状态。令人惊讶的是，实际上这可能会降低性能，因为如果读操作是以无序的方式提交给文件系统的，操作系统提前读的算法便会失效。
  
  在检查操作的返回值之前，必须确认操作已经完成。当aio_error返回的值既非EINPROGRESS亦非-1时，表明操作完成。除了这些值之外，如果返回值是0以外的任何值，说明操作失败了。一旦检查过这些情况，便可以安全的调用aio_return来获取i/O操作的返回值来。
  
  只要还有事情要做，就可以提交异步I/O操作。当存在为使用的AIO控制块时，可以提交一个异步读操作，读操作完成后，编译缓冲区中的内容并将它提交给一个异步写请求。当所有AIO控制块都在使用中时，通过调用aio_suspend等待操作完成。
  
  把一个块写入输出文件时，我们保留了在从输入文件读取数据时的偏移量。因而写的顺序并不重要。这一策略仅在输入文件中每个字符和输出文件中对应的字符偏移量相同的情况下适用，我们在输出文件中既没有添加字符也没有删除字符。
  
  这个实例中并没有使用异步通知，因为使用同步编程模型更加简单。如果在I/O操作进行时还有别的事情要做，那么额外的工作可以包含在for循环当中。然而，如果需要阻止这些额外工作延迟翻译文件的任务，那么就需要组织下代码使用异步通知。多任务情况下，决定程序如何构建之前需要先考虑各个任务的优先级。

  [AIO详解](https://github.com/walkerqiao/walkman/blob/master/docs/APUE/aio_desc.md)
  
### 14.6 readv和writev函数
  readv和writev函数用于在一次函数调用中读、写多个非连续缓冲区。有时也将这两个函数称为散布读(scatter read)和聚集写(gather write).
  
```
#include <sys/uio.h>
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
```
  这两个函数的第二个参数是指向iovec结构数组的一个指针:
```
struct iovec {
  void *iov_base; /** starting address of buffer **/
  size_t iov_len; /** size of buffer **/
}
```
  iovshuzu中的元素数由iovcnt指定，其最大值受限于IOV_MAX. 图14-22显示了这两个函数的参数和iovec之间的关系。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/iov_structure_with_readv_writev.png)
  
  writev函数从缓冲区中聚集输出数据的顺序是iov[0], iov[1], 直至iov[iovcnt - 1]. writev返回输出的字节总数，通常应等于所有缓冲区长度之和。
  
  readv函数将读入的数据按照上述同样的顺序散布到缓冲区中。readv总是先填满一个缓冲区后，然后再填写下一个。readv返回都到的字节总数。如果遇到文件尾端，已无数据可读，则返回0.
  
> 这两个函数始于4.2BSD,后来，SVR也提供它们。在Single Unix Specification的XSI扩展中包括了这两个函数。

  实例
  在20.8节中的_db_writeidx函数中，需要将两个缓冲区中的内容连续地写到一个文件中。第二个缓冲区是调用者传递过来的一个参数，第一个缓冲区是我们创建的，它包含了第二个缓冲区的长度以及文件中其他信息的文件偏移量。有3中方法可以实现这一要求。
  1. 调用两次write, 每个缓冲区一次。
  2. 分配一个大到足以包含两个缓冲区的新缓冲区。将两个缓冲区的内容复制到新缓冲区中，然后对这个新缓冲区调用一次write.
  3. 调用writev输出两个缓冲区。
  
  20.8节的解决方案使用了writev, 但是将它与另外两个方法进行比较，对我们是很有启发的。下图显示了上面3种方法的结果。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/writev_vs_others.png)

  图中解释略去。结论: 应当尽量少的系统调用次数来完成任务。如果我们只写少量的数据，将会发现复制数据然后使用一次write比用writev要划算。但也可能发现，我们管理自己的分段缓冲区会增加程序额外的复杂性成本，所以从性能成本的角度来看不合算。
  

### 14.7 readn和writen函数
  管道、FIFO以及某些设备(特别是终端和网络)有下列两种性质：
  1. 一次read操作所返回的数据可能少于所要求的数据，即使还没有达到文件尾端也可能是这样。这不是一个错误，应当继续读该设备。
  2. 一次write操作的返回值也可能少于指定输出的字节数。这可能是由于某个因素造成的，例如，内核输出缓冲区变满。这也不是错误，应当继续写余下的数据。(通常，只有非阻塞描述符，或捕捉到一个信号时，才发生这种write的中途返回。)

  在读、写磁盘文件时从未见到过这种情况，除非文件系统用完了空间，或者接近了配额限制，不能将要求写的数据全部写出。
  
  通常，在读写一个管道、网络设备或终端时，需要考虑这些特性。下面两个函数readn和writen的功能分别是读、写指定的N字节数据，并处理返回值可能小于要求值的情况。这两个函数只是按需多次调用read和write直至读、写了N字节数据。
```
#include "apue.h"

ssize_t readn(int fd, void *buf, size_t nbytes);
ssize_t writen(int fd, void *buf, size_t nbytes);
```
> 类似于本书很多实例所用的出错处理例程，我们定义这两个函数是便于在后面的例子中使用。readn和writen不是哪个标准的组成部分。

  要将数据写到上面所提到的文件类型时，就可调用writen, 但是仅当实现就知道要接收数据的数据量时，才调用readn. 下面是readn和writen的实现:
```
#include "apue.h"
ssize_t /** read n bytes from a fd **/
readn(int fd, void *ptr, size_t n)
{
  size_t nleft;
  ssize_t nread;
  
  nleft = n;
  while(nleft > 0) {
    if((nread = read(fd, ptr, nleft)) < 0) {
      if(nleft == n)
        return (-1); /** error, return -1 **/
      else
        break; /** error, return amount read so for **/
    } else if(nread == 0) {
      break; /** EOF **/
    }
    
    nleft -= nread;
    ptr += nread;
  }
  
  return (n - nleft); /** return >= 0 **/
}

ssize_t             /* Write "n" bytes to a descriptor  */
writen(int fd, const void *ptr, size_t n)
{
    size_t        nleft;
    ssize_t        nwritten;

    nleft = n;
    while (nleft > 0) {
        if ((nwritten = write(fd, ptr, nleft)) < 0) {
            if (nleft == n)
                return(-1); /* error, return -1 */
            else
                break;      /* error, return amount written so far */
        } else if (nwritten == 0) {
            break;
        }
        nleft -= nwritten;
        ptr   += nwritten;
    }
    return(n - nleft);      /* return >= 0 */
}
```
  注意，若在已经读、写了一些数据之后出错，则这两个函数返回的是已经传输的数据量，而非错误。与此类似，在读时，如达到文件尾端，而且在此之前已成功地读了一些数据，但尚未满足所要求地量，则readn返回已复制到调用者缓冲区中的字节数。
  
### 14.8 内存映射I/O
  存储映射I/O(memory-mapped I/O)能将一个磁盘文件映射到存储空间的一个缓冲区上，于是，当从缓冲区中取数据时，就相当于读文件中的相应字节。与此类似，将数据存入缓冲区时，相应字节就自动写入文件。这样，就可以在不使用read和write的情况下执行I/O.
  
> 存储映射I/O伴随虚拟存储系统已经用了很多年。1981年，4.1BSD以其vread和vwrite函数提供了一种不同形式的存储映射I/O. 4.2BSD实际上并没有包含mmap函数。SUSv4把mmap函数从可选项规范中移到了基础规范中。所有的遵循POSIX的系统都需要支持它。

  为了使用这种功能，应首先告诉内核将一个给定的文件映射到一个存储区域中。这是由mmap函数实现的。
```
#include <sys/mmap.h>
void *mmap(void *addr, size_t len, int prot, int flag, int fd, off_t off);
```
  addr参数用于指定映射存储区的起始地址。通常将其设置为0，这表示由系统选择该映射区的起始地址。此函数的返回值是该映射区的起始地址。
  
  fd参数是指定要被映射文件的描述符。在文件映射到地址空间之前，必须先打开该文件。len参数是映射的字节数，off是要映射字节在文件中的偏移量。
  
  prot参数指定了映射存储区的保护要求。见下图：
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/mmap_prot.png)
  
  可将prot参数指定为PROT_NONE, 也可以指定为PROT_WRITE, PROT_EXEC的任意组合的按位或。对指定映射存储区的保护要求不能超过文件open模式访问权限。例如，若该文件是只读打开的，那么对映射存储区就不能指定PROT_WRITE.
  
  在说明flag参数之前，先看一下存储映射文件的基本情况。图14-26显示了一个存储映射文件。

### 14.9 总结


