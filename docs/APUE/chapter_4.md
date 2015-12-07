## 第四章 文件和目录

### 4.1 引言
  上一章我们说明了执行I/O操作的基本函数。其讨论围绕普通文件的I/O进行--打开一个文件、读或写一个文件。 本章将观察文件系统的其他特征和文件的性质。我们将从stat函数开始，逐个说明stat结构的每一个成员以了解文件的所有属性。 在此过程中，我们将说明修改这些属性的各个函数(更改所有者、更改许可权等)，还将更详细地察看Unix文件系统地结构以及符号连接。 本章结束部分介绍对目录进行操作的各个函数，并且开发了一个以降序遍历目录层次结构的函数。
  
### 4.2 stat, fstat, fstatat, lstat函数
  本章讨论的重视是三个stat函数以及它们所返回的信息。
```
#include <sys/types.h>
#include <sys/stat.h>

int stat(const char *pathname, struct stat *buf);
int fstat(int filedes, struct stat *buf);
int lstat(const char *pathname, struct stat *buf);
```
  给定一个pathname，stat返回一个与此命名文件有关的信息结构，fstat函数获得已在描述符filedes上打开的文件的有关信息。lstat函数类似于stat, 但是当命名的文件是一个符号连接时，lstat返回该符号连接的有关信息， 而不是由该符号连接引用的文件的信息。
  
  第二个参数是一个指针，它指向一个我们应提供的结构。这些函数填写由buf指向的结构。 该结构的实际定义可能随实现而有所不同，但其基本形式是:
```
struct stat {
     mode_t          st_mode;      /** 文件类型及模式(权限) **/
     ino_t           st_ino;       /** i-node number (serial number) **/
     dev_t           st_dev;       /** device number (filesystem) **/
     dev_t           st_rdev;      /** device number for special files **/
     nlink_t         st_nlink;     /** number of links **/
     uid_t           st_uid;       /** user ID or owner **/
     gid_t           st_gid;       /** group ID or owner **/
     off_t           st_size;      /** size in bytes, for regular files **/
     struct timespec st_atim;      /** 最后访问时间 **/
     struct timespec st_mtim;      /** 最后修改时间 **/
     struct timespec st_ctim;      /** 文件最后状态改变时间 **/
     blksize_t       st_blksize;   /* best I/O block size */
     blkcnt_t        st_blocks;    /* number of disk blocks allocated */
};
```
  注意，除了最后两个以外，其他各成员都位基本系统数据类型。我们将说明此结构的每个成员以了解文件的属性。

### 4.3 文件类型
  至今我们已介绍了两种不同的文件类型--普通文件和目录。Unix系统的大多数文件是普通文件或目录，但是也有另外一些文件类型:
  1. 普通文件(regular file): 这是最常见的文件类型， 这种文件包含了某种形式的数据。至于这种数据是文本还是二进制数据对于内核而言并无区别。 对普通文件内容的解释由处理该文件的应用程序进行。
  2. 目录文件(directory file): 这种文件包含了其他文件的名字以及指向与这些文件有关信息的指针。对一个目录文件具有读许可权的任一进程都可以读该目录的内容，但只有内核可以写目录文件。
  3. 字符特殊文件(character special file): 这种文件用于系统中某些类型的设备。
  4. 块特殊文件(block special file): 这种文件典型地用于磁盘设备。系统中地所有设备或者是字符特殊设备，或者是块特殊设备。
  5. FIFO: 这种文件用于进程间地通信，有时也将其命名位管道。
  6. 套接字(socket): 这种文件用于进程间的网络通信。 套接口可以用于在一台宿主机上的进程之间的非网络通信。
  7. 符号连接(symbolic link): 这种文件指向另一个文件。
  
  文件类型信息包含在stat结构的st_mode成员中。 可以用表4-1中的宏确定文件类型。这些宏的参数都是stat结构中的st_mode成员。
```
宏              文件类型
S_ISREG()       普通文件
S_ISDIR()       目录文件
S_ISCHR()       字符特殊文件
S_ISBLK()       块特殊文件
S_ISFIFO()      管道或FIFO
S_ISLNK()       符号连接(POSIX.1或SVR4无此类型)
S_ISSOCK()      套接字(POSIX.1或SVR4无此类型)
```
  实例: 从命令行参数读取参数，并判断每个参数对应的文件属于什么类型的。
```
#include <sys/types.h>
#include <sys/stat.h>

#include "apue.h"

int main(int argc, char **argv)
{
    int i;
    struct stat buf;
    char *ptr;

    for(i = 0; i < argc; i++)
    {   
        printf("%s: ", argv[i]);
        if(lstat(argv[i], &buf) < 0) {
            err_ret("lstat error");
            continue;
        }   

        if      (S_ISREG(buf.st_mode)) ptr = "regular";
        else if (S_ISDIR(buf.st_mode)) ptr = "directory";
        else if (S_ISCHR(buf.st_mode)) ptr = "character special";
        else if (S_ISBLK(buf.st_mode)) ptr = "block special";
        else if (S_ISFIFO(buf.st_mode)) ptr = "fifo";

#ifdef S_ISLNK
        else if (S_ISLNK(buf.st_mode)) ptr = "symbolic link";
#endif
#ifdef S_ISSOCK
        else if (S_ISSOCK(buf.st_mode)) ptr = "socket";
#endif
        else ptr = "** unknown mode **";
        printf("%s\n", ptr);
    }   
    exit(0);
}
```

