## 第16章 网络IPC: Sockets

### 16.1 引言
  上一章我们考察了各种Unix系统所提供的经典进程间通信机制(IPC): 管道、FIFO、消息队列、信号量以及共享存储。这些机制允许在同一台计算机上运行的进程可以相互通信。本章将考察不同计算机(通过网络连接)上的进程相互通信的机制:网络间通信(network IPC).
  
  在本章中，我们将描述套接字网络进程间通信接口，进程用该接口能够和其他进程通信，无论它们是在同一台计算机还是在不同的计算机上。实际上，这正是套接字接口设计目标之一:同样的接口可以用于计算机间通信，也可用于计算机内通信。尽管套接字接口可以采用许多不同的网络协议进行通信，但本章的讨论限制在因特网事实上的通信标准:TCP/IP协议栈。
  
  POSIX.1中指定的套接字API是基于4.4BSD套接字接口的。尽管这些年套接字接口有些细微的变化，但是当前的套接字接口与20世纪80年代早期4.2BSD所引入的接口很类似。
  
  本章仅是一个套接字API的概述。

### 16.2 Socket描述符
  套接字是通信端点的抽象。正如使用文件描述符访问文件，应用程序使用套接字描述符访问套接字。套接字描述符在Unix系统中被当作是一种文件描述符。事实上，许多处理文件描述符的函数(如read/write)都可以处理套接字描述符。
```
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
```
  参数domain去定通信的特性，包括地址格式。16.1图总结了由POSIX.1指定的各个域。各个域都有自己标识地址的格式，而表示各个域的常数都以AF_开头，意指地址族(address family).
  
  我们将在17.2节讨论Unix域。大多数系统还定义了AF_LOCAL域，这是AF_UNIX的别名。AF_UNSPEC域可代表任何域。历史上，有些平台支持其他网络协议，如AF_IPX域代表的NetWare协议族，但这些协议的域常数没有被POSIX.1标准定义。
```
图 16.1
--------------------------
域          描述
--------------------------
AF_INET     IPv4因特网域
AF_INET6    IPv6因特网域
AF_UNIX     Unix域
AF_UNSPEC   未指定
```
  参数type确定套接字的类型，进一步确定通信特征。下图总结了由POSIX.1定义的套接字类型，但在实现中可以自由添加其他类型的支持。
```
----------------------------------------------------------------------------
类型            描述
----------------------------------------------------------------------------
SOCK_DGRAM      固定长度的、无连接的、不可靠的报文传递
SOCK_RAW        IP协议的数据报接口(在POSIX.1中是可选的)
SOCK_SEQPACKET  固定长度的、有序的、可靠的、面向连接的报文传递
SOCK_STREAM     有序的、可靠的、双向的、面向连接的字节流
----------------------------------------------------------------------------
```
  参数protocol通常是0，表示为给定的域和套接字类型选择默认协议。当对同一个域和套接字类型支持多个协议时，可以使用protocol选择一个特定协议。在AF_INET通信域中，套接字类型SOCK_STREAM的默认协议是传输控制协议(TCP). 在AF_INET通信域中，套接字类型SOCK_DGRAM的默认协议是UDP. 下面列出了为因特网域套接字定义的协议:
```
----------------------------------------------------------------------------
协议          描述
----------------------------------------------------------------------------
IPPROTO_IP    IPv4网际协议
IPPROTO_IPV6  IPv6网际协议(在POSIX.1中为可选)
IPPROTO_ICMP  因特网控制报文协议(Internet Control Message Protocol)
IPPROTO_RAW   原始IP数据包协议(在POSIX.1中为可选)
IPPROTO_TCP   传输控制协议
IPPROTO_UDP   用户数据报协议(User Datagram Protocol)
```

  对于数据报(SOCK_DGRAM)接口，两个对等进程之间通信时不需要逻辑连接。只需要向对等进程所使用的套接字送出一个报文。
  因此数据包提供了一个无连接的服务。另一方面，字节流(SOCK_STREAM)需要在交换数据之前，在本地套接字和通信的对等进程套接字之间建立一个逻辑连接。
  
  数据报是自包含报文。发送数据报近似于给某人邮寄信件。你能邮寄很多新，但你不能保证传递的次序，并且可能有些信件会丢失在路上。每封信件包含接受者地址，使这封信件独立于所有其他信件。每封信件可能送达不同的接受者。
  相反，使用面向连接的协议通信就像与对方打电话。首先，需要通过电话建立一个连接，连接建立好之后，彼此能双向的通信。每个连接是端到端的通信链路。对话中不包含地址信息，就向呼叫两端存在一个点对点虚拟连接，并且连接本身按时特定的源和目的地。
  
  SOCK_STREAM套接字提供字节流服务，所以应用程序分辨不出报文的界限。这意味着从SOCK_STREAM套接字读数据时，它也许不会返回所有由发送进程所写的字节数。最终可以获得发送过来的所有数据，但也许要通过若干次函数调用才能得到。
  
  SOCK_SEQPACKET套接字和SOCK_STREAM套接字很类似，只是从该套接字得到的是基于报文的服务而不是字节流服务。这意味着从SOCK_SEQPACKET套接字接收的数据量与对方所发送的一致。流控制传输协议(SCTP)提供了因特网域上的顺序数据报服务。
  
  SOCK_RAW套接字提供一个数据报接口，用于直接反问下面的网络层(即因特网域中的IP层)。使用这个接口时，应用程序负责构造自己的协议头部，这是因为传输协议(如TCP和UDP)被绕过了。的那个创建一个原始套接字时，需要有超级用户特权，这样可以防止恶意应用程序绕过内奸安全机制来创建报文。
  
  调用socket与调用open相类似。在两种情况下，均可获得用于I/O的文件描述符。当不再需要该文件描述符时，调用close来关闭对文件或套接字的访问，并且释放该描述符以便重新使用。
  
  虽然套接字描述符本质上是一个文件描述符，但不是所有参数为文件描述符的函数都可以接受套接字描述符。 下图总结了到目前为止所讨论的大多数以文件描述符为参数的函数使用套接字描述符的行为。未指定和由实现定义的行为通常意味着该函数对套接字描述符无效。例如lseek不能以套接字描述符为参数，因为套接字不支持文件偏移量的概念。
```
---------------------------------------------------------------
函数                        使用套接字描述符时的行为
---------------------------------------------------------------
close                       释放套接字
dup和dup2                   和一般文件描述符一样复制
fchdir                      失败，并且将errno设置为ENOTDIR
fchmod                      未指定
fchown                      由实现定义
fcntl                       支持一些命令，包括F_DUPFD, F_DUPFD_CLOEXEC, F_GETFD, F_GETFL, F_GETOWN, F_SETFD, F_SETFL和F_SETOWN.
fdatasync, fsync            由实现定义
fstat                       支持一些stat结构成员，但如何支持由实现定义
ftruncate                   未指定
ioctl                       支持部分命令，依赖于底层设备驱动
lseek                       由实现定义(通常失败会将errno设置为ESPIPE)
mmap                        未指定
poll                        正常工作
pread,pwrite                失败时会将errno设置为ESPIPE
read,readv                  与没有任何标志位的recv等价
select                      正常工作
write,writev                与没有任何标志位的send等价
```
  套接字通信是双向的。可以采用shutdown函数来禁止一个套接字的I/O.
