# I/O复用：select和poll函数

## 什么是I/O复用

进程需要一种预先告知内核的能力，使得内核一旦发现进程指定一个或多个I/O条件就绪，它就通知进程，这种能力叫I/O复用

## I/O复用典型场景

客户处理多个描述符

一个服务器既处理listenfd, 又处理connfd

一个服务器既要处理TCP，又要处理UDP

一个服务器处理多个服务或多个协议

## 五种I/O模型

阻塞式I/O,非阻塞式I/O,I/O复用,信号驱动式I/O,异步I/O

## 同步I/O和异步I/O对比

同步I/O导致请求进程阻塞，直到I/O操作完成

异步I/O不导致请求进程阻塞

## select函数

允许进程指示内核等待多个事件中的任何一个发生，并只有在有一个或多个事件发生或经历一段指定时间后才唤醒它。

``` C
#include<sys/select.h>
#include<sys/time.h>
int select(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout);
```

timeout参数，告知内核等待所指定描述符中任何一个就绪花多少时间。

fd_set三个参数指定让内核测试读、写、异常条件的描述符

maxfdp1值为最大描述符编号+1

注：描述符集内任何与未就绪描述符对应的位返回时均清成0



## 描述符就绪条件



### pselect函数的第六个参数



## poll函数