```
$ ./filetype /vmunix /etc/ /dev/ttya /dev/sd0a /var/spool/cron/FIFO /bin /dev/printer
/vmunix: regular
/etc: directory
/dev/ttya: character special
/dev/sd0a: block special
/var/spool/cron/FIFO: fifo
/bin: symbolic link
/dev/printer: socket
```

  上面代码中我们特地使用了lstat而没有使用stat, 因为stat是不检查符号连接的。
  
  早期的Unix版本，并不提供S_ISxxx, 于是需要将st_mode与屏蔽字S_IFMT逻辑与，然后与名为S_IFxxx的常数相比较。
```
#define S_ISDIR(mode) (((mode) & S_IFMT) == S_IFDIR)
```

  我们说过，普通文件是最主要的文件类型，但是观察一下在一个特定系统中各种文件的比例是很有意思的。 下表显示了一个中等规模的系统中的统计值。这一数据由后面4.21节的程序得到的。
```
文件类型          计数值            比例(%)
普通文件          30 369            91.7
目录               1 901            5.7
符号连接             416            1.3
字符特殊             373            1.1
块特殊                61            0.2
套接口                 5            0.0
FIFO                   1            0.0
```


### 4.4 设置用户ID和组ID
  与一个进程相关联的ID有六个或更多，它们如下:
  1. 实际代表谁的IDs:   实际用户ID, 实际组ID。
  2. 用于文件存取许可权检查IDs:   有效用户ID, 有效组ID, 添加组ID。决定文件访问权
  3. 由exec函数保存的IDs: 保存设置－用户ID, 保存设置－组ID. 在执行一个程序时包含了有效用户ID和有效组ID的副本。

  通常，有效用户ID等于实际用户ID, 有效组ID等于实际组ID.
  每个文件有一个所有者和组所有者，所有者由stat结构中的st_uid表示，组所有者则由st_gid成员表示。
  
  当执行一个程序时，进程的有效用户ID通常就是实际用户ID, 有效组ID通常是实际组ID. 但是可以在文件方式字(st_mode)中设置一个特殊标志，其定义是"当执行此文件时，将进程的有效用户ID设置为文件的所有者(st_uid)." 与此相类似，在文件方式字中可以设置另一位，它使得执行此文件的进程的有效组ID设置为文件的组所有者st_gid. 在文件方式字中的这两位被称之为设置-用户-ID(set-user-ID)位和(set-group-ID)位。
  
  例如，若文件所有者是超级用户，而且设置了该文件的set-user-ID位，然后当该程序由一个进程运行时，则该进程具有超级用户优先权。不管执行此文件的进程实际用户ID是什么，都做这种处理。作为一个例子，Unix程序passwd(1)允许任一用户改变其口令，该程序是一个set-user-ID程序。因为该程序能将用户的新口令写入口令文件中(一般是/etc/passwd或/etc/shadow), 而只有超级用户才具有对该文件的写许可权，所以需要使用set-user-ID特征。 因为运行set-user-ID程序的进程通常得到额外的许可权，所以编写这种程序时要特别谨慎。
  
  再返回到stat函数，set-user-ID及set-group-ID都包含在st_mode值中。这两位可用常数S_ISUID和S_ISGID测试。

### 4.5 文件存取许可权
  st_mode值也包含了对文件的存取许可权位。当提及文件时，指的是前面所提到的任何类型的文件。所有文件类型都由存取许可权。很多任人为只有普通文件有存取许可权，这是一种误解。
  
  每个文件有9各存取许可权位，可将它们分成三类，见下表:
