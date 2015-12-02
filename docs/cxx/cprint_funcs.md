## C语言printf系列函数
  C语言提供了各种`*print*`函数，都定义在stdio.h中:
```
1. 把结果写入到标准输出流stdout
int printf(const char *format, ...)   // util C99               
int printf(const char *restrict format, ...)   // since C99                      

2. 把结果写入到输出流stream中
int fprintf(FILE *stream, const char *format, ...) // util C99                           
int fprintf(FILE *restrict stream, const char *restrict format, ...) // since C99        

3. 把结果写入到字符串缓冲区中， 如果要写入的字符串长度超出给定buffer指定的数组长度，行为是未知的。
int sprintf(char *buffer, const char *format, ...)                // util C99            
int sprintf(char *restrict buffer, const char *restrict format, ...)  // since C99       

4. 把结果写入到字符串缓冲区中，至少写入bufsz - 1字符。结果字符串以\0终止，除非bufsz为0. bufsz为0，不写入任何东西
int snprintf(char *restrict buffer, int bufsz, const char *restrict format, ...)       // since C99

5. 
int printf_s(FILE *restrict stream, const char *restrict format, ...)              // since C99

6.
int fprintf_s(char *restrict buffer, rsize_t bufsz, const char *restrict format, ...)   // since C99

7.
int sprintf_s(char *restrict buffer, rsize_t bufsz, const char *restrict format, ...) // since C99

8.
int snprintf_s(char *restrict buffer, rsize_t bufsz, const char *restrict format, ...) // since C99
```
  从给定位置加载数据，将它们转换为等价的字符串中，然后将结果写入各种接受端。
  1. 将结果写到输出流stdout中
  2. 将结果写到输出流stream中
  3. 将结果输出到字符串缓冲中。对要写入的字符串(包括字符串结束符号\0)超出缓冲区数组指针的尺寸，行为是未定义的。
  4. 将结果写入字符串缓冲区中。至少写入bufsz - 1个字符。 结果字符串会以\0终止，除非bufsz为0. 如果bufsz为0， 不写入任何东西， buffer也就是一个空指针， 然而返回值(要写入的字节数目)仍然计算并返回。
  5. 5-8和1-4一样，除了运行时检查下面的错误，并调用当前安装的约束处理函数:
  <ul>
    <li>转换说明符%n出现在格式中</li>
  </ul>
  6. 



## 参考链接
* [CPP REFERENCE](http://en.cppreference.com/w/c/io/fprintf)
