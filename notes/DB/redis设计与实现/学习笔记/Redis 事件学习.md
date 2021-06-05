## Redis事件

Redis服务器是一个事件驱动程序,服务器需处理以下两类事件:

* 文件事件
* 时间事件

单线程模型 + I/O多路复用技术



### 文件事件处理器

四个组成部分: 套接字 + I/O多路复用器 + 分派器(dispatcher) + 事件处理器(handler)

![文件事件](C:\PC\learning\notes\DB\redis设计与实现\学习笔记\文件事件.PNG)




**Reactor模式：**

https://cloud.tencent.com/developer/article/1488120

对于支持多连接的服务器，总结为2种fd+3种事件



2种fd:

listenfd: 一般只有一个，用来监听一个特定的端口(如80)

connfd: 每个连接都有一个connfd, 用来收发数据。

3种事件：

* listenfd进行accept监听，创建一个connfd
* 用户态，内核态copy数据，每个connfg对应2个应用缓冲区：readbuf, writebuf
* 处理connfd发来的数据，业务逻辑处理，准备response到writebuf





基于Redis源码实现一个简化的事件处理器

* 基于Reactor模型

* 只处理文件事件，不处理时间事件
* select, epoll分别实现





可读事件 -> connect, 客户端write, close

可写事件 -> 客户端read, 





API设计

void aeMain(aeEventLoop *eventLoop)



listenToPort -> 先创建监听套接字， 执行socket , bind, listen

为每个服务器套接字创建连接应答处理器，通过aeCreateFileEvent实现









