```
st_mode屏蔽   意义
---------------------------
S_IRUSR       用户-读
S_IWUSR       用户-写
S_IXUSR       用户-执行
---------------------------
S_IRGRP       组-读
S_IWGRP       组-写
S_IXGRP       组－执行
---------------------------
S_IROTH       其他-读
S_IWOTH       其他-写
S_IXOTH       其他-执行
---------------------------
```
  上表所示，分别表示文件所有者、所有者所在组成员、以及其他组成员的读写执行权限。chmod(1)用于修改这些权限，该命令允许使用u表示用户，g表示用户所在组，o表示其他组。
  
  第一个规则是，我们用名字打开任一类型的文件时，对该名字中包含的每一个目录，包括它可能隐含的当前工作目录都具有执行许可权。这就是为什么对于目录其执行许可权常被称为搜索位的原因。
  例如，为了打开文件/usr/dict/words, 需要具有对目录/，/usr, /usr/dist的执行许可权。然后，需要对该文件本身的适当许可权，这取决于以何种方式打开它(只读,读-写等)。
  
  如果当前目录是/usr/dict, 那么为了打开文件words， 需要有对该目录的执行许可。 如在指定打开文件words时，可以隐含当前目录，而不用显示地提及/usr/dict, 也可使用./words.
  
  注意，对于目录的读许可权和执行许可权的意义不同。 读许可权允许我们读目录，获得在该目录中所有文件名的列表。当一个目录时我们要存取文件的一个分量时，对该目录的执行许可权使我们可通过该目录(也就是搜索该目录，寻找一个特定的文件名。)
  引用隐含目录的另一个例子是，如果PATH环境变量指定了一个我们不具有执行许可权的目录，那么shell绝不会在该目录下找到可执行文件。
  
  对于一个文件的读许可权决定我们是否能够打开该文件进行读操作。这对应于open函数的O_RDONLY和O_RDWR标志。
  
  对于一个文件的写许可权决定了我们是否能够打开该文件进行写操作。 这对应于open函数的O_WRONLY和O_RDWR标志。
  
  为了在open函数中对一个文件指定O_TRUNC标志，必须对该文件具有写许可权。
  
  为了在一个目录中创建一个新文件，必须对该目录具有写许可权和执行许可权。
  
  为了删除一个文件，必须对包含该文件的目录具有写许可权和执行许可权。对该文件本身则不需要读写权限。
  
  如果用6个exec函数中的任何一个执行某个文件，都必须对该文件具有执行许可权。
  
  进程每次打开、创建或删除一个文件时，内核就进行文件存取许可权测试，而这种测试可能涉及文件的所有者(st_uid和st_gid), 进程的有效ID(有效用户ID和有效组ID)以及进程的添加组ID(若支持的话)。 两个所有者ID时文件的性质，而有效ID和添加组ID则是进程的性质。内核进行的测试是:
  1. 若进程的有效用户ID是0(超级用户)，则允许存取。这给予了超级用户对文件系统进行处理的最充分的自由。
  2. 若进程的有效用户ID等于文件的所有者ID(也就是该进程拥有此文件):
  <ul>
    <li>若适当的所有者存取许可权位被设置，则允许存取</li>
    <li>否则拒绝存取</li>
    <li>适当的存取许可权位指的是，若进程为读而打开该文件，则用户读位应为1；若进程为写而打开该文件，则用户-写位应为1. 若进程将执行该文件，则用户-执行位应为1</li>
  </ul>
  3. 若进程的有效组ID或进程的添加组ID之一等于文件的组ID:
  <ul>
    <li>若适当的组存取许可权位被设置，则允许存取</li>
    <li>否则拒绝存取</li>
  </ul>
  4. 若使得那个的其他用户存取许可位被设置，则允许存取，否则拒绝存取。
  
  按顺序执行这四步。注意，如若进程拥有此文件，则按用户存取许可权批准或拒绝该进程对文件的存取--不查看组存取许可权。相类似，进程并不拥有该文件，但进程属于某个适当的组，按组存取许可权批准或拒绝该进程对文件的存取--不察看其他用户的存取许可权。

### 4.6 新文件和目录的所属关系
  在第三章中，当说明用open或creat创建新文件时，没有说明赋予新文件的用户ID和组ID的值是什么。4.20节将说明如何创建一个新目录以及mkdir函数。关于新目录的所有权的规则与本节将说明的新文件的所有权规则相同。
  
  新文件的用户ID设置为进程的有效用户ID， 关于组ID, POSIX.1允许选择下列之一作为新文件的组ID.
  1. 新文件的组ID可以是进程的有效组ID
  2. 新文件的组ID可以是它所在目录的组ID.

### 4.7 access和faccessat函数
  正如前面所说，当用open函数打开一个文件时，内核以进程的有效用户ID和有效组ID为基础执行其存取许可权测试。有时，进程也希望按照其实际用户ID和实际组ID来测试其存取能力。例如，当一个进程使用set-user-ID或set-group-ID特征作为另一个用户组运行时，这就可能需要。即使一个进程可能已经set-user-ID为root, 它仍可能相验证实际用户能否存取一个给定的文件。access函数是按实际用户ID和实际组ID进行存取许可权测试的。
```
#include <unistd.h>
int access(const char *pathname, int mode)
```
  其中mode是按下列常数逐位或运算:
  * R_OK: 测试读许可权
  * W_OK: 测试写许可权
  * X_OK: 测试执行许可权
  * F_OK: 测试文件是否存在

  实例：展示access函数的使用
```
#include <sys/types.h>
#include <fcntl.h>
#include "apue.h"

int
main(int argc, char **argv)
{
    if(argc != 2)
        err_quit("usage: a.out <pathname>");

    if(access(argv[1], R_OK) < 0)
        err_ret("access error for %s", argv[1]);
    else
        printf("read access OK\n");

    if(open(argv[1], O_RDONLY) < 0)
        err_ret("open error for %s", argv[1]);
    else
        printf("open for reading OK\n");

    exit(0);
}
```
  
