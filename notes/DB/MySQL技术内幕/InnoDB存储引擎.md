## 第一章 MySQL体系结构和存储引擎

MySQL体系结构 (**插件式存储引擎**!)

![MySQL体系结构](C:\PC\learning\notes\DB\MySQL技术内幕\MySQL体系结构.PNG)

可以查询MySQL数据库支持的存储引擎

```
show engines;
```

* MyISAM
* InnoDB

### 连接方式

1、TCP/IP（默认方式）

```mysql
mysql -h [ip] -u [user] -p;
```

2、共享内存、命名管道 (本地 windows)

3、UNIX域套接字（本地 Linux）

```mysql
show variables like 'socket';
+---------------+-----------------------------+
| Variable_name | Value                       |
+---------------+-----------------------------+
| socket        | /var/run/mysqld/mysqld.sock |
+---------------+-----------------------------+
# mysql -u [user] -S /tmp/mysql.sock
```



## 第2章 InnoDB存储引擎

### 2.2 InnoDB存储引擎的版本

| 版本          | 功能                                             |
| ------------- | ------------------------------------------------ |
| 老版本 InnoDB | 支持ACID，行锁设计，MVCC                         |
| InnoDB 1.0.x  | 继承上述所有功能，增加了compress, dynamic页格式  |
| InnoDB 1.1.x  | 继承上述所有功能，增加了Linux AIO, 多回滚段      |
| InnoDB 1.2.x  | 继承上述所有功能，增加全文索引支持，在线索引添加 |

OLTP　——　Online transaction processing

 ### 2.3 InnoDB体系架构

![InnoDB存储引擎体系结构](C:\PC\learning\notes\DB\MySQL技术内幕\InnoDB存储引擎体系结构.PNG)

InnoDB存储引擎有多个内存快，这些内存块组成一个大内存池，负责如下工作：

* 维护所有进程/线程序访问的多个内部数据结构。
* 缓存磁盘上的数据，方便快速的读取。同时在对磁盘文件的数据修改之前在这里缓存。
* 重做日志(redo log)缓冲。

查看InnoDB Version

```
mysql> show variables like 'innodb_version';
+----------------+--------+
| Variable_name  | Value  |
+----------------+--------+
| innodb_version | 5.7.34 |
+----------------+--------+
```

#### 后台线程

InnoDB存储引擎是多线程的模型，其后台有多个不同的后台线程，负责不同的任务。

1、Master Thread

核心线程，负责将缓冲池数据异步刷新到磁盘，保证数据一致性，包括脏页刷新,合并插入缓冲,UNDO页的回收。

2、IO Thread

InnoDB存储引擎中大量使用AIO(Async IO)处理写IO请求，提高数据库的性能。

```mysql
show engine innodb status;
```

查看IO Thread

```
mysql> show variables like 'innodb_%io_threads';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_read_io_threads  | 4     |
| innodb_write_io_threads | 4     |
+-------------------------+-------+
```

通过'show engine innodb status'查看InnoDB中的IO Threads

```
mysql>show engine innodb status;
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
```

3、Purge Thread

事务被提交后，其所使用的undolog可能不再需要，因此需要PurgeThread回收已使用分配的undo页。

1.1版本之前，purge操作仅在InnoDB存储引擎中的Master Thread完成，没有独立线程。

1.1版本开始，purge操作可以独立在单独的线程中执行。启用方式如下：

```
[mysqld]
innodb_purge_threads=1
```

1.2版本开始，InnoDB支持多个Purge Thread

```
mysql> show variables like 'innodb_purge_threads';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| innodb_purge_threads | 4     |
+----------------------+-------+
```

4、Page Cleaner Thread

将脏页刷新操作放到单独线程完成，进一步提高InnoDB性能

#### 内存

http://mysql.taobao.org/monthly/2017/05/01/

https://blog.csdn.net/qq_18312025/article/details/78587334

#### 1. 缓冲池

数据库系统中，由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池技术提高数据库性能。缓冲池简单说是一块内存区域，通过内存速度弥补磁盘速度较慢对数据库性能的影响。

