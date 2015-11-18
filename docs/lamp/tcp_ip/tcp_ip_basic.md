#TCP/IP基础知识

## 2.5 TCP/IP分层模型与通信示例

### 2.5.1 数据包首部
```
+--------------+----------+-----------+------------+
| 以太网包首部 | IP包首部 | TCP包首部 |    数据    |
+--------------+----------+-----------+------------+
_              |<-----------以太网数据------------>|
_              +----------+-----------+------------+
_              | IP包首部 | TCP包首部 |    数据    |
_              +----------+-----------+------------+
_                         |<--------IP数据-------->|
_                         +-----------+------------+
_                         | TCP包首部 |    数据    |
_                         +-----------+------------+
_                                     |<-TCP数据-->|
```
  每个分层种，都会对所发数据附加一个首部，在这个首部中包含了该层必要的信息，如发送的目标地址以及协议相关信息。通常，为协议提供的信息为包首部，所要发送的内容为数据。 在下一层的角度来看，从上一层收到的包全部都被视为本层的数据。
  
```
  包、帧、数据报、段、消息
  以上五个术语都用来表述数据的单位，大致区分如下:
  包: 可以说是全能性术语。
  帧: 用来标识数据链路层中包的单位。 
  数据报: 是IP/UDP等网络层以上的分层中包的单位。 
  段: 则表示TCP数据流中的信息。
  消息: 则是指应用协议中数据的单位。
```

```
  包首部就像是协议的脸
  网络中传输的数据包由两部分组成: 一部分是协议所要用到的首部，另一部分是上层传过来的数据。
  首部的结构由协议的具体规范详细定义。例如，识别上一层协议的域应该从包的哪一位开始取多少个比特、
  如何计算校验和并插入包的哪一位等。相互通信的两端计算机如果在识别协议的序号以及校验和的计算方法上不一样，就根本无法实现通信。
  
  因此，在数据包的首部，明确标明了协议应该如何读取数据。反过来说，看到首部，也就能够了解该协议必要的信息以及所要处理的内容。
  因此，看到包首部就如同看到协议的规范。难怪有人说首部就像是协议的脸了。
```

### 2.5.2 发送数据包
  假设甲给乙发送电子邮件，内容为:“早上好”。而从TCP/IP通信上看，是从一台计算机A向另一台计算机B发送电子邮件。我们就通过这个例子来讲解一下TCP/IP通信的过程。
  
#### 1. 应用程序处理
  启动应用程序新建邮件，将收件人邮箱填好，再由键盘输入邮件内容"早上好", 鼠标点击发送按钮就可以开始TCP/IP通信了。
  
  首先，应用程序中会进行编码处理。例如，日文电子邮件使用ISO-2022-JP或UTF-8进行编码。这些编码相当于OSI的表示层功能。
  
  编码转换后，实际邮件不一定会马上被发送出去，因为有些邮件的软件又一次同时发送多个邮件的功能，也可能会有用户点击收信按钮以后才一并接收新邮件的功能。 像这种何时建立连接何时发送数据的管理功能，从某种宽泛的意义上看属于OSI参考模型中会话层的功能。
  
  应用在发送邮件的那一刻建立TCP连接，从而利用这个TCP连接发送数据。 它的过程首先是将应用的数据发送给下一层TCP,再作实际的转发处理。
  
#### 2. TCP模块的处理
  TCP根据应用的指示，负责建立连接，发送数据以及断开连接。 TCP提供将应用层发来的数据顺利发送至对端的可靠传输。
  
  为了实现TCP的这一功能，需要再应用层数据的前端附加一个TCP首部，TCP首部中包括源端口号和目标端口号(用以识别发送主机跟接收主机上的应用)、序号(用以发送的包中那部分是数据)以及校验和(用以判断数据是否被损坏)。随后将附加了TCP首部的包再发送给IP.
  
#### 3. IP模块的处理
  IP将TCP传过来的TCP首部和TCP数据合起来当作自己的数据，并在TCP首部的前端加上自己的IP首部。因此,IP数据报中IP首部后面紧跟着TCP首部，然后才是应用的数据数据首部和数据本身。IP首部中包含接收端IP地址以及发送端IP地址。紧随IP首部的还有用来判断其后面数据是TCP还是UDP的信息。
  
  IP包生成后，参考路由控制表决定接受此IP包的路由或主机。随后，IP包将被发送给连接这些路由器或主机网络接口的驱动程序，以实现真正发送数据。
  
  如果上不知道接收端MAC地址，可以利用ARP(Address Resolution Protocol)查找。只要知道了对端的mac地址，就可以将mac地址和IP交给以太网的驱动程序，实现数据传输。
  
#### 4. 网络接口(以太网驱动)的处理
  从IP传过来的IP包，对于以太网驱动来说不过就是数据。给这个数据附加上以太网首部并进行发送处理。 以太网首部中包含接收端MAC地址、发送端MAC地址以及标志以太网类型的以太网数据的协议。根据上述信息产生的以太网数据包通过物理层传输给接收端。发送吹中的FCS由硬件计算，添加到包的最后。设置FCS的目的是为了判断数据包是否由于噪声而被破坏。
  