### 4.8 umask函数
  至此，我们已经说明了每个文件相关联的9个存取许可权位，在此基础上我们可以说明与每个进程相关联的方式创建屏蔽字。
  umask函数为进程设置文件方式创建屏蔽字，并返回以前的值。(这是少数几个没有出错返回的函数之一。)
```
#include <sys/types.h>
#include <sys/stat.h>

mode_t umask(mode_t cmask);
```
  其中，参数cmask是由S_IRUSR, S_IWUSR等常量进行逐位"或"构成的。
  
  在进程创建一个新文件或新目录时，就一定会使用文件创建屏蔽字(回忆3.3和3.4节，在那里我们说明了open和creat函数。这两个函数都有一个参数mode,它指定了新文件的存取许可权)。我们将在4.20节说明如何创建一个新目录，在文件方式创建屏蔽字中为1的位，在文件mode中的相应位则一定被转成0.
  
  实例:程序创建两个文件，创建第一个时，umask值为0，创建第二个时，umask值禁止所有组和其他存取许可权。若运行此程序可得到如下结果，从中可见存取许可权是如何设置的。
```
bogon:io apple$ umask
0022
bogon:io apple$ ls -la foo bar 
-rw-------  1 apple  staff  0 12  4 11:39 bar
-rw-rw-rw-  1 apple  staff  0 12  4 11:39 foo
bogon:io apple$ umask
0022
```

```
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>

#include "apue.h"

#define RWRWRW (S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH)
int
main(void)
{
    umask(0);
    if(creat("foo", RWRWRW) < 0)
        err_sys("creat error for foo");

    umask(S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH);
    if(creat("bar", RWRWRW) < 0)
        err_sys("creat error for bar");

    exit(0);
}
```
  大多数Unix系统用户根本不会和umask的值打交道。通常登录时通过shell启动文件设置一次，然后不再更改。然而，当写需要创建文件的程序时， 如果我们需要确保特定访问权限位启用，我们必须在进程运行时修改umask值。例如，如果我们想要启用任何任都可以读取文件，我们应该设置umask为0. 否则，umask值在进程运行时会产生负面影响，会导致某些权限位被关掉。
  
  在之前的例子中，我们使用shell的umask命令在创建文件前后打印文件模式。展示结果可以看到，子进程中修改umask不会影响父进程的umask. 
  
  用户可以设置umask值来控制创建文件的默认权限。 以八进制表示，一位表示要masked off的一个权限。 
```
Mask bit       meaning
0400          user-read
0200          user-write
0100          user-execute
0040          group-read
0020          group-write
0010          group-execute
0004          other-read
0002          other-write
0001          other-execute
```
  
### 4.9 chmod, fchmod, fchmodat函数
  chmod, fchmod, fchmodat函数允许我们对既有文件进行权限修改。
```
#include <sys/stat.h>
int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
int fchmodat(int fd, const char *pathname, mode_t mode, int flag);
```
  chmod在指定的文件上进行操作，而fchmod则对已打开的文件进行操作。fchmodat在pathname参数是绝对路径或当fd参数值为AT_FDCWD，pathname参数相对路径时行为和chmod一样。否则，fchmodat根据打开的fd计算pathname. flag参数可以用于改变fchmodat的行为--设置为AT_SYMLINK_NOFOLLOW，fchmodat不会跟着symbolic links.
  
  为了更改一个文件的许可权位，进程的有效用户ID必须等于文件的所有者，或者该成进程必须具有超级用户许可权。
  参数mode时下表所示常数的某种逐位或运算。
```
mode                  说明
-----------------------------------
S_ISUID       执行时set-user-ID
S_ISGID       执行时set-group-ID
S_ISVTX       保存正文
-----------------------------------
S_IRWXU       用户(所有者)读写和执行
  S_IRUSR     用户(所有者)读
  S_IWUSR     用户(所有者)写
  S_IXUSR     用户(所有者)执行
-----------------------------------
S_IRWXG       组读、写、执行
  S_IRGRP     组读
  S_IWGRP     组写
  S_IXGRP     组执行
-----------------------------------
S_IRWXO       其他读、写、执行
  S_IROTH     其他读
  S_IWOTH     其他写
  S_IXOTH     其他执行
-----------------------------------
```

```
#include <sys/types.h>
#include <sys/stat.h>

#include "apue.h"

int
main(void)
{
    struct stat statbuf;

    /** turn on set-group-ID and turn off group-execute **/
    if(stat("foo", &statbuf) < 0)
        err_sys("stat error for foo");

    if(chmod("foo", (statbuf.st_mode & ~S_IXGRP) | S_ISGID) < 0)
        err_sys("chmod error for foo");

    /** set absolute mode to "rw-r--r--" **/
    if(chmod("bar", S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH) < 0)
        err_sys("chmod error for bar");

    exit(0);
}
```
  编译并运行，结果如下:
