## 第三章 文件I/O

### 3.1 引言
  本章开始讨论Unix系统,先说明可用的文件I/O函数--打开文件、读文件、写文件等等。大多数Unix文件I/O只需用到5个函数:open, read, write, lseek以及close。然后说明不同缓存器长度对read和write函数的影响。
  本章所说明的函数经常被称之为不带缓存的I/O(unbuffered I/O, 与将在第5章中说明的标准I/O函数相对照)。术语--不带缓存指的是每个read和write都调用内核中的一个系统调用. 这些不带缓存的I/O函数不是ANSI C的组成部分,但是是POSIX.1和XPG3的组成部分。
  只要涉及在多个进程间共享资源,原子操作的概念就变成非常重要。我们将通过文件I/O和传送给open函数的参数来讨论此概念。并进一步讨 论在多个进程间如何共享文件,并涉及内核的有关数据结构。在讨论了这些特征后,将说明dup、fcntl和ioctl函数。

### 3.2 文件描述符
  对于内核而言,所有打开文件都由文件描述符引用。文件描述符是一个非负整数。当打开一个现存文件或创建一个新文件时,内核向进程返回一个文件描述符。当读、写一个文件时, 用open或creat返回的文件描述符标识该文件,将其作为参数传送给read或write.
  按照惯例,UNIX shell使文件描述符0与进程的标准输入相结合,文件描述符1与标准输出相结合,文件描述符2与标准出错输出相结合。这是UNIX shell以及很多应用程序使用的惯例, 而与内核无关。尽管如此,如果不遵照这种惯例,那么很多Unix应用程序就不能工作。
  在POSIX.1应用程序中,幻数0、1、2应被代换成符号常数STDIN_FILENO、STDOUT_FILENO和STDERR_FILENO。这些常数都定义在头文件<unistd.h>中。
  文件描述符的范围是0~OPEN_MAX(见表2-7)。早期的Unix版本采用的上限值是19(允许每个进程打开20个文件),现在很多系统则将其增加至63。
  
### 3.3 open和openat函数
  调用open函数可以打开或创建一个文件。
```
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int open(counst char * pathname, int oflag, .../*, mode_t mode */);
```
  我们将第三个参数写为..., 这是ANSI C说明余下参数的数目和类型可以变化的方法。对于open函数而言，仅当创建文件时才使用第三个参数。在函数原型中此参数放置在注释中。
  * pathname: 要打开或创建的文件名字
  * oflag: 用来说明此函数的多个选择项。用下列一个或多个常量进行或运算构成oflag参数(这些常量都定义在fcntl.h头文件中)：
  <ul>
    <li>O_RDONLY: 只读打开</li>
    <li>O_WRONLY: 只写打开</li>
    <li>O_RDWR: 读写打开</li>
    <li>很多实现将O_RDONLY定义为0，O_WRONLY定义为1，O_RDWR定义为2，以与早期的系统兼容</li>
    <li>O_APPEND: 每次写时都加到文件的尾端。</li>
    <li>O_CREAT: 此文件不存在则创建它。使用此选项时，需同时说明第三个参数mode, 用其说明该文件的存取许可权位。</li>
    <li>O_EXCL: 如果同时指定了O_CREAT，而文件已经存在，则出错。这可测试一个文件是否存在，如果不存在则创建此文件成为一个原子操作。</li>
    <li>O_RTRUNC: 如果此文件存在，而且为只读或只写成功打开，则将其长度截短为0</li>
    <li>O_NOCTTY: 如果pathname指的是终端设备，则不将此设备分配作为此进程的控制终端。</li>
    <li>O_NONBLOCK: 如果pathname指的是一个FIFO、一个块特殊文件或一个字符特殊文件，则此选项为此文件的本次打开操作和后续的I/O操作设置非阻塞方式。</li>
    <li>O_SYNC: 使每次write都等到物理I/O操作完成。</li>
  </ul>
  
  由open返回的文件描述符一定是最小的未用描述符数字。这一点被很多应用程序用来在标准输入、标准输出或标准出错上打开一个新的文件。 例如，一个应用程序可以先关闭标准输出,然后打开另一个文件，实现就能了解到该文件一定会在文件描述副1上打开。 在3.12节说明dup2函数时，可以了解到有更好的方法来保证在一个给定的描述符上打开一个文件。
  文件名和路径名截短
  如果NAME_MAX是14，而我们却试图在当前目录中创建一个其文件名包含15个字符的新文件，此时会发生什么呢? 按照传统，早期的系统V版本，允许这种使用方法，但是总是将文件名截短为14个字符，而BSD类的系统则返回出错ENAMEETOOLONG. 这一问题不仅仅与创建新文件有关。如果NAME_MAX为14，而存在一个其名为恰恰就是14个字符的文件，那么以pathname作为其参数的任一函数(open, stat等)都会遇到这一问题。

  在POSIX.1中，常数_POSIX_NO_TRUNC决定了是否要截短过长的文件名或路径名，或者返回一个出错。第12章将说明此值可以针对各个不同的文件系统进行变更。
  
  若_POSIX_NO_TRUNC有效，则在整个路径名超过PATH_MAX,或路径名中的任一个文件名超过NAME_MAX时，返回ENAMEETOOLONG.

