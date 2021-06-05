# Redis学习笔记：字典

[TOC]

### 前言

字典是一种用于保存键值对的数据结构。

**Redis数据库**使用字典做为底层实现，字典也是**哈希键**的底层实现之一。

C语言中没有内置字典这个数据结构，Redis自己实现了字典。

### Redis字典的实现

### 1. 数据结构设计

Redis的字典使用哈希表作为底层实现，一个哈希表又由多个哈希表节点组成。

#### 1.1 哈希表的数据结构定义

```C
typedef struct dictht {			// dict.h/dictht			
	dictEntry **table;			// 哈希表数组
	unsigned long size;			// 哈希表大小
	unsigned long sizemask;		// 用于计算索引，大小为size - 1
	unsigned long used;			// 哈希表已有节点数
} dictht;
```

* `table`是一个二维数组，数组的每个元素指向哈希表节点`dictEntry`。

* `size`表示哈希表大小，`used`表示哈希表已有节点数。

* `sizemask`用于计算一个键应该放到`table`数组的哪个索引上，`sizemask`大小始终等于`size - 1`

#### 1.2 哈希表节点的数据结构定义

```C
typedef struct dictEntry {		// dict.h/dictEntry
	void *key;					// 键
    union {						// 值，可以保存指针，或者有符号/无符号整数；6.0版本还可以保存double类型
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    struct dictEntry *next;		// 指向链表中的下一个节点，链地址法解决键冲突
} dictEntry;
```

* 每个`dictEntry`结构保存一个键值对，`key`保存键，`value`保存值。

* `next`指向链表中的下一个节点。Redis使用**链地址法**解决键冲突问题，将多个分配到哈希表数组的同一个索引上的节点组成单链表。新节点总是添加到链表头，复杂度为O(1)

#### 1.3 字典的数据结构定义

```C
typedef struct dict {
	dictType *type;			// type和privdata用于多态
	void *privdata;
	dictht ht[2];			// 哈希表
	int trehashidx;			// 表示当前rehash进度，当哈希表没有进行rehash时，值为-1
    int iterators; 			// 迭代器数量
}
```

* `ht`属性包含2个哈希表。通常情况只使用`ht[0]`, `ht[1]`只在rehash场景下使用。

* `trehashidx`属性表示rehash进度，如当前没有进行rehash, 该值为-1

* `type`和`private`属性用于创建多态字典。`dictType`数据结构如下：

```C
typedef struct dictType {
    unsigned int (*hashFunction)(const void *key);    // 计算hashCode
    void *(*keyDup)(void *privdata, const void *key); // 复制key
    void *(*valDup)(void *privdata, const void *obj); // 复制value
    int (*keyCompare)(void *privdata, const void *key1, const void *key2); // 对比key
    void (*keyDestructor)(void *privdata, void *key); // 销毁key
    void (*valDestructor)(void *privdata, void *obj); // 销毁value
} dictType;
```

将上述几个数据结构定义串联起来，下图表示一个Redis字典结构：

![捕获](C:\Users\pc\Desktop\捕获.PNG)