```
bogon:io apple$ ls -la foo bar
-rw-------  1 apple  staff  0 12  4 11:39 bar
-rw-rw-rw-  1 apple  staff  0 12  4 11:39 foo
bogon:io apple$ ./chmod 
bogon:io apple$ ls -la foo bar 
-rw-r--r--  1 apple  staff  0 12  4 11:39 bar
-rw-rwSrw-  1 apple  staff  0 12  4 11:39 foo
```
  `statbuf.st_mode & ~S_IXGRP`: 表示原来的st_mode与'rw-rw-rw-'进行与运算。 然后和S_ISGRP或运算。 因此foo的权限被修改为-rw-rwSrw-. 而bar权限直接修改为rw-r--r--. 通过S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH。
  
  同时我们可以看到，chmod后，文件日期并不发生变化。在4.18节中，我们会了解到chmod函数更新的只是i节点最近一次被更改的时间。按系统默认方式，ls -l列出的时最后修改文件内容的时间。
  chmod函数在下列条件下自动清除两个许可权位。
  * 如果我们试图设置普通文件的粘住位(S_ISVTX),而且又没有超级用户优先权，那么mode中的粘住位自动被关闭。这意味着只有超级用户才能设置普通文件的粘住位。这样做的理由是可以防止不怀好意的用户设置粘住位，并试图以此方式填满交换区(如果系统支持保存－正文特征的话)。
  * 新建文件的组ID可能不是调用进程所属的组。回忆一下，新文件的组ID可能是父目录的组ID. 特别地，如果新文件地组ID不等于进程地有效组ID或者进程添加组中的一个， 以及进程没有超级用户优先数，那么set-group-ID会被关闭。 这就防止了用户创建一个set-group-ID文件，而该文件是由并非该用户所属的组拥有的。
  

### 4.10 粘住位(Sticky Bit)
  S_ISVTX位有一段有趣的历史。 在Unix的早期版本中，有一位被称为粘住位(sticky bit)。如果一个可执行文件的这一位被设置了，那么在该程序第一次执行并结束时，该程序正文的一个文本被保存在交换区。(程序的正文部分时机器指令部分。)这使得下次执行该程序的时候能较快地将其装入内存区。 其原因是: 在交换区，该文件是被连续存放的， 而在一般的Unix文件系统中，文件的各数据块很可能是随即存放的。 对于常用的应用程序，例如文本编辑程序和编译程序的个部分常常设置它们所在的文件粘住位。 自然，对交换区中可以同时存放的设置了粘住位的文件数有一定限制的， 以免过多占用交换区空间，但无论如何这是一个有用的技术。 因为在系统再次自举前，文件的正文部分总是在交换区中，所以使用了名字“粘住”。 后来的Unix版本称之位save-text bit, 因此也就有了常数S_ISVTX. 现今较新的Unix系统大多数都具有虚存系统以及快速文件系统，所以不再需要使用这种技术。
  
  SVR4和4.3+BSD中粘住位的主要针对目录。如果对一个目录设置了粘住位，则只有对该目录具有写许可权的用户并且满足下列条件之一，才能删除或更名该目录下的文件:
  * 拥有该文件
  * 拥有此目录
  * 是超级用户

  目录/tmp和/var/spool/uucp/public是设置粘住位的候选者--这两个目录是任何用户都可以在其中创建文件的目录。这两个目录对任一用户的许可权通常都是读写和执行。 但是用户不能删除或更名属于其他人的文件，为此在这两个目录的文件方式中都设置了粘住位。
  
### 4.11 chown, fchown, fchownat和lchown函数
  chown函数可用于更改文件的用户ID和组ID.
```
#include <sys/types.h>
#include <unistd.h>

int chown(const char *pathname, uid_t owner, gid_t group);
int fchown(int fd, uid_t owner, gid_t group);
int lchown(const char *pathname, uid_t owner, gid_t group);
```
  除了所引用的文件是符号连接以外，这三个函数的操作相类似。 在符号连接情况下， lchown更改符号连接本身的所有者，而不是该符号连接被指向的文件。
  
  



### 4.12 文件尺寸
  stat结构的成员st_size包含了以字节为单位的该文件的长度。 此字段只对普通文件、目录文件和符号连接有意义。
  
  对于普通文件，其文件长度可以是0，在读这种文件时，将得到文件结束指示。
  对于目录，文件长度通常是一个数，例如16或512的整倍数。
  对于符号连接，文件长度是在文件名中的实际字节数。例如:
  lrwxrwxrwx 1 root       7 Sep 25 07:14 lib -> usr/lib
  
  其中，文件长度7就是路径名usr/lib的长度(注意，因为符号连接文件长度总是由st_size指示，所以符号连接并不包含通常C语言用作名字结尾的null字符。)
  
  SVR4和4.3+BSD也提供字段st_blksize和st_blocks. 第一个是对文件I/O较好的块长度，第二个是所分配的实际512字节块块数。 回忆一下3.9节，其中提到了当我们将st_blksize用于读操作时，读一个文件所需的最少时间量。为了效率的缘故，标准I/O库也试图一次读、写st_blksize字节。
  
  要知道，不同的unix版本其st_blocks所用的单位可能不是512字节块。使用此值并不是可移植的。
  
  文件中的空洞
  在前面我们提到普通文件可以包含空洞。前面3.2中用实例说明了下。 空洞时由超过文件结尾端的位移量设置，并写了某些数据后造成的。作为一个例子，考虑下列情况:
