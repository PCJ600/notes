# 基本UDP编程

#### 怎么理解UDP的无连接?

#### 怎么理解UDP的不可靠?

如一个客户数据报丢失，客户永远阻塞于recvfrom

如客户数据报到达服务器，但服务器的应答丢失，客户永远阻塞于recvfrom

#### UDP常见应用程序

DNS, NFS, SNMP(简单网络管理协议)

#### 掌握UDP服务器程序函数调用关系图



#### recvfrom和sendto函数

```C
#include <sys/socket.h>
ssize_t recvfrom(int sockfd, void *buff, size_t nbytes, int flags,
                struct sockaddr *from, socklen_t *addrlen);
ssize_t sendto(int sockfd, const void *buff, size_t nbytes, int flags,
               const struct sockaddr *to, socklen_t addrlen);
```

前三个参数等同于read和write函数三个参数：描述符，指向读入或写出的缓冲区的指针，读写字符数。

flags参数一般置0， 最后两个参数指示谁发送了数据报

返回值:读写的长度，写一个长度为0的数据报是可行的，UDP情况下，会形成一个只包含一个IP首部和一个8字节的UDP首部而无数据的报文。recvfrom返回0也是可以的，不像TCP返回0表示对端已关闭连接。

#### UDP回射程序客户端中recvfrom的第五、六个指针为空的风险

任何进程无论是与本客户进程相同的主机还是不同的主机上，都可以向客户的IP地址和端口发送数据报。

#### recvfrom返回的IP地址并非客户发送数据报的目的IP地址



#### 服务器进程未运行

客户端键入文本后，永远阻塞于recvfrom

在客户主机能够往服务器主机发送UDP数据报前，需要一次ARP请求和应答的交换

基本规则：对于一个UDP套接字，由它引发的异步错误并不返回，除非已连接

#### UDP的connect函数

除非套接字已连接，否则异步错误不会返回到UDP套接字。

UDP调用connect后，只是检查是否存在立即可知的错误(不可达目的地)，记录对端的IP地址和端口号。

已连接UDP套接字特点:

1.不能使用sendto指定目的地址，改用write和send

2.不必使用recvfrom获取数据报的发送者，改用read,recv,recvmsg

3.由已连接UDP套接字引发的异步错误会返回给它们所在进程

#### TCP和UDP是否可指定目的地协议地址

| 套接字类型 | write或send | 不指定目的sendto | 指定目的sendto |

| TCP | 可以 | 可以 | EISCONN |

| UDP，已连接 | 可以 | 可以 | EISCONN |

| UDP, 未连接 | EDESTADDREQ | EDESTADDREQ  | 可以 |

#### 给一个UDP套接字多次调用connect

目的：1. 指定新的IP地址和端口号 2.断开套接字

为了断开一个已连接UDP的套接字，再次调用connect把套接字地址结构的地址族设为AF_UNSPEC。调用connect进程断开套接字的连接。

#### 性能

未连接的UDP套接字上给两个数据报调用sendto函数涉及以下步骤：

连接套接字，输出第一个数据报，断开套接字连接

连接套接字，输出第二个数据报，断开套接字连接

已连接的套接字调用两次write涉及以下步骤：

连接套接字， 输出第一个数据报，输出第二个数据报

#### UDP缺乏流量控制



#### SO_REUSEADDR套接字选项







































