## Redis学习笔记：数据库

[TOC]

### 前言

详细介绍Redis数据库的实现，解答以下几个问题：

* Redis服务器是怎么保存数据库的？客户端又是怎么切换数据库的？
* 数据库的增、删、改、查的实现
* 键的过期时间是怎么保存的，又是如何删除的？怎么判断一个键是否过期？
* 过期键的删除策略有哪些？每种策略的优缺点分析？Redis采用的是哪种策略，具体又是怎么实现的？

### 1. 服务器中的数据库

Redis将所有数据库都保存在服务器状态 `server.h/redisServer`结构的db数组中，db数组中的每一项表示一个数据库, dbnum表示数据库个数。 

```C
struct redisServer {
	redisDb *db;	// db数组保存所有数据库
    int dbnum;		// 表示数据库个数
    ...
};
```

服务器初始化时，默认创建16个数据库。可以通过修改配置文件的databases选项更改数据库的数量。

```txt
# modify /etc/redis/redis.conf
databases 16
```

客户端可通过`config get databases`命令查看数据库的数量。

```
127.0.0.1:6379> config get databases
1) "databases"
2) "16"
```

### 2. 客户端切换数据库 

每个客户端都有自己的目标数据库，默认情况下客户端的目标数据库为0号数据库。客户端可以通过执行SELECT [dbid] 命令切换目标数据库。

Redis服务器中，使用redisClient结构(6.0版本此结构体名称改为client)表示客户端属性，结构中的db属性表示客户端的目标数据库，如下：;

```C
struct redisClient {
	redisDb *db;
	// ...
} redisClient;
```

通过修改db指针，指向redisServer.db数组中的某个元素，来实现目标数据库的切换，源码参考`db.c/selectDb`

```C
int selectDb(redisClient *c, int id) {
    if (id < 0 || id >= server.dbnum)
        return REDIS_ERR;
    c->db = &server.db[id];	// 通过修改db指针，指向redisServer.db数组中的某个元素，实现目标数据库的切换
    return REDIS_OK;
}
```

### 3. 数据库键空间、增删改查操作

Redis是一个Key-Value型数据库，服务器中的每个数据库都由一个redisDb结构表示，其中redisDb结构的dict字典保存了数据库中的所有键值对，将这个字典称为**键空间**(key space)

```C
typedef struct redisDb {
	dict *dict;		// 数据库键空间，保存数据库中所有键值对
	dict *expires;	// 过期字典
};
```

以下介绍Redis数据库增、删、改、查操作的实现：

####  查询键的实现

在键空间中查询给定键是否存在， 通过lookupKey函数实现：

```C
robj *lookupKey(redisDb *db, robj *key) {
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {        
        robj *val = dictGetVal(de);  // 如果键存在，就取出值
        // 更新时间信息（只在不存在子进程时执行，充分利用写时复制机制）
        if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
            val->lru = LRU_CLOCK();
        return val;
    } else {
        return NULL;
    }
}
```

#### 增加键的实现

 将新键值对添加到键空间，通过dbAdd函数实现：

```C
void dbAdd(redisDb *db, robj *key, robj *val) {
    sds copy = sdsdup(key->ptr);
    int retval = dictAdd(db->dict, copy, val);	// 增加键值对
	// ...
 }
```

#### 删除键的实现

删除给定的键，注意需同时删除这个键的过期时间， 通过dbDelete函数实现：

```C
int dbDelete(redisDb *db, robj *key) { 
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr); // 先删除键的过期时间
    if (dictDelete(db->dict,key->ptr) == DICT_OK) {
        return 1; // 删除成功返回1
    } else {
        return 0; // 删除失败返回0
    }
}
```

#### 更新键的实现

通过dbOverwrite函数实现：

```C
void setKey(redisDb *db, robj *key, robj *val) {
    if (lookupKeyWrite(db,key) == NULL) {	// 如果key不存在，新增键值对
        dbAdd(db,key,val);
    } else {	// 如果key已经存在，更新键值对
        dbOverwrite(db,key,val);
    }
	// ...
}
void dbOverwrite(redisDb *db, robj *key, robj *val) {
    dictEntry *de = dictFind(db->dict,key->ptr);
	redisAssertWithInfo(NULL,key,de != NULL);
	dictReplace(db->dict, key->ptr, val);	// 更新键空间
}
```
#### 其他对键空间的操作

| 命令         | 功能                       |
| ------------ | -------------------------- |
| FLUSHALL     | 清空所有数据库             |
| FLUSHDB      | 清空目标数据库             |
| RANDOMKEY    | 随机返回一个键             |
| DBSIZE       | 返回目标数据库的键值对数量 |
| EXISTS [key] | 判断键是否存在             |

### 4. 键的生存时间

#### 如何设置键的生存时间？

**expire, pexpire**命令以秒/毫秒为精度，对数据库的某个键设置生存时间(Time to Live：TTL)。经过指定的时间后，服务器会自动删除生存时间为0的键。

```
127.0.0.1:6379> expire key 5	# 设置生存时间为5秒，5秒后服务器自动删除这个键
(integer) 1
127.0.0.1:6379> TTL key			# TTL命令返回这个键的生存时间，单位:秒
(integer) 3
127.0.0.1:6379> get key			# 5秒后，键过期，被服务器自动删除
(nil)
```