```
$ ls -l core
-rw-r--r-- 1 stevens 8483248 Nov 18 12:18 core
$ du -s core
272 core
```
  文件core的长度超过8M字节，而du命令则报告该文件所使用的磁盘空间总量是272个512字节块(139 264字节). 很明显，此文件有很多空洞。
  
  正如我们在3.6节中提及，read函数对于没有写过的字节位置都到的数据字节是0，如果执行wc -c core 可看到 8483248 core. 由此可见，正常的I/O操作读至整个文件长度。
  
  如果使用公共程序，例如cat(1)复制这种文件，那么所有这些空洞都被写成实际数据字节0.

### 4.13 文件截短
  有时候我们需要在文件尾端处截去一些数据以缩短文件。将一个文件的长度截短为0是一个特例， 用O_TRUNC标志可以做到这一点。为了截短文件可以调用函数truncate和ftruncate.
```
#include <unistd.h>
int truncate(const char *pathname, off_t length);
int ftruncate(int fd, off_t length);
```
  这两个函数将路径名为pathname或打开文件描述符filedes指定的一个现存文件的长度截短为length. 如果该文件以前的长度大于length, 则超过length以外的数据就不再能存取。如果以前的长度短于length, 则其后果与系统有关。 如果某个实现的处理是扩展该文件，则在以前的文件尾端和新的文件尾端之间的数据将读作0(也就是在文件中创建了一个空洞)。

### 4.14 文件系统
  为了说明文件连接的概念，先要对文件系统的结构有基本了解。 同时，了解i节点和指向一个i节点的目录项之间的区别也是很有益处的。
  目前有多种Unix文件系统的实现。例如，SVR4支持两种不同类型的磁盘文件系统:传统的Unix系统V文件系统(S5), 以及统一文件系统UFS. 
  
  我们可以把一个磁盘分成多个分区。每个分区可以包含一个文件系统。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/disk_range_fs.png)
  
  i节点是固定长度的记录项，它包含有关文件的信息。
```
在V7中，i节点占用64个字节，在4.3+BSD中，i节点占用128个字节。 在SVR4中，在磁盘上一个i节点的长度与文件系统的类型有关:
S5 i节点占用64字节，而UFS i节点占用128字节。
```
  如果要了解i节点的详情， 需要参考下图:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/inode.png)
  注意上图中的下面几个点:
  * 在图中有两个目录项指向同一i节点。每个i节点中都有一个连接计数，其值是指向该i节点的目录项数。只有当连接计数减少为0时，才可删除该文件(也就是可以释放该文件占用的数据块)。 这就是为什么"解除对一个文件的连接"操作并不总是意味着“释放该文件占用的磁盘块”的原因。 这也就是为什么删除一个目录项的函数被称之为unlink而不是delete的原因。 在stat结构中，连接计数包含在st_nlink成员中，其基本系统数据类型时nlink_t. 这种连接类型称之为硬连接。
  * 另外一种连接类型称为符号连接。 对于这种连接，该文件的实际内容(在数据块中)包含了该符号连接所指向的文件名字。符号连接的文件类型S_IFLNK.
  * i节点包含了所有与文件有关的信息: 文件类型、文件存取许可权位、文件长度和指向该文件所占用的数据块的指针等等。stat结构中的大多数信息都取自i节点。只有两项数据存放在目录项中: 文件名和i节点编号数。 i节点编号数的数据类型是ino_t.
  * 因为目录项中的i节点编号数指向同一文件系统中的i节点，所以不能使一个目录项指向另一个文件系统的i节点。这就是为什么ln(1)命令不能跨越文件系统的原因。
  * 当在不更改文件系统的情况下位一个文件更名时，该文件的实际内容并未移动，只需要构造一个指向现存i节点的新目录项，并删除老的目录项。 例如，为将/usr/lib/foo更名为/usr/foo, 如果/usr/lib和/usr在同一文件系统上，则文件foo的内容无需移动。这就是mv(1)命令的通常操作方式。
  
  我们说明了普通文件的连接计数的概念，但是对于目录文件的连接计数字段又如何呢? 假定我们在工作目录构建了一个新目录: mkdir testdir.
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/links_dirs.png)
  

### 4.15 link, linkat, unlink, unlinkat和remove函数
  前一节，我们看到一个文件可以由多个目录指向它的i-node。 我们可以使用link, linkat向给定的文件创建连接。