### 3.4 creat函数
  也可以使用creat函数创建一个新文件。
```
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int creat(counst char * pathname, mode_t mode);
```
  creat的不足之处是它以只写方式打开所创建文件。在提供open的新版本之前，如果要创建一个临时文件，并要先写入该文件，然后又读该文件，则必须先调用creat, close,然后再调用open. 现在则可使用下列方式调用open:
  open(pathname, O_RDWR|O_CREAT|O_TRUNC, mode)
### 3.5 close函数
```
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int close(int filedes);
```
  关闭一个文件时也释放该进程加在该文件上的所有锁记录。
  当一个进程终止时，它所有的打开文件都由内核自动关闭。很多程序都使用这一功能而不显示地用close关闭打开的文件。

### 3.6 lseek函数
  每个打开的文件都有一个与其关联的当前文件偏移量。它是一个非负整数，用以度量从文件开始处计算的字节数。通常，读写操作都从当前文件位置偏移量处开始，并使位移量增加所读或所写的字节数。按系统默认，当打开一个文件时，除非指定O_APPEND选择项，否则该位移量被设置为0.
```
#include <sys/types.h>
#include <unistd.h>

off_t lseek(int filedes, off_t offset, int whence);
```
  lseek成功返回新文件位移，若出错返回-1.
  对参数offset, whence的值有关。
  * 若whence是SEEK_SET, 则该文件的位移量设置为距文件开始处offset个字节。
  * 若whence是SEEK_CUR, 则该文件的位移量设置为当前值加offset, offset可正可负。
  * 若whence是SEEK_END, 则该文件的位移量设置为文件长度加offset, offset可正可负。
  
  若lseek成功执行，则返回新的文件位移量，为此可以用下面的方式确定一个打开文件的当前位移量:
```
off_t currpos;
currpos = lseek(fd, 0, SEEK_CUR);
```
  这种方法也可用来确定所涉及的文件是否可以设置位移量。如果文件描述符引用的是一个管道或FIFO, 则lseek返回-1,并将errno设置为EPIPE.
```
  三个符号常数SEEK_SET, SEEK_CUR, SEEK_END是由系统V引进的。
  在系统V之前，whence被指定为0(绝对偏移量),1(相对于当前位置的偏移量)，2(相对文件尾端的位移量)。
  很多软件仍直接使用这些数字进行编码。
  
  在lseek中字符l表示长整型。在引入off_t数据类型之前，offset参数和返回值是长整型。 
  lseek是由V7引进的，当时C语言中增加了长整形。(在V6中，用函数seek和tell提供类似功能)
```

```
#include "apue.h"
#include <sys/types.h>

int main(void)
{
    if(lseek(STDIN_FILENO, 0, SEEK_CUR) == -1) 
        printf("cannot seek\n");

    else
        printf("seek ok\n");

    exit(0);
}
```
  普通文件允许lseek, 其位移量必须是非负数，但是，某些设备可能允许负的位移量。但对于普通文件，其位移量必须是非负数。因为位移量可能是负值，所以在比较lseek的返回值时应当谨慎，不要测试它是否小于0，而要测试是否等于-1.