（图片来源：[http://redisbook.com/preview/dict/rehashing.html](http://redisbook.com/preview/dict/rehashing.html)）

### 2. 哈希算法

先根据key计算出哈希值，再根据哈希值计算索引值。代码如下：

```C
#define dictHashKey(ht, key) (ht)->type->hashFunction(key)
unsigned int hash = dictHashKey(ht, key);	// 先计算hash值，结果为32位
unsigned int index = hash & ht->sizemask;	// 再根据hashCode计算索引
```

Redis3.0采用MurmurHash2算法计算哈希值，可参考`dict.c/dictGenHashFunction`实现

MurmurHash算法参考：[https://github.com/aappleby/smhasher](https://github.com/aappleby/smhasher)

### 3. rehash操作

#### 3.1 为什么要rehash?

使用链地址法解决键冲突问题的哈希表，其性能取决于它的负载因子(load_factor)。**负载因子 = 哈希表已有节点数 /  哈希表大小。**

* 负载因子等于1时，哈希表性能最好。
* 负载因子远大于1时，哈希表退化为多个单链表，查找效率大大降低。

* 负载因子远小于1时，哈希表存在大量空链表，浪费空间。

#### 3.2 rehash的执行过程

为了让哈希表的负载因子始终维持在合理范围，Redis通过rehash操作对哈希表进行扩展或收缩操作，rehash的执行过程如下：（参考`dict.c/dictRehash`）

1. 为ht[1]分配空间。注意分配大小始终为2的幂，取决于执行的操作：

   * 如执行扩展操作，分配大小至少为`ht[0]->used`的两倍，参考`dict.c/_dictExpandIfNeeded`

   * 如执行收缩操作，分配大小至少为`ht[0]->used`，参考`dict.c/dictResize`

2. 对于ht[0]中所有的键值对，重新计算每个键的哈希值和索引值，并添加键值对到ht[1]中。

3. 清空ht[0], 将ht[1]设置为ht[0]，在ht[1]上新建一个空白哈希表，完成rehash操作。

#### 3.3 触发rehash的条件

**以下任意一个条件满足时，触发哈希表的扩展操作：**(参考`dict.c/_dictExpandIfNeeded`)

* 服务器没有执行BGSAVE或BGREWRITEAOF命令，且哈希表负载因子大于等于1
* 服务器正在执行BGSAVE或BGREWRITEAOF命令，且哈希表负载因子大于等于`dict_force_resize_ratio`(在3.0版本，这个值为5)

```C
// dict.c
static int dict_can_resize = 1; // 标识字典是否启用rehash，若正在执行BGSAVE操作，值为0，否则为1				
static unsigned int dict_force_resize_ratio = 5; 
static int _dictExpandIfNeeded(dict *d) {
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```

Redis执行BGSAVE命令时，会fork一个子进程，在子进程里执行数据库的持久化操作。为了利用操作系统的[写时复制](https://www.cnblogs.com/biyeymyhjob/archive/2012/07/20/2601655.html)机制，在子进程存在期间，Redis会将负载因子提高，以避免不必要的内存写入。

**当负载因子小于0.1时，触发哈希表的收缩操作：**(参考`dict.c/htNeedsResize`)

```C
// dict.c
#define REDIS_HT_MINFILL        10
int htNeedsResize(dict *dict) {
    long long size, used;
    size = dictSlots(dict);
    used = dictSize(dict);
    return (size && used && size > DICT_HT_INITIAL_SIZE &&
            (used*100/size < REDIS_HT_MINFILL)); // 负载因子小于0.1时，执行收缩操作
}
```

#### 3.4 渐进式rehash

**为什么用渐进式rehash？**

* 字典的rehash过程不是一次性完成的，而是分多次，渐进式地完成。

* 如果哈希表中存在大量的键值对，庞大的计算量会导致服务器阻塞一段时间，用户也无法接受。

##### 渐进式rehash的步骤

渐进式rehash主要通过`_dictRehashStep`和`dictRehashMilliseconds`两个方法实现，步骤如下：

* 首先将`rehashidx`值置为0，表示即将进行rehash操作。
* 每当执行一次字典的添加、删除、查找、或更新操作，程序都会调用`_dictRehashStep`， 将ht[0]->table的`rehashidx`索引上的所有键值对rehash到ht[1]->table, rehash完成后将`rehashidx`属性加1
* 当ht[0]的所有键值对都rehash到ht[1]后，将`rehashidx`置为-1，表示rehash完成。
* 另外，Redis服务器常规任务执行时，会调用`dictRehashMilliseconds`方法，在指定毫秒内对字典执行**主动rehash**。

可以看出，**渐进式rehash采用分治思想，将rehash计算量平摊到对字典的增、删、改、查操作上，避免了一次性rehash的庞大计算量**。

##### 渐进式rehash对哈希表的操作

* rehash过程中，对字典的插入、删除、查找、更新操作会在两个哈希表上同时进行。

* rehash过程中，新添加的键值对只保存在ht[1]中，这保证了ht[0]中的键值对数量只减不增，最终变成空表。

以`dictAddRaw`为例，该函数实现将键插入到字典的功能：

```C
dictEntry *dictAddRaw(dict *d, void *key) {
    int index;
    dictEntry *entry;
    dictht *ht;

    // 执行单步rehash。 渐进式rehash要求对字典的增、删、改、查操作中都尝试进行单步rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 计算键在哈希表中的索引值, 如果值为-1，表示键已经存在立即返回
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;

    // 如字典正在rehash ，新键添加到ht[1]，否则添加到ht[0]
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    // 为新节点分配空间，将新节点插入到链表头，更新哈希表已使用节点数
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;
    dictSetKey(d, entry, key); // 设置新节点的键
    return entry;
}
```

### 字典相关的API

```C
// 创建一个字典
dict *dictCreate(dictType *type, void *privDataPtr);
// 释放字典
void dictRelease(dict *d);
// 添加一个键值对到字典，复杂度O(1)
static int dictAdd(dict *ht, void *key, void *val);
// 先尝试将键值对添加到字典，但如果键已经在字典中，就用新值替换原有值，复杂度O(1)
static int dictReplace(dict *ht, void *key, void *val);
// 删除字典中指定的键值对，复杂度O(1)
static int dictDelete(dict *ht, const void *key);
// 获取指定键的节点值, 复杂度O(1)
void *dictFetchValue(dict *d, const void *key);
// 扩展字典到指定大小
int dictExpand(dict *ht, unsigned long size);
// 缩小指定字典
int dictResize(dict *d);
// 对字典单步rehash, 复杂度O(1)
static void _dictRehashStep(dict *d);
// 毫秒时间内，对字典rehash, 复杂度O(N)
int dictRehashMilliseconds(dict *d, int ms);
```

### 参考资料

【1】《Redis设计与实现》 第4章 字典

【2】[COW奶牛！Copy On Write机制了解一下](https://juejin.cn/post/6844903702373859335)

【3】https://redisbook.readthedocs.io/en/latest/internal-datastruct/dict.html

【4】《数据结构与算法分析 C语言描述》 5.3 分离链接法

【5】[Hash Function](http://www.cse.yorku.ca/~oz/hash.html)