```
#include <unistd.h>
int link(const char *existingpath, const char *newpath);
int linkat(int efd, const char *existingpath, int nfd, const char *newpath, int flag);

/** both return 0 if OK, -1 on error
```
  这些函数创建新的目录入口newpath, 它引用存文件文件existingpath. 如果newpath已经存在， 返回错误。 只有newpath的最后一部分被创建。 其他的部分必须已经存在。
  
  linkat函数，原来存在的文件使用efd和existingpath来指定， 新的pathname是由nfd和newpath指定。默认情况下，如果文件描述符被设置为AT_FDCWD， 那么相应的pathname如果是相对路径，那么就根据当前目录计算相对路径。 如果路径pathname为绝对路径，那么相应的参数文件描述符被忽略掉。
  
  如果既存文件为符号连接，flag参数控制linkat函数是否创建一个连接到符号连接或者符号连接要指向的文件。如果flag设置为AT_SYMLINK_FOLLOW， 那么就会创建符号连接的连接。 如果flag is clear, 那么会给符号连接自己创建一个连接。
  
  新目录入口的创建和增加连接计数必须是原子操作。(回忆前面3.11原子操作的概念)
  
  很多实现需要两个pathname都在相同的文件系统中， 虽然POSIX.1允许支持不同文件系统的连接实现。如果实现支持创建目录的硬连接，那么仅仅限制超级用户可以创建。 这个限制的存在是因为硬连接可能会导致文件系统的循环， 文件系统的很多进程都无法处理这种循环。很多文件系统是心啊禁用硬连接也是出于这个原因。
  
  要删除既存目录入口， 我们可以调用unlink函数。
  
```
#include <unistd.h>
int unlink(const char *pathname);
int unlinkat(int fd, const char *pathname, int flag);
```

  这两个函数删除既存目录入口， 并递减pathname引用的文件计数。 如果还有其他连接连到该文件，可以通过其他入口来访问这些文件数据。 如果发生错误， 文件不变动。
  
  就像前面提到的， unlink一个文件， 我们必须有包含该目录入口(就是我们想要删除的目录入口)目录的写权限和执行权限。我们在4.10也提到过，如果粘住位设置在这个目录上， 那么必须具有这个目录的写权限， 并满足下面条件之一:
  * 拥有这个文件
  * 拥有该目录
  * 具有超级用户权限
  

  仅仅当连接数减少到0， 文件的内容才会被删除。 其他阻止文件内容被删除掉的条件: 只要一些有进程打开这个文件， 它的内容就不被删除。 当文件关闭， 内核首先检查有多少进程打开该文件。 如果达到0， 那么内核然后检查连接数量， 如果也为0， 那么这个文件的内容就会被删除掉。
  
  如果pathname参数是相对路径， 然么unlinkat函数根据fd文件描述符指定相应的目录来计算pathname的路径。 如果fd参数设置为AT_FDCWD, 那么pathname相对于调用进程的当前工作目录。如果pathname为绝对路径，那么忽略掉fd参数。
  
  flag参数给调用者一种改变unlinkat默认行文的机会。当设置为AT_REMOVEDIR标志，unlinkat函数可以用于删除目录， 类似于rmdir. 若flag为空，unlinkat操作类似unlink.
  
```
#include <fcntl.h>
#include "apue.h"


int main(void)
{
    if(open("tempfile", O_RDWR) < 0)
        err_sys("open error");

    if(unlink("tempfile") < 0)
        err_sys("unlink error");

    printf("file unlinked\n");

    sleep(15);
    printf("done\n");
    exit(0);
}
```
  运行上面的代码，可以看到， 进程尚未结束的时候，虽然调用了unlink，但是磁盘里边的空间尚未释放， 而此时tempfile已经不存在了。当进程结束， tempfile的内容才被清除， 空间得以释放。
  
  unlink的这个特性常常被程序用于清理自己创建的临时文件， 程序奔溃后，临时文件也不会保留在磁盘中。 进程创建文件使用open或creat， 然后立即使用unlink来删除它。 文件内容尚未删除， 因为进程还打开它。 只有当进程关闭或者终止的时候，这样才让内核来关闭它打开的文件， 文件内容也会被删除掉。
  
  如果pathname是一个符号连接， unlink删除符号连接， 而非符号连接引用的文件。没有给定符号连接来函数删除符号连接引用的文件的函数。
  
  超级用户能调用unlink使用pathname指定目录，如果系统支持， 但是函数rmdir应用于替代unlink来删除目录。 我们后面4.21会介绍rmdir函数。
  
  我们也可以使用remove函数来删除一个文件或目录。 对于文件，remove等价于unlink. 对于目录来说，remove等价于rmdir.
```
#include <stdio.h>
int remove(const char *pathname);
```

  
### 4.16 rename和renameat函数
  文件或目录可以使用rename, renameat函数来重命名。