```
#include <sys/socket.h>
int shutdown(int sockfd, int how);
```
  如果how是SHUT_RD(关闭读端)，那么无法从套接字读取数据。如果how是SHUT_WR(关闭写端)，那么无法使用套接字发送数据。如果how是SHUT_RDWR, 则既无法读取数据，又无法发送数据。能够关闭一个套接字，为何还使用shutdown呢?这里有若干理由。首先，只有最后一个活动引用关闭时，close才释放网络端点。这意味着如果复制一个套接字(如采用dup),要知道关闭了最后一个引用它的文件描述符才会释放这个套接字。而shutdown允许使一个套接字处于不活动状态，和引用它的文件描述符数目无关。其次，有时可以很方便的关闭套接字双向传输中的一个方向。例如，如果想让所通信的进程能够确定数据传输何时结束，可以关闭该套接字的写端，然后通过该套接字读端仍可以继续接受数据。

### 16.3 寻址
  上一节学习了如何创建和销毁一个套接字。在学习用套接字做一些有意义的事情之前，需要知道如何标识一个目标通信进程。进程标识有两部分组成。一部分是计算机的网络地址，她可以帮助标识网络上我们想与之通信的计算机。另一部分是该计算机上用端口号表示的服务，她可以帮助标识特定的进程。
  
#### 16.3.1 字节序
  与同一台计算机上的进程进行通信时，一般不用考虑字节序。字节序时一个处理器架构特性，用于指示像整数这样的大数据类型内部的字节如何排序。图16-5显示了一个32位整数中的字节序是如何排序的。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/bytes_order_in32bit_system.png)
  
  如果处理器架构支持大端字节序，那么最大字节地址出现在最低有效字节上。小端字节序则相反:最低有效字节包含最小字节地址。注意，不管字节如何排序，最高有效字节总在左边，最低有效字节总在右边。因此，如果想给一个32位整数赋值0x04030201,不管字节序如何，最高有效字节都将包含4，最低有效字节都将包含1.
````
max ----------------> 0 有效字节
+-+-+-+-+-+-+-+-+-+-+-+
最大有效字节------->最低有效字节

大端字节序: 最大字节地址出现在最低有效字节上
小端字节序: 最低字节地址出现在最低有效字节上
````
  如果接下来，想将一个字符指针(cp)强制转换到这个整数地址，就会看到字节序带来的不同。在小端字节序的处理器上，cp[0]指向最低有效字节因而包含1，cp[3]指向最高有效字节因而包含4. 而在大端字节的处理器上，cp[0]指向最高有效字节因而包含4，cp[3]指向最低有效字节因而包含1. 下面是本文讨论的4种平台的字节序.
```
------------------------------------------------------------------
操作系统      处理器架构            字节序
------------------------------------------------------------------
FreeBSD 8.0   Intel Pentium         小端
Linux 3.2.0   Intel Core i5         小端
Mac OS X 10.6.8 Intel Core 2 Duo    小端
Solaris 10    Sun SPARC             大端
```
> 有些处理器可以配置成大端，也可以配置成小端，因而使问题变得更让人困惑。

  网络协议指定了字节序，因此异构计算机系统能够交换协议信息而不会被字节序所混淆。TCP/IP协议栈使用大端字节序。应用程序交换格式化数据时，字节序问题就会出现。对于TCP/IP，地址用网络字节序来表示，所以应用程序有时需要在处理器的字节序与网络字节序之间转换它们。例如，以一种易读的形式打印一个地址时，这种转换很常见。
  
  对于TCP/IP应用程序，有4个用来在处理器字节序和网络字节序之间实施转换的函数。
```
#include <arpa/inet.h>
uint32_t htonl(uint32_t hostint32); /** 处理器字节序转网络字节序 长整型32位 **/
unit16_t htons(uint16_t hostint16); /** 处理器字节序转网络字节序 short类型16位 **/
uint32_t ntohl(uint32_t netint32); /** 网络字节序转处理器字节序 32位 **/
uint16_t ntohs(uint16_t netint16); /** 网络字节序转处理器字节序 16位 **/
```

  h表示主机字节序，n表示网络字节序。l表示长整型(4字节)，s表示短整型(2字节)。 虽然在使用这些函数时包含的是<arpa/inet.h>头文件，但系统实现经常是在其他头文件种声明这些函数的，指示这些头文件都包含在arpa/inet.h中。对于系统来说把这些函数实现为宏也是很常见的。
  
