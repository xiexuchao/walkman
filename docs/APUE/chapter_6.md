## 第六章 系统数据文件和信息

### 6.1 引言
  有很多操作系统需要使用一些与系统有关的数据文件，例如，口令文件/etc/passwd和组文件/etc/group就是经常由多种程序使用的两个文件。用户每次登录入Unix系统，以及每次执行ls -l命令时都要使用口令文件。
  
  由于历史原因，这些数据文件都是ASCII文本文件，并且使用标准I/O库读这些文件。但是对于较大的系统，顺序扫描口令文件变得很花费时间，我们想以非ASCII文本格式存放这些文件，但仍想应用程序提供一个可以处理任何一种文件格式的界面。对于这些数据文件的可移植性界面是本章的主题。 本章也包括了系统标识函数、时间和日期函数。

### 6.2 密码文件
  Unix口令文件包含了表6-1中所示的各字段，这些字段包含在<pwd.h>中国年定义的passwd结构中。
```
struct passwd {
  char *pw_name; /* user name */
  char *pw_passwd; /* encrypted password */
  uid_t pw_uid; /* user uid */
  gid_t pw_gid; /* user gid */
  __darwin_time_t pw_change;  /* password change time */
  char *pw_class; /* user access class */
  char *pw_gecos; /* Honeywell login info */
  char *pw_dir; /* home directory */
  char *pw_shell; /* default shell */
  __darwin_time_t pw_expire; /* account expiration */
};
表6-1:
成员        说明
pw_name     用户名
pw_passwd   加密密码
pw_uid      数值用户ID
pw_gid      数值组ID
pw_gecos    注释字段
pw_dir      初始化工作目录
pw_shell    初始shell(用户程序)
```
  

### 6.3 阴影口令

### 6.4 组文件

### 6.5 添加组ID

### 6.6 实现差异

### 6.7 其他数据文件

### 6.8 登录会计

### 6.9 系统标识

### 6.10 时间和日期例程

### 6.11 总结
