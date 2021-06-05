# Redis学习笔记：独立功能的实现





缓存穿透

缓存雪崩

缓存击穿

Redis内存淘汰

Redis配置文件

blomm过滤器





### 事务

 事务三阶段

* 事务开始 MULTI
* 命令入队 将命令放入事务队列，向客户端返回QUEUED
* 事务执行

#### 事务状态

每个客户端都有自己的事务状态，这个事务状态保存在客户端的mstate属性

```C
typedef struct redisClient {
	multiState *mstate;
	// ...
};
typedef struct multiState {
    multiCmd *commands;		// 事务队列，FIFO顺序
    int count;				// 已入队命令计数
};
// 事务命令
typedef struct multiCmd {
    robj **argv;				// 参数
    int argc;					// 参数数量
    struct redisCommand *cmd;	// 命令指针
} multiCmd;
```

EXEC命令实现原理：参考`multi.c/execCommand`

#### WATCH命令 —— 乐观锁

在EXEC命令执行之前， 监视任意数量的数据库的键，检查被监视的键是否被修改过，如果是，服务器拒绝执行事务。

**乐观锁：**https://zhuanlan.zhihu.com/p/156853879

每个数据库保存着一个watched_keys字典，这个字典的键是被WATCH监视的数据库键，值为一个链表，链表中记录所有监视该键的客户端

```C
typedef struct redisDb {
	dict *watched_keys;
} redisDb;
```

#### 监视机制的触发

对watched_keys作检查，查看是否有客户端正在监视刚被命令修改过的数据库键，如果有，执行`multi.c/touchWatchedKey`将监视被修改键的客户端的REDIS_DIRTY_CAS标识打开，标识事务安全性遭到破坏

#### 事务的ACID性质

* 原子性(Atomicity) 
* 一致性(Consistency) 
* 隔离性(Isolation)
* 耐久性(Durability)

##### 原子性

将事务中的多个操作当成一个整体来执行，服务器要么执行所有操作，要么一个操作不执行。

Redis的事务是原子性的，但不支持事务回滚机制。(如果某个命令执行期间出错，后续命令会继续执行)

##### 一致性

事务开始和结束之间的中间状态不会被其他事务看到。

##### 隔离性

数据库中多个事务并发地执行，各事务之间不会互相影响，且在并发状态下执行的事务和串行执行的事务产生的结果完全相同。因为Redis使用单线程模型的方式执行事务，并且服务器保证，执行事务期间不会对事务进行中断，因此，Redis事务总以串行方式运行，具有隔离性。

##### 耐久性

当一个事务执行完毕时，执行这个事务所得的结果已经被保存到永久性存储介质里，即使事务执行完毕停机后，执行事务的结果也不会停机。

当运行在AOF持久化模式下，且appendfsync选项的值为always时，是具备耐久性的(不完全是这样^_^)

* no-appendfsync-on-rewrite选项打开时，在执行BGSAVE或BGWRITEAOF期间，服务器暂停对AOF文件同步，尽可能减少I/O阻塞，这样事务的耐久性失去保证。



### 排序

SORT [列表键/集合键] [ASC|DESC] [limit offset count] [store destination]

* SORT命令将被排序键包含元素存放到数组中，然后对数组排序。

* SORT命令默认假定被排序键都是数字值，如使用ALPHA选项表示对字符串排序

### 二进制位数组

#### 用法

getbit [key] offset

setbit [key] offset value

bitcount [key]

bitop [AND/OR/XOR/NOT] [key] [...]

#### GETBIT， SETBIT的实现

SDS，逆序存储(SET扩展时，无需移动数位，效率高)

byte = [offset % 8], bit = (offset mod 8) + 1

#### BITCOUNT的实现

综合使用查表法+variable-precision SWAR法

* 首先考虑4字节对齐，必须先计算没有对齐的位数中1的个数

* 待处理二进制位大于128，用variable-precision-SWAR
* 小于128，通过查表法，每次一字节(键长为8位的表，表大小为sizeof(char) * 256 = 256字节)

#### BITOP的实现

没啥技巧，使用C语言内置的位操作实现，逐字节操作即可。

### 场景

如统计连续三天登录的用户数量，可以给每天设置一个位数组，通过与操作实现

### 慢查询日志

常用选项：

* slowlog-log-slower-than选项指定执行时间超过多少微妙的命令请求记录到慢查询日志。
* slowlog-max-len选项指定服务器最多保存多少条慢查询日志。
* slowlog reset删除所有慢查询日志

慢查询日志命令解析

```txt
127.0.0.1:6379> CONFIG SET slowlog-log-slower-than 300
OK
127.0.0.1:6379> CONFIG SET slowlog-max-len 100
OK
127.0.0.1:6379> SLOWLOG get
1) 1) (integer) 0				# 日志标识符
   2) (integer) 1616910153		# 命令执行时的UNIX时间戳
   3) (integer) 258				# 耗时，微妙计算
   4) 1) "keys"					# 命令参数
      2) "*"
   5) "127.0.0.1:54448"
   6) ""
```

#### 实现 (slowlog.c/slowlogCommand)

服务器状态中包含几个和慢查询日志功能有关的属性：

```C
struct redisServer {
	long long slowlog_entry_id;					// 下一条慢查询日志的ID，递增
	list *slowlog;								// 用链表保存所有慢查询日志。
	long long slowlog_log_slower_than;			// 执行时间超过多少就记录一条慢查询日志
	unsigned long slowlog_max_len;				// 最多保存的慢查询日志条数
};
```

slowlog链表的每一个节点保存一个slowlogEntry结构

```C
typedef struct slowlogEntry {
	long long id;				// 标识符 = 0
	time_t time;				// 命令执行时的时间 = 1616910153
	long long duration;			// 执行命令消耗的时间，微妙为单位 = 258
	robj **argv;				// 命令 = keys "*"
	int argc;					// 命令参数数量 = 2
};
```

##### 添加一条慢查询日志

每次执行命令时，Redis记录微秒格式的时间戳，差值即为执行命令耗费时长，参考`slowlogPushEntryIfNeeded`函数，伪代码如下：

```
before = unixtime_now_in_us(); 				// 记录执行命令时的时间
execute_command(argv, argc, client);		// 执行命令
after  = unixtime_now_in_us();				// 记录执行命令后的时间
slowlogPushEntryIfNeeded(argv, argc, before-after);
```

执行命令的关键函数**call();**

### 监视器（`redis.c/monitorCommand`）

执行monitor命令，客户端可以将自己变成一个监视器，实时接收打印服务器当前处理的命令请求

服务器状态中包含监视器链表结构：

```C
struct redisServer {
	// ...
	list *monitors;		// 每个节点包含一个redisClient结构，保存客户端信息s
};
```

























































