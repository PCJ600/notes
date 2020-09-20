## redis基本命令和基本类型

### 一、字符串类型

#### 查询

keys pattern 获取符合规则的键名表, 支持如下正则表达式:

| 符号 | 含义                                      |
| ---- | ----------------------------------------- |
| ?    | 匹配一个字符                              |
| *    | 匹配任意个(包括0)字符                     |
| []   | 匹配括号任一字符，用'-'符号可表示一个范围 |
| \    | 转义符号，如匹配?，使用\?                 |

```shell
keys "*" 						#遍历redis所有键
exists key 						#判断一个键是否存在
type key 						#获取键值的类型
```

#### 删除

```shell
del key 										#删除一个键，返回删除键的个数
redis-cli KEYS "str*" | xargs redis-cli DEL		#批量删除键
redis-cli DEL 'redis-cli KEYS "str*"'
```

#### 赋值与取值

```shell
set key value
get key 
```

增加指定的整数

```shell
incr key					#加1
incrby key increment		#增加指定increment
decr key 					#减少1
decrby key increment 		#减少指定整数
```

向尾部追加值

```shell
append key value			#若键不存在则设置为valu,返回追加后字符串总长度
```

获取键值的字符串长度

```shell
strlen key
```

同时设置\获取多个键值

```shell
mget key1 key2 ...
mset key1 value1 key2 value2 ...
```

位操作

```shell
getbit key offset
setbit key offset value
bitcount key [start] [end] #获取字符串类型键中值是1的二进制位个数,start,end限制统计字节范围
... 
bitpos foo 1 #获取键foo中第一个二进制位为1的偏移量
```

### 二、散列类型

散列类型是一种字典结构，存储了字段和字段值的映射，但字段值只能是字符串

适用存储对象，如存储汽车，可用color,name,price字段存储汽车颜色、名称、价格

#### 赋值与取值

```shell
hset key field vale			#赋值
hget key field				#取值
hmset key field value [field value ...]	#同时设置多字段
hmget key field [field ... ]			#同时获取多字段
hgetall key								#不知道键中多少字段，可使用hgetall命令
```

HSET命令不区分插入和更新操作，意味着修改数据时不用事先判断是否存在来决定是插入还是更新

#### 判断字段是否存在

```shell
hexists key field 			#判断字段是否存在
hsetnx key field value		#字段不存在时赋值
```

#### 增加数字

```shell
hincrby person score 60 	#若不存在，默认增加之前的值为0
```

#### 删除字段

```shell
hdel key field [field ...] 		#删除一个或多个字段
```

#### 只获取字段名或字段值

```shell
hkeys key						#只获取字段名
kvals key						#只获取字段值
hlen key						#获取字段数
```

#### 三、列表类型

存储一个有序的字符串列表，常用操作时两端添加元素，或获取列表的某个片段

使用双向链表实现，索引访问较慢，最多容纳2的32次方-1个键

#### 向列表两端增加元素

```shell
lpush key value [value ...]			#列表左端增加元素
rpush key value [value ...]			#列表右端增加元素
lpush numbers 2 3 4					#向空列表左边依次添加4,3,2
```

#### 从列表两端弹出元素

```shell
lpop eky							#从列表左边弹出元素
rpop key							#从列表右边弹出元素
llen key							#返回列表长度，键不存在时返回0
```

#### 获得列表片段

```shell
lrange key start stop				#获得列表片段
lrange numbers 0 2					#获取第0,1,2个元素
```





























































































