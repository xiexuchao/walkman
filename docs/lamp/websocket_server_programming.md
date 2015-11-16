编写WebSocket服务器
=============
  一个WebSocket服务器说白了就是一个监听着协议规定的端口的TCP程序。可以用各种支持Berkeley Sockets的服务器编程语言来编写， 比如C(++)或Python, 甚至PHP和服务器端的javascript。 下面我们介绍下基本的编程思想， 具体语言就不介绍了。
  
第一步: WebSocket握手
-----------
  首先服务器必须通过标准的TCP套接字来监听所有外来的连接请求。 这个因平台而异。
  我们假设服务器运行在example.com上， 端口号8000， 而且能够响应/chart上的get请求。
  
### 客户端握手请求
  不管怎么设计，WebSocket握手请求都是由客户端发起， 因此必须知道如何解读客户端信息。 首先客户端会发送类似这样的连接请求(HTTP版本号最低为1.1, 而且请求方法必须为GET).
```
GET /chart HTTP/1.1
Host: example.com:8000
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

  
参考文献
==========
1. [编写WebSocket服务器](https://developer.mozilla.org/zh-CN/docs/WebSockets/Writing_WebSocket_servers)