#### 16.3.2 地址格式
  一个地址标识一个特定通信域的套接字端点，地址格式与这个特定的通信域相关。为了使不同格式地址能够传入到套接字函数，地址会被强制转换成一个通用的地址结构sockaddr:
```
struct sockaddr{
  sa_family_t sa_family; /** address family **/
  char sa_data[]; /** variable-length address **/
  ...
}
```
  套接字实现可以自由地添加额外地成员并且定义sa_data成员地大小。例如，在Linux中，该结构定义如下:
```
struct sockaddr{
  sa_family_t sal_family; /** address family **/
  char sa_data[4]; /** variable-length address **/
}
```
  但是在FreeBSD中，该结构定义如下:
```
struct sockaddr{
  unsigned char sa_len; /** total length **/
  sa_family_t sa_family;
  char sa_data[4];
}
```

  因特网地址定义在<netinet/in.h>头文件中。在IPv4因特网域(AF_INET)中，套接字地址用结构sockaddr_in表示:
```
struct in_addr {
  in_addr_t s_addr; /** IPv4 address **/
}

struct sockaddr_in {
  sa_family_t sin_family; /** address family **/
  in_port_t sin_port; /** port number **/
  struct in_addr sin_addr; /** IPv4 address **/
}
```
  数据类型in_port_t定义成uint16_t.数据类型in_addr_t定义成uint32_t. 这些整数类型在<stdint.h>中定义并指定了相应地位数。
  
  与AF_INET域相比较，IPv6因特网域(AF_INET6)套接字地址用结构sockaddr_in6表示。
```
struct in6_addr{
  uint8_t s6_addr[16]; /** ipv6 address **/
}

struct sockaddr_in6{
  sa_family_t sin6_family; /** address family **/
  in_port_t sin6_port; /** port number **/
  uint32_t sin6_flowinfo; /** traffic class and flow info **/
  struct in6_addr sin6_addr; /** IPv6 address **/
  uint32_t sin6_scope_id; /** set of interfaces for scope **/
}
```

  这些都是Single Unix Specification要求地定义。每隔实现可以自由添加更多字段。例如，在Linux中，sockaddr_in定义如下:
```
struct sockaddr_in {
  sa_family_t     sin_family;
  in_port_t       sin_port;
  struct in6_addr sin6_addr;          /** ipv4 address **/
  unsigned char   sin_zero[8];        /** filter **/
}
```
  其中成员sin_zero为填充字段，应该全部被设置为0.
  
  注意，尽管sockaddr_in与sockaddr_in6结构相差比较大，但它们均被强制转换成sockaddr结构输入到套接字例程中。在17.2节，将会看到Unix域套接字地址地结构与上述两个因特网套接字地址格式地不同。
  
  有时，需要打印出能被人理解而不是计算机所能理解地地址格式。BSD网络软件包含函数inet_addr和inet_ntoa, 用于二进制地址格式与点分十进制字符表示之间地相互转换。 但是这函数仅适用于IPv4地址。有两个新函数inet_ntop和inet_pton具有xiangsi功能，而且同时支持IPv4和IPv6地址。
  
```
#include <arpa/inet.h>
const char *inet_ntop(int domain, const void *restrict addr, char *restrict str, socklen_t size);
int inet_pton(int domain, const char *restrict str, void *restrict addr);
```
  函数inet_ntop将网络字节序的二进制地址转换成文本字符串格式。inet_pton将文本字符串格式转换成网络字节序的二进制地址。参数domain仅支持两个值: AF_INET和AF_INET6. 
  
  对于inet_ntop, 参数size指定了保存文本字符串的缓冲区str的大小。


### 16.4 连接确立

### 16.5 数据传输

### 16.6 Socket选项

### 16.7 Out-of-Band数据

### 16.8 非阻塞和异步I/O

### 16.9 总结

