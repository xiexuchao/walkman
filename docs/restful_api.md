RESTful API设计指南
===================

  网络应用程序，分为前端、后端两部分。 当前的发展趋势，就是前端设备层出不穷(手机、平板、桌面电脑、其他专用设备...)

  因此必须有一种统一的机制，方便不同的前端设备与后端进行通信。 这导致API架构的流行， 甚至出现API First的设计思想。RESTful API是目前比较成熟的一套互联网应用程序的API设计理论。
  
  
一. 协议
===========
  API与用户的通信协议，总是使用HTTPs协议。
  
二. 域名
===========
  应该尽量将API部署在专用域名之下。 eg: https://api.example.com
  
三. 版本(versioning)
===========
  应该将API版本号放入URL. https://api.example.com/v1/
  另外一种做法是将版本号放入HTTP头信息中， 但不如放入URL方便和直观。 github采用这种做法。

四. 路径(Endpoint)
===========
  路径又称"终点"(endpoint), 表示API但具体网址。
  在RESTful架构中，每个网址代表一种资源(resource), 所以网址中不能有动词， 只能有名词， 而且所有的名词往往与数据库的表名对应。 一般来说， 数据库中的表都是同种记录的集合(collection), 所以API中的名词也应该使用复数。
  
  举例来说，有一个API提供动物园(zoo)的信息， 还包括各种动物和雇员的信息， 则它的路径应该设计称下面这样:
```
  https://api.example.com/v1/zoos
  https://api.example.com/v1/animals
  https://api.example.com/v1/employees
```
