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

### 4.4 设置用户ID和组ID

### 4.5 文件访问权限

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
