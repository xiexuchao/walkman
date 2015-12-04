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
  4. 

### 4.6 新文件和目录的所属关系

### 4.7 access和faccessat函数

### 4.8 umask函数

### 4.9 chmod, fchmod, fchmodat函数

### 4.10 粘住位(Sticky Bit)

### 4.11 chown, fchown, fchownat和lchown函数

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