* 读：先将磁盘读到页放到缓冲池(FIX)，下次再读相同页时，先判断该页是否在缓冲池中，若命中直接读取该页，否则读取磁盘上的页。
* 写：先修改在缓冲区中的页，然后再以一定频率刷到磁盘上。注意页从缓冲池刷新回磁盘操作不是在每次页发生更新时触发，**而是通过一种称为Checkpoint机制刷新回磁盘。**

缓冲池通过参数`innodb_buffer_pool_size`设置

1G = 1024M = 1024 * 1024KB = 1024 * 1024 * 1024B

```
mysql> show variables like 'innodb_buffer_pool_size'; (128M)
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| innodb_buffer_pool_size | 134217728 |
+-------------------------+-----------+
1 row in set (0.00 sec)
```

#### InnoDB内存数据对象

具体来说，缓存池中数据页类型：索引页，数据页，undo页、插入缓冲、自适应哈希索引、Inno存储锁信息

![InnoDB内存数据对象](C:\PC\learning\notes\DB\MySQL技术内幕\InnoDB内存数据对象.PNG)

从Inno DB 1.0.x版本开始，允许多个缓冲区实例。每个页根据哈希值平均分配到不同缓冲池实例中。好处是减少数据库内部的资源竞争，增加数据库的并发处理。

查看缓冲池实例个数：

```
mysql> show variables like 'innodb_buffer_pool_instances';
+------------------------------+-------+
| Variable_name                | Value |
+------------------------------+-------+
| innodb_buffer_pool_instances | 1     |
+------------------------------+-------+
```

三条重要的链表：**Free List, LRU List, FLU List**

#### InnoDB的LRU算法

InnoDB存储引擎中的缓冲池，基于LRU(最近最少使用)管理

InnoDB存储引擎中，缓冲池中页大小默认为16KB

InnoDB中的LPU算法，加入midpoint位置，**新读取到的页并不是直接放入首部，而是放入到列表的midpoint位置， 默认配置在5/8位置处**，midpoint位置由参数innodb_old_blocks_pct控制。

```
mysql> show variables like 'innodb_old_blocks_pct';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_old_blocks_pct | 37    |
+-----------------------+-------+
```

InnoDB中，把midpoint之后列表称为old列表，之前的列表称为new列表；new列表中的页是最活跃的热点数据

#### LRU List

![img](https://upload-images.jianshu.io/upload_images/17793180-57a55489387d76f9.png?imageMogr2/auto-orient/strip|imageView2/2/w/735/format/webp)

#### <font color='red'>为什么不采用朴素LPU算法</font>

InnoDB中，LRU List分为两部分，默认前5/8为young list，存储热key, 后3/8为old list.

新读入的page默认加在old list头部，只有满足**一定条件**后才移到young list, 主要为了预读数据页和**全表扫描污染**buffer poll

```
mysql> show variables like 'innodb_old_blocks_time';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_old_blocks_time | 1000  |
+------------------------+-------+
```

#### innodb_old_blocks_time

这个参数表示页被读取到mid位置后，需等待多久才会被加入到LPU列表的热端。默认1s

```sql
set global innodb_old_blocks_time=1000;
```

放在冷热数据交界处，默认1s，过了这1s还能存活，就从old调到young.

#### 冷热数据区的监控

```
mysql>show engine innodb status;
page made young XXX, not young XXX
0.00 youngs/s, 0.00 non-youngs/s
```

* youngs/s： 坚持到了1s，数据由old到new。
* non-youngs/s: 没有坚持到1s, 于是被刷出去。

##### non-youngs/s值过大原因

* 可能存在严重的全表扫描
* 冷数据区设置过小
* time设置的过大（冷数据没坚持到指定时间，被刷掉）

yonngs/s的值过大原因

* 冷数据区设置过大
* time设置过小

### Free List

InnoDB从1.0.x版本支持压缩页功能，将原本16K页压缩为1KB, 2KB, 4KB, 8KB

对于非16K的页，通过unzip_LRU列表管理。

```
mysql>show engine innodb status;
...
LRU len: 1539, unzip_LRU: 156
```

看出LRU共有1539个页，其中unzip_LPU列表有156个页。这里需注意，LPU中的页包含了unzip_LPU列表中的页



