```
在80386上运行的SVR4支持/dev/kmem设备，它可以具有负的偏移量。因为位移量off_t时带符号数据类型，所以文件的最大长度减少一半。
例如，若off_t是32位整型，则文件最大长度是2^31字节。
```

  lseek仅将当前的文件偏移量记录在内核中，它并不引起任何I/O操作。然后，该位移量用于下一个读写操作。
  
  文件偏移量可以大于文件的当前长度。在这种情况下，对该文件的下一次写将延长该文件，并在文件中构成一个空洞，这一点是允许的。位于文件中但没有写过的字节都被读为0.
```
#include "apue.h"
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

char buf1[] = "abcdefghij";
char buf2[] = "ABCDEFGHIJ";

int main(void)
{
    int fd; 
    if((fd = creat("file.hole", FILE_MODE)) < 0)
        err_sys("creat error");

    if(write(fd, buf1, 10) != 10) 
        err_sys("buf1 write error");
    /** offset now = 10 **/

    if(lseek(fd, 40, SEEK_SET) == -1) 
        err_sys("lseek error");
    /** offset now = 40 **/

    if(write(fd, buf2, 10) != 10) 
        err_sys("buf2 write error");
    /** offset now = 50 **/

    exit(0);
}
```
  编译，运行程序，然后可以看到如下结果:
```
bogon:io apple$ cat file.hole 
abcdefghijABCDEFGHIJbogon:io apple$ 
bogon:io apple$ od -c file.hole 
0000000    a   b   c   d   e   f   g   h   i   j  \0  \0  \0  \0  \0  \0
0000020   \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0
0000040   \0  \0  \0  \0  \0  \0  \0  \0   A   B   C   D   E   F   G   H
0000060    I   J                                                        
0000062
```
  使用od(1)命令观察该文件的实际内容。命令行中的-c标志表示以字符的方式打印文件内容。从中可以看到，文件中间的30个未写字节都被读成0. 每一行开始的一个七位数都是以八进制形式表示的字节位移量。
  
#### [od命令简介](http://www.ualberta.ca/dept/chemeng/AIX-43/share/man/info/en_US/a_doc_lib/cmds/aixcmds4/od.htm)
```
usage: od [-aBbcDdeFfHhIiLlOosvXx] [-A base] [-j skip] [-N length] [-t type]
          [[+]offset[.][Bb]] [file ...]
```

### 3.7 read函数
  用read函数从打开文件中读数据。
```
#include <unistd.h>
ssize_t read(int filedes, void *buf, size_t nbytes);
```
  如read成功，则返回读到的字节数。如已达到文件的尾端，则返回0.
  很多情况可使实际都到的字节数少于要求读的字节数:
  * 读普通文件时，在读到要求字节数之前已经达到了文件尾端。 例如，若在到达文件尾端之前还有30个字节，而需要读100个字节，则read返回30，下一次再调用read时，它将返回0(文件尾端)。
  * 当从终端设备读时，通常一次最多读一行。
  * 当从网络读时，网络中的缓冲机构可能造成返回值小于所要求读的字节数。
  * 某些面向记录的设备，例如磁带，一次最多返回一个记录。
  
  读操作从文件的当前位移量处开始，在成功返回之前，该位移量增加实际读的字节数。
  POSIX.1在几个方面对此函数原型做了更改。 其经典定义是:
```
  int read(int filedes, char *buff, unsigned nbytes);
```
  首先，为了与ANSI C一致，其第二个参数由char *改为void *. 在ANSI C中， 类型void *用于表示类属指针。其次，其返回值必须是一个带符号的整数(ssize_t), 以返回正字节数、0表示文件尾端，-1表示出错。 最后，第三个参数在历史上是一个不带符号整数，以允许一个16位的实现可以一次读或写65534个字节。 在1990 POSIX.1标准中，引进了新的基本系统数据类型ssize_t以提供带符号的返回值，size_t则被用于第三个参数。
  

### 3.8 write函数

### 3.9 I/O效率

### 3.10 文件共享

### 3.11 原子操作

### 3.12 dup和dup2函数

### 3.13 sync, fsync和fdatasync函数

### 3.14 fcntl函数

### 3.15 ioctl函数

### 3.16 /dev/fd

### 3.17 总结
