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
  由Unix内核提供的基本时间服务是国际标准时间公元1970年1月1日00:00:00以来经过的秒数。这种秒数数据类型time_t表示的。我们称它们为日历时间。日历时间包括时间和日期。Unix在这方面与其他操作系统的区别: 
  * 以国际标准时间而非本地时间计时
  * 可自动进行转换，例如变换到夏日制
  * 将时间和日期作为一个量值保存。time函数返回当前时间和日期。
```
#include <time.h>
time_t time(time_t *calptr);
```
  时间值作为函数值返回。如果参数非null, 则时间值也存放在由calptr指向的单元内。
  一旦取得了这种以秒计的很大时间值后，通常要调用另一个时间函数将其变换为人们可读的时间和日期。 比如localtime, mktime, ctime, strftime都受到环境变量TZ的影响。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/times_relation.png)
  
  两个函数localtime和gmtime将日历时间变换成以年、月、日、时、分、秒、周日表示的时间，并将这些存放在一个tm结构中:

```
struct tm{
  int tm_sec; /** seconds after the minutes: [0-61] **/
  int tm_min; /** minutes after the hour: [0-59] **/
  int tm_hour; /** hours after midnight: [0, 23] **/
  int tm_mday; /** day of the month: [1, 31] **/
  int tm_mon; /** month of the year: [0, 11] **/
  int tm_year; /** years since 1900 **/
  int tm_wday; /** days since Sunday: [0-6] **/
  int tm_yday; /** days since January 1: [0, 365] **/
  int tm_isdst; /** daylight saving time flag: <0, 0, >0 **/
}
```
  秒可以超过59的原因是，可以表示闰秒。注意，除了月日字段，其他字段的值都以0开始。如果夏时制生效，则夏时制标志值为正；如果已非夏时制时间则为0； 如果此信息不可用，则为负。
```
#include <time.h>
struct tm *gmtime(const time_t *calptr);
struct tm *localtime(const time_t *calptr);
```
  localtime和gmtime之间的区别是:localtime将日历时间变换成本地时间，而gmtime则将日历时间变换成国际标准时间的年、月、日、时、分、秒、周日。
  函数mktime以本地时间的年月日等做为参数，将其变换成time_t值。
```
#include <time.h>
time_t mktime(struct tm *tmptr);
```
  asctime和ctime函数产生形式为26字节字符串，这与date(1)命令的系统默认输出形式类似:
  `Tue Jan 14 17:49:03 1992\n\0`
```
#include <time.h>
char *asctime(const struct *tmptr);
char *ctime(const time_t *calptr);
```
  asctime的参数是指向年月日等字符串的指针，而ctime的参数是指向日历时间的指针。
  最后一个时间函数是strftime, 它是非常复杂的printf类的时间值函数。
```
#include <time.h>
size_t strftime(char *buf, size_t maxsize, const char *format, const struct tm *tmptr);
```
  最后一个参数是要格式化的时间值，由一个指向年、月、日、时、分、秒、周日时间值的指针说明。格式化结果存放在长度为maxsize个字符的buf数组中，如果buf长度足以存放格式化结果以及一个null终止符，则该函数返回在buf中存放的字符数(不包括null终止符)，否则该函数返回0.
  
  format参数控制时间的格式。如同printf函数一样，变换说明的形式是%之后跟一个特定字符。 format的其他字符则按原样输出。两个连续的百分号在输出中产生一个百分号，与printf不同之处是，每个变化说明产生一个鼎昌输出字符串，在format字符串中没有字段宽度修饰符。 
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/strftime.png)
  
```
#include "apue.h"

#include <time.h>
#define TIME_STR_BUFF_SIZE 26

int
main(void)
{
    time_t tnow;
    time(&tnow);

    printf("now time string length is: %ld, and content is: %s" 
        , strlen(ctime(&tnow))
        , ctime(&tnow)
    );  

    struct tm *tmnow;
    tmnow = localtime(&tnow);

    printf("now time string length is: %ld, and content is: %s" 
        , strlen(asctime(tmnow))
        , asctime(tmnow)
    );  

    char buf[TIME_STR_BUFF_SIZE];
    strftime(buf, TIME_STR_BUFF_SIZE, "%a %b %d %Y", tmnow);
    printf("strftime tnow is: %s\n", buf);
}
```
  
### 6.11 总结
