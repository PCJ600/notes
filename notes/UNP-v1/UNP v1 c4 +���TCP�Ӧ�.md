## 基本TCP套接字编程

#### Socket函数

```C
#include <sys/socket.h>
int socket(int family, int type, int protocol);
```

**功能：**

​    为执行网络I/O，必须先调用socket函数，指定通信类型(v4,v6,TCP,UDP…)

**参数：**

* family指明协议族，常见值：AF_INET, AF_INET6

* type指明类型 常见值：SOCK_STREAM, SOCK_DGRAM

* protocal一般设为0，选择所给定的family和type组合的系统默认值

**返回值：**

​    成功时返回非负整数值，表示套接字描述符，即sockfd 

#### connect函数

TCP客户端用connect函数建立于TCP服务端的连接

```C
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);
```

**参数：**

​    Sockfd指socket函数返回的描述符，二、三参数必包含服务器IP和端口号。

**返回值：**

​    成功返回0，失败返回-1

**注意事项：**

客户调用connect前无需调bind, 内核会确定源IP，选择一个临时端口作源端口

TCP连接中connect出错的几种情况：

* 客户未收SYN响应，返回ETIMEOUT错

* 若对客户SYN响应为RST，表示主机在我们指定端口没有进程等待与之连接，此时立即返回ECONNREFUSED错

* Connect失败后，必须close当前的套接字描述符并重新调用socket

#### Bind函数

把一个本地协议地址赋予一个套接字，IP+端口号

```C
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
```

**返回值：**

​    成功返回0

IPV4用法 servaddr.sin_addr.s_addr = htonl(INADDR_ANY);

IPV6用法 serv.sin6_addr = in6addr_any

#### Listen函数

将未连接的socket转化为被动套接字，从CLOSED状态转为LISTEN状态

```C
#include<sys/socket.h>
int listen(int sockfd, int backlog); 
```

**返回值**

​    成功则为0， 出错则为-1

Backlog指定内核应为相应套接字排队的最大连接个数

(1)  未完成连接队列，SYN到达，但未完成三路握手SYN_RCVD

(2)  已完成连接队列，已完成三路握手，ESTABLISHED

三路握手完成，将未完成连接队列的项移到已完成连接队列对尾

Accept被调用时，已完成连接队列的对头项返回给进程，若该进程为空，进程投入睡眠。

当一个客户SYN到达，若队列为满，TCP忽略该分节，并不发送RST

原因:如发送RST，客户无法区别是该端口没有服务器监听，还是有监听，只是队列已满。

#### Accept函数

服务器调用，从已完成连接队列头返回下一个已完成连接，若队列空则睡眠

```c
#include<sys/socket.h>
int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen)
```

**参数**

Sockfd为监听套接字

Cliaddr, addrlen返回已连接的客户的地址

**返回值：**

Accept返回成功，其返回值为**内核生成的已连接套接字描述符**

**注意事项：**    

若对返回的客户协议地址不感兴趣，把cliaddr, addrlen置为空

#### Getsockname和getpeername函数

返回与某个套接字关联的本地协议地址(sockname)，或外地协议地址(peername)













