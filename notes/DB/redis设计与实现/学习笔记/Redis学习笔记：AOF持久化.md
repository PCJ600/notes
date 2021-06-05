# Redis学习笔记（九）：AOF持久化

[TOC]

### 简介

Redis提供了AOF(append only file)持久化功能，通过保存服务器执行的写命令的方式记录数据库状态。本文介绍如下内容：

* AOF持久化的实现 (命令追加、文件写入、AOF重写、AOF后台重写)
* 如何通过AOF文件还原数据库
* AOF持久化的配置选项
* AOF和RDB两种持久化方式的比较



### 1. AOF持久化的实现

以下介绍AOF持久化的实现方式，内容分别如下：

* 命令追加

* 文件写入和同步 
* AOF重写
* AOF后台重写

#### 1.1 命令追加

AOF功能开启后，每当服务器执行完一条写命令，这条写命令就会以协议格式追加到服务器状态的aof_buf缓冲区中。

```C
struct redisServer {
	sds aof_buf;
	// ...
};
```

例如，客户端向服务端发送`set number 1`命令，服务器执行完SET后，会将如下协议内容追加到aof_buf缓冲区

*3\r\n$3\r\nset\r\n$6\r\n\number\r\n\$1\r\n\1\r\n

其中，\r\n表示换行符，我们打开对应的AOF文件，可以看到文件末尾追加了如下内容：

```
*3
$3
set
$6
number
$1
1
```

#### 1.2 文件写入和同步

**问题：**aof_buf缓冲区在内存中，它是在什么时间点，以何种策略被写入到AOF文件？

Redis服务器是一个事件驱动的程序，主进程就是一个事件循环(参考aeMain函数)，负责处理两类事件：文件事件、时间事件。

服务器处理文件事件时可能会执行写命令，这使得相应的协议内容被追加到aof_buf缓冲区，**因此服务器在结束一个事件循环前，会调用flushAppendOnlyFile函数，考虑是否将aof_buf缓冲区的内容写入到AOF文件。**

以Redis 6.0版本的源码为例，事件主循环aeMain的实现如下：

```C
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) { // 事件主循环，处理文件事件、时间事件...
    	// AE_ALL_EVENTS: 文件事件、时间事件
    	// AE_CALL_BEFORE_SLEEP: 一次事件循环中，调用aeApiPoll之前执行的处理函数 (flushAppendOnlyFile)
        aeProcessEvents(eventLoop, AE_ALL_EVENTS | AE_CALL_BEFORE_SLEEP | AE_CALL_AFTER_SLEEP);
    }
}
```

单次事件循环aeProcessEvents函数的实现如下：

```
// 单次事件循环 aeProcessEvents
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
	// ...
	eventLoop->beforesleep(eventLoop);	    // beforesleep为函数指针，其指向的函数中会调用flushAppendOnlyFile方法！
	numevents = aeApiPoll(eventLoop, tvp);	// 通过I/O多路复用接口(select/poll/epoll),获取所有就绪的文件事件。
    // ... 处理文件事件 + 时间事件
	return processed;	// 返回处理的事件总数
}
```

`flushAppendOnlyFile`函数的行为根据服务器的配置选项`appendfsync`决定，该选项有三种取值，每种值对应的行为如下：

| appendfsync选项的取值 | flushAppendOnlyFile函数的行为                                | 安全性                                        |
| ---------------------------- | ------------------------------------------------------------ | --------------------------------------------- |
| always                | 总是将aof_buf缓冲区内容写入并同步到AOF文件                   | 最高，只丢失一个事件循环中的数据              |
| everysec              | 如距离上次同步AOF文件时间超过1秒，才对AOF文件进行同步操作，注意该同步操作通过一个线程专门负责执行 | 会丢失约1秒种的数据                           |
| no                    | 对AOF文件同步操作由操作系统自己决定                          | 最低，会丢失距离上次同步AOF文件之后的所有数据 |

可以看出，`everysec`选项兼顾了性能和安全性，这也是官方推荐的默认选项。

注：`fsync`，`fdatasync`同步函数，可以强制操作系统立即将内存缓冲区中数据写入磁盘。

#### 1.3 AOF重写

随着服务器持续运行，执行的写命令会越来越多，导致AOF文件越来越大，影响性能。因此我们需要对AOF文件大小加以控制，在不改变数据库状态的前提下，压缩AOF文件体积 —— 这就是Redis提供的AOF重写功能。

**举例:**

对一个列表键做如下写操作，为了保存这个列表键，AOF文件需记录3条命令，如下所示：

```react
redis> rpush list1 a b		# [a, b]
redis> rpop list1			# [b]
redis> rpush list1 c		# [b, c]
```

如果想用更少的命令记录这个列表键，最简单的方法是直接读取这个列表键的值，用`rpush list1 b c`替代上面的3条命令。

通过这个例子可以看出AOF重写的实现要点：**AOF重写通过读取服务器数据库状态来实现，而不是去分析现有的AOF文件！** 源码实现参考`rewriteAppendOnlyFileBackground`函数。

#### 1.4 AOF后台重写

AOF重写功能涉及大量写操作，Redis不希望AOF重写造成服务器无法处理请求，所以**将AOF重写放到子进程里执行**(这点和RDB持久化的BGSAVE思路类似)，这使得父进程不被阻塞，可以继续处理请求。

这种处理方式会引入了一个问题：子进程执行AOF重写时，服务器会继续处理请求，可能会执行新的写命令，导致数据库状态发生变化，与AOF文件中的数据库状态不一致！

为了解决这种数据不一致的问题，**Redis设置了一个AOF重写缓冲区，在子进程进行AOF重写期间，服务器将客户端的写命令请求同时追加到AOF缓冲区和AOF重写缓冲区**。

子进程完成AOF重写工作后，通知父进程将AOF重写缓冲区中的内容追加到新的AOF文件中，再原子性地覆盖旧的AOF文件，完成整个AOF后台重写。

源码实现参考`rewriteAppendOnlyFileBackground`函数和`backgroundRewriteDoneHandler`函数

#### 1.5 如何通过AOF文件还原数据库

创建一个无网络连接的伪客户端(fd值为-1)，从AOF文件中读出每条指令并执行，一直到AOF文件中所有的写命令执行完毕为止。源码实现参考`loadAppendOnlyFile`函数

### 2. AOF配置选项

常用配置选项如下：

```
appendonly yes                      # 值为yes，表示开启AOF持久化功能。
appendfilename "appendonly.aof"     # 指定aof文件名称
appendfsync everysec                # 指定AOF文件的写入方式， everysec表示每秒同步一次
```

### 3. AOF和RDB比较

RDB：文件相对较小，恢复较快，适合数据备份、灾难恢复。

AOF：文件相对较大，备份频率高(要设置fsync 策略), 适合故障恢复。

需针对不同业务场景选择合适持久化方式：

* 只用来做缓存 —— 可以关闭持久化功能。

* 对丢失数据不敏感 —— 仅使用RDB；对丢失数据敏感 —— 综合使用RDB + AOF

### 参考资料

【1】《Redis设计与实现》第11章 AOF持久化

【2】[Redis 持久化 RDB/AOF 详解与实践](https://gitchat.csdn.net/activity/5d5117876f8c3424da08b7af?utm_source=so)





