Redis中，可以使用**expire, pexpire, expireat, pexpireat**设置键的生存时间，用法如下：

| 命令               | 功能               |
| ------------------ | ----------------------- |
| expire [key] [ttl] | 设置键的生存时间为ttl秒 |
| pexpire [key] [ttl] | 设置键的生存时间为ttl毫秒 |
| expireat [key] [timestamp] | 设置过期时间为秒级时间戳 |
| pexpireat [key] [timestamp] | 设置过期时间为毫秒级U时间戳 |

Redis源码实现中，expire, pexpire, expireat命令最终都会转化为pexpireat命令，相关源码如下：

```C
void expireCommand(redisClient *c) {				// expire 命令
    expireGenericCommand(c,mstime(),UNIT_SECONDS);
}
void expireatCommand(redisClient *c) {				// expireat 命令
    expireGenericCommand(c,0,UNIT_SECONDS);
}
void pexpireCommand(redisClient *c) {				// pexpire 命令
    expireGenericCommand(c,mstime(),UNIT_MILLISECONDS);
}
void pexpireatCommand(redisClient *c) {				// pexpireat 命令
    expireGenericCommand(c,0,UNIT_MILLISECONDS);
}
void expireGenericCommand(redisClient *c, long long basetime, int unit);
```

#### TIME命令介绍

time命令用于返回当前服务器时间，返回值包含两个字符串，意义如下：

```txt
127.0.0.1:6379> TIME
1) "1615638731"			# 表示当前时间，格式为UNIX时间戳
2) "628667"				# 表示当前这一秒中，已经流逝的微秒数，1秒=1000000微妙，这个值总小于1000000
```

#### redis如何保存过期时间？

redisDb结构的expires字典保存了数据库中所有键的过期时间，这个expire字典我们称之为**过期字典**。

* 过期字典的键是一个指针，指向键空间的某个键对象
* 过期字典的值是一个long long类型整数，毫秒精度的UNIX时间戳。

pexpireat命令在过期字典中查找给定键，并设置值为过期时间(格式为UNIX时间戳)；具体实现可参考expireGenericCommand函数。

#### redis如何移除过期时间？

persist命令可以移除一个键的过期时间， 效果相当于反向执行pexpireat命令：在过期字典中查找给定键，删除这个键对应的值；具体实现可参考persistCommand函数。

```
127.0.0.1:6379> EXPIRE key 100		# 设置key的生存时间100秒
(integer) 1
127.0.0.1:6379> TTL key				# 返回生存时间
(integer) 97
127.0.0.1:6379> PERSIST key			# 移除key的生存时间
(integer) 1
127.0.0.1:6379> TTL key				# 生存时间为-1, 表示为永久键
(integer) -1
```

####  怎么判断一个key是否过期

TTL命令以秒为单位返回键的生存时间，PTTL命令以毫秒为单位返回键的生存时间。

TTL、PTTL命令**通过计算键的过期时间和当前时间的差**来实现。

判断一个key是否过期的步骤如下，具体实现可以参考expireIfNeeded函数：

* 首先，检查给定键是否在过期字典中，如果存在，取得键的过期时间
* 其次，检查当前UNIX时间戳是否大于键的过期时间，如果大于表示已过期。

### 5. 过期键删除策略

有三种常见的过期键删除策略，分别如下：

* 定时删除：设置键过期时间的同时创建一个定时器，定时器超时后立即删除该键。
* 惰性删除：放任键过期不管，直到需要读写改键时才检查是否过期，如过期就删除该键。
* 定期删除：每隔一段时间，对数据库做一次检查，删除过期的键。

三种策略的优缺点分析：

* 定时删除：对内存友好，对CPU不友好，影响服务器的响应时间和吞吐量。

* 惰性删除：对CPU友好，但浪费内存，可能导致内存泄漏。
* 定期删除：是对前两种策略的折中，其难点在于确定删除操作执行时长和频率。

#### Redis使用的过期键删除策略

Redis综合使用了惰性删除和定期删除这两种策略，策略具体实现如下：

#### 惰性删除

Redis的惰性删除策略在expireIfNeed函数实现，所有读写数据库的命令在执行之前都会调用expireIfNeeded函数对输入键进行检查：

* 如果输入键已经过期，expireIfNeeded函数将输入键删除，命令当做键不存在的情况去执行。
* 如果输入键未过期，expireIfNeed函数什么也不做，继续执行实际的命令流程。

#### 定期删除

过期键的定期删除策略由redis.c/activeExpireCycle函数实现。每当Redis的时间事件serverCron函数周期性执行时，activeExpireCycle函数就随之被调用。 这个周期默认为0.1秒，可以通过配置文件的hz选项修改这个值。

```
# modify /etc/redis/redis.conf
# The range is between 1 and 500, however a value over 100 is usually not
# a good idea. Most users should use the default of 10 and raise this up to
# 100 only in environments where very low latency is required.
hz 10
```

activeExpireCycle函数的实现原理：在规定时间内，分多次遍历服务器中的数据库，从数据库的expires字典中随机检查一部分键的过期时间，并删除其中的过期键。

### 参考资料

《Redis设计与实现》第9章 数据库