### 2.5.3 经过数据链路的包

  分组数据报(简称包)经过以太网流动时，从前往后依次被附加了以太网包首部、IP包首部、TCP首部(或者UDP首部)以及应用自己的包首部和数据。而包的最后则追加了以太网包尾(Ethernet Tailer).
  每个包首部中至少都会包含两个信息:一个是发送端和接收端地址，另一个是上一层协议类型。
  
  经过每个协议分层时，都必须有识别包发送端和接收端的信息。以太网会用MAC地址，IP会用IP地址，TCP/UDP则会用端口号作为识别两段主机的地址。即使再应用程序中，像电子邮件这样的信息也是一种地址标识。这些地址信息都在每个包经由各个分层时，附加到协议对应的包首部里边。
  
```
|<----数据链路层---->|<------网络层------>|<--传输层->|<---会话层/表示层/应用层--->|<---数据链路层--->|
+------+------+------+------+------+------+-----+-----+----------------------------+------------------+
|接收端|发送端|以太网|发送端|接收端| 协议 | 源  |目标 |                            |                  |
| MAC  |  MAC | 类型 | IP   |  IP  | 类型 |端口 |端口 |            数据            |   循环冗余校验   |
| 地址 | 地址 |      | 地址 | 地址 |      | 号  | 号  |                            |                  |
+------+------+---.--+------+------+--.---+-----+--.--+----------------------------+------------------+
_                 |                   |            |      
_                 |                   |            |
|<------以太网头--+->|<----IP首部-----+-->|<-TCP/UDP->|<-----应用包头及首部------->|<-以太网Trailer-->|
_                 |          ^        |        ^ 头|               ^
_                 |          |        |        |   |               |
_                 +----------+        +--------+   +---------------+    (服务器端口号表示应用协议)
_                 表示首部的          表示首部的      表示首部的
_                 下一个协议          下一个协议      下一个协议

<---------------------------------------数据流动的方向------------------------------------------------
----------------------------------------数据处理的顺序----------------------------------------------->
```

  此外，每个分层的包首部中还包含一个识别位，它是标识上一层协议的种类信息。例如以太网的包首部中的以太网类型，IP中的协议类型以及TCP/UDP中两个端口的端口号都是起着识别协议类型的作用。就是在应用的首部信息中，有时也会包含一个用来识别其数据类型的标签。
  
### 2.5.4 数据包接收处理
  包的接收流程是发送流程的逆序过程。
#### 5. 网络接口(以太网驱动)的处理
  主机收到以太网包以后，首先从以太网首部找到MAC地址判断是否位发送给自己的包。如果不是发送给自己的包则丢弃数据。
  而如果接收到了恰好是发给自己的包，就查找以太网包首部中的类型从而确定以太网协议所传送过来的数据类型，如果这时不是IP而是其他诸如ARP的协议，就把数据传给ARP处理。总之，如果以太网首部的类型域包含了一个无法识别的协议类型，则丢弃数据。
  
#### 6. IP模块的处理
  IP模块收到IP包首部以及后面的数据部分以后，也做类似的处理。如果判断得出包首部中的IP地址与自己的IP地址匹配，则可接收数据并从中查找上一层的协议。如果上一层是TCP就将IP包首部之后的数据传给TCP处理；如果是UDP则将IP包首部后面的部分传给UDP处理。 对于由路由器的情况下，接收端地址往往不是自己的地址，此时，需要借助路由控制表，在调查应该送达的主机或路由器以后再转发数据。
  
#### 7. TCP模块的处理
  在TCP模块中，首先会计算以下校验和，判断数据是否被破坏。然后检查是否在按序号接收数据。最后检查端口号，确定具体的应用程序。
  数据接收完毕后，接收端则发送一个确认回执给发送端。如果这个回执信息未能达到发送端，那么发送端会认为接收端没有接收到数据而一直反复发送。
  数据被完整的接收以后，会传给由端口号识别的应用程序。
  
#### 8. 应用程序的处理
  接收端应用程序会直接接收发送端发送的数据。通过解析数据可以获知邮件的收件人地址是乙的地址。如果主机B上没有乙的邮件信箱，那么主机B返回给发送端一个无此收件地址的报错信息。
  
  但在这个例子中，主机B上恰好有乙的手贱香港，所以主机B和收件人乙能够收到电子邮件的正文。邮件会被保存到本机的磁盘上。如果保存也能正常进行，那么接收端会返回一个“处理正常”的回执给发送端。反之，一旦出现磁盘满，邮件未能成功保存等问题，就会发送一个处理异常的回执给发送端。
  由此，用户乙就可以利用主机B上的邮件客户端接收并阅读由主机A上的用户甲发送过来的电子邮件。
  
  