```
#include <stdio.h>
int rename(const char *oldname, const char *newname);
int renameat(int oldfd, const char *oldname, int newfd, const char *newname);
```
  有几种条件需要针对这几个函数进行描述， 依赖oldname是否引用文件、目录还是符号连接。 我们也描述如果newname已经存在的情况。
  1. 如果oldname指定的是文件而非目录， 那么我们可以重命名文件或符号连接。 这种情况下，newname存在的话，它不能是目录。 如果newname存在并且不是目录，那么删除它，然后将oldname重命名为newname. 这时我们需要具有包含oldname和newname的目录的写权限, 既然我们需要两个目录都改变。
  2. 如果oldname指定的是目录， 那么我们是在重命名目录。 如果newname存在，它必须引用一个目录， 并且目录必须为空.(当我们说目录为空时，我们的意思是目录的入口只有.和..)。 如果newname存在，并且是空目录， 它被删除，然后oldname重命名为newname. 另外，当我们重命名目录时， newname不能包含路径oldname为前缀的名字。例如，我们不能将/usr/foo 重命名为/usr/foo/testdir, 因为， 旧的路径名/usr/foo是新路径名/usr/foo/testdir的路径前缀，不能被删除掉。
  3. 如果oldname或newname是符号连接引用， 那么连接本身被处理， 而非它们涉及的文件被处理。
  4. 我们不能重命名.和..。 更准确的说，.和..都不能出现在oldname和newname的最后一个部分中。
  5. 特别情况，如果oldname和newname都引用同一个文件， 那么该函数成功返回，但是什么也不做。
  
  如果newname已经存在，我们需要权限删除它。 同时，因为我们需要删除oldname的目录入口，可能创建newname的目录入口，我们需要包含oldname和newname的目录的写权限和执行权限。

  renameat函数提供的功能类似于rename函数。除了当oldname和newname都引用相对路径名。 如果oldname指定相对路径名， 它相对oldfd来计算路径名。newname同样相对于newfd。 oldfd, newfd都可以设置为AT_FDCWD, 根据当前目录计算相应目录路径名。
  
  
  
### 4.17 符号链接
  符号连接是文件的间接指针， 不想前面章节描述的硬连接， 硬连接是直接连向文件的i-node. 符号连接引入是相对硬连接的限制而来的。
  * 硬连接通常需要连接和文件在相同的文件系统中。
  * 只有超级用户才能创建目录的硬连接(当底层文件系统支持的话)
  
  符号连接没有它指向的文件系统的限制， 任何人都能创建到目录的符号连接。 符号连接一般用于"移动"文件或者整个目录的层级结构到另外一个系统的某个位置。

  当使用那些通过名称引用文件的函数时，我们总是需要知道函数是否遵循符号连接。如果函数遵循符号连接，函数的pathname参数通过符号连接引用文件指针。否则，pathname参数引用连接自己， 而非连接指向的文件。表4.17汇总了函数是否遵循符号连接。函数mkdir, mkdifo, mknod,rmdir不出现在这个表中， 因为它们在pathname是符号连接时会报错。函数接收一个文件描述符参数的函数，比如fstat和fchmode， 也没有列出来， 因为这些函数返回文件描述符(通常打开)来处理符号连接。 历史的原因，chown是否遵循符号连接的实现不相同。 现代所有的系统，chown不遵循符号连接。
```
函数      不遵循符号连接    遵循符号连接
------------------------------------------
access                           o
chdir                            o
chmod                            o
chown                            o
creat                            o
exec                             o
lchown          o
link                             o
lstat           o
open                             o
opendir                          o
pathconf                         o
readlink        o
remove          o
rename          o
stat                             o
truncat                          o
unlink          o
------------------------------------
```
  上表中有一个例外的情况，当open调用使用O_CREAT或O_EXCL设置调用时。 这种情况下，如果pathname引用符号连接，open会失败，设置errno为FEXIST. 这个行为是为了关闭一个安全漏洞，以便特权进程不能被欺骗向错误的文件写入东西。
  

### 4.18 创建和读取符号链接


### 4.19 文件的时间
  对每个文件保持有三个时间字段，它们的意义如下:
  * st_atime: 文件数据的最后存取时间, 比如read. 可以使用ls -lu查看
  * st_mtime: 文件数据的最后修改时间, 比如write. 可以使用ls -l查看
  * st_ctime: 文件i节点状态的最后修改时间， 比如chmod, chown命令。 可以使用ls -lc查看。

  注意，修改时间(st_mtime)和更改状态时间(st_ctime)之间的区别。 修改时间是文件内容最后一次被修改的时间。 更改状态时间是该文件的i节点最后一次被修改的时间。在本章中，我们已经说明了很多操作，它们影响到i节点，单并没有更改文件的实际内容:文件的存取许可权、用户ID、连接数等等。因为i节点中的所有信息都是与文件的实际内容分开存放的，所以，除了文件数据修改时间外，还需要更改状态时间。
  
  注意，系统并不保存对一个i节点最后一次存取时间，所以access和stat函数并不更改这三个时间中的任一个。
  
  系统管理源通常使用存取时间来删除在一定时间范围内没有存取过的文件。典型的例子就是删除在过去一周内没有存取过的名为a.out和core文件。find(1)常用来进行这种操作。
  
  修改时间和更改状态时间可以用来规定那个其内容已经被修改或其i节点已经被更改的那些文件。
  
  

### 4.20 futimens, utimensat和utimes函数

### 4.21 mkdir, mkdirat和rmdir函数

### 4.22 读取目录

### 4.23 chdir, fchdir和getcwd函数

### 4.24 特殊设备文件

### 4.25 sync和fsync函数

### 4.26 文件存取许可权位小节

### 4.27 小节
