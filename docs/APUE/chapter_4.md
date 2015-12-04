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
  



### 4.12 文件尺寸

### 4.13 文件截取

### 4.14 文件系统

### 4.15 link, linkat, unlink, unlinkat和remove函数

### 4.16 rename和renameat函数

### 4.17 符号链接

### 4.18 创建和读取符号链接

### 4.19 文件的时间

### 4.20 futimens, utimensat和utimes函数

### 4.21 mkdir, mkdirat和rmdir函数

### 4.22 读取目录

### 4.23 chdir, fchdir和getcwd函数

### 4.24 特殊设备文件

### 4.25 sync和fsync函数

### 4.26 文件存取许可权位小节

### 4.27 小节
