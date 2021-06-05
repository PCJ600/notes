# Redis学习笔记（八)：RDB持久化

[TOC]

## 简介

Redis是基于内存的数据库，服务器进程退出后，内存中的数据会丢失。为了解决这个问题，Redis提供了RDB持久化机制，将内存中的数据保存到硬盘，从而避免数据意外丢失。

## RDB持久化

RDB持久化将某个时间点上的数据库状态保存到一个RDB文件，这个RDB文件是一个经过压缩的二进制文件，Redis服务器可以通过读取RDB文件还原数据库状态。

RDB文件的生成路径和文件名通过配置文件指定，如下所示：

```
# dbfilename指定RDB文件名
dbfilename dump.rdb

# dir表示工作目录, RDB文件会写入这个目录
dir /var/redis/6379
```

### 1. RDB文件如何创建

RDB持久化既可以手动触发，也可以通过配置文件的选项定期触发。

#### 1.1 save, bgsave命令

执行`save`, `bgsave`命令，可以手动生成RDB文件。两者的区别如下：

* `save`命令会阻塞Redis服务器进程，直到RDB文件创建完毕；阻塞期间服务器无法处理任何请求。
* `bgsave`命令会fork一个子进程，由子进程负责创建RDB文件，父进程能够继续处理命令请求。

**注：**`bgsave`命令执行期间，服务器会拒绝执行客户端的`save`, `bgsave`命令，原因是防止竞争条件。

创建RDB文件的实际动作通过`rdb.c/rdbSave`函数完成,3.0版本的实现细节如下:

* 先创建临时文件`"temp-%d.rdb"`, %d为当前进程id。
* 遍历所有数据库,将数据库状态写入到该临时文件, 写入完成后调用`fflush`, `fsync`确保数据被写入硬盘。

* 调用`rename`方法,原子性地覆盖旧的RDB文件，覆盖成功后，将`dirty`计数器清零并记录当前时间(即最后一次完成RDB持久化的时间)。

#### 1.2 定期触发

##### 1.2.1 保存条件怎么设置

在配置文件中更改`save <seconds> <changes>`选项，可以设置多个**保存条件**。只要其中某个保存条件被满足，服务器就执行`bgsave`命令。

官方给出的save选项的典型设置和意义如下：

```
# Save the DB on disk:
#   save <seconds> <changes>
#
#   Will save the DB if both the given number of seconds and the given
#   number of write operations against the DB occurred.
#
#   In the example below the behavior will be to save:
#   after 900 sec (15 min) if at least 1 key changed
#   after 300 sec (5 min) if at least 10 keys changed
#   after 60 sec if at least 10000 keys changed
#
#   Note: you can disable saving completely by commenting out all "save" lines.
#
#   It is also possible to remove all the previously configured save
#   points by adding a save directive with a single empty string argument
#   like in the following example:
#
#   save ""

save 900 1				# 如果900秒内至少1个键被修改，就执行bgsave命令
save 300 10				# 300秒内至少10个键被修改，就执行bgsave命令
save 60 10000			# 60秒内至少10000个键被修改，就执行bgsave命令
```

可以看出，如果想关闭RDB功能，只需注释掉所有save选项后，再添加`save ""`即可

##### 1.2.2 实现细节

Redis服务器状态通过redisServer结构表示，其中`saveparams`属性用于记录所有的保存条件，如下：

```C
struct redisServer {
	struct saveparam *saveparams; // 一维数组, = {{900, 1}, {300, 10}, {60, 10000}}
    // ...
};
struct saveparam {
	time_t seconds;	// 秒数
	int changes;	// 修改数
};
```

光有保存条件还不行，我们还需要知道**上一次成功执行RDB持久化的时间**，以及**距离上一次成功执行RDB持久化，服务器对数据库总共做了多少次修改**，才能和每个保存条件做比较，最终判断某个保存条件是否被满足。

因此，redisServer结构中新增`dirty`计数器和`lastsave`属性，如下：

```c#
struct redisServer {
	long long dirty;	// 计数器
	time_t lastsave;	// 上一次成功执行持久化的时间，格式是UNIX时间戳
    // ...
};
```

* `dirty`计数器记录距离上一次成功执行RDB持久化，服务器对数据库一共做了多少次修改。
* `lastsave`属性记录上一次成功执行RDB持久化的时间，格式为UNIX时间戳。

Redis服务器会周期性地检查保存条件是否满足, 源码实现可参考`redis.c/ServerCron`

```C
// serverCron是Redis服务器的时间事件，周期性执行，默认周期为0.1s, 可以通过改配置选项hz修改这个默认周期
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
	// ...
    for (j = 0; j < server.saveparamslen; j++) {
        struct saveparam *sp = server.saveparams+j;
        if (server.dirty >= sp->changes &&
            server.unixtime-server.lastsave > sp->seconds &&
            (server.unixtime-server.lastbgsave_try >
            REDIS_BGSAVE_RETRY_DELAY ||
            server.lastbgsave_status == REDIS_OK)) {
            rdbSaveBackground(server.rdb_filename); // 某个保存满足，就执行BGSAVE命令
            break;
         }
    }	
    // ...
}
```

### 2. RDB文件如何加载

Redis没有提供专门加载RDB文件的命令，只要Redis在启动时检测到了RDB文件的存在，就会自动加载RDB文件,加载RDB文件的动作由`rdb.c/rdbLoad`函数完成。

**注**: 如果Redis开启了AOF持久化功能, 服务器会优先使用AOF文件而不是RDB文件还原数据库。

### 3. RDB文件结构和解析方法

能借助工具离线分析RDB文件即可, 掌握RDB文件结构是非必要的，只需了解：

* RDB文件是一个经过压缩的二进制文件，对于不同类型的键，会使用不同的方式去存储。
* RDB文件不保存已过期的键，但是会保存键的过期时间。

Redis官方提供了`redis-check-rdb`工具用于检测RDB文件。

举例，在某个空的redis数据库中新增5个key，如下所示：

```
# 每种类型键增加1个							   # 查询键的命令
set key1 123								# get key1
rpush list1 1 2 3							# lrange 0 -1
hset hashtable1 k1 v1 k2 v2 k3 v3 			# hgetall hashtable1
sadd set1 1 2 3								# smembers set1
zadd zset1 1.0 m1 2.0 m2 3.0 m3				# zrange zset1 0 -1 withscores
```

可以通过`ob -cx dump.rdb`命令解析这个RDB文件, 但输出结果不够直观；更好的方式是借助开源社区已有的RDB分析工具, 可以参考：https://github.com/sripathikrishnan/redis-rdb-tools 

这里将RDB文件转换为json格式查看，命令如下：

```txt
# rdb -c json /var/redis/6379/dump.rdb
[{
"set1":["1","2","3"],
"zset1":{"m1":"1","m2":"2","m3":"3"},
"hashtable1":{"k1":"v1","k2":"v2","k3":"v3"},
"list1":["1","2","3"],
"str1":"123"}]
```

## 参考资料

【1】《Redis设计与实现》  第10章 RDB持久化

【2】[Redis RDB文件离线分析](https://cloud.tencent.com/developer/article/1394329)































