# Redis学习笔记：跳跃表

[TOC]

### 前言

跳跃表是**一种以O(log N)期望时间支持查找、插入、删除操作的、有序的**数据结构。

Redis使用跳跃表作为**有序集合键**的底层实现之一。 

跳表的基本实现原理参考：[《Skip lists: a probabilistic alternative to balanced trees》](http://www.cl.cam.ac.uk/teaching/0506/Algorithms/skiplists.pdf)

### Redis中的跳表实现

Redis的跳表由zskiplistNode, zskiplist两个数据结构定义。

跳跃表节点的实现如下，由`redis.h/zskiplistNode`定义：

```C
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;							// 成员对象，唯一
    double score;						// 跳表按分值排序，不唯一
    struct zskiplistNode *backward;		// 后退指针，用于表尾到表头访问
    struct zskiplistLevel {
        struct zskiplistNode *forward;	
        unsigned long span;				// 跨度
    } level[];							// 层
} zskiplistNode;
```

Redis中跳表和普通跳表区别如下：

* 添加span属性，表示跨度，用于计算排位。
* 添加backward后退指针，用于表尾到表头访问节点，每次只能后退一个节点。
* ele表示SDS对象，必须是唯一的，而普通跳表存储的值可以不唯一。 
* 添加score属性，表示分值，这是Redis跳表排序的依据，score的值允许重复。如score值相同，按ele字典序排列

Redis通过zskiplist结构来持有跳表：

```C
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;	// 定位表头、表尾复杂度为O(1)
    unsigned long length;					// O(1)获取跳表长度
    int level;								// 层高，注意表头节点的层高不计算在内
} zskiplist;
```

### 跳表API

| API                                                          | 功能               | 复杂度  |
| ------------------------------------------------------------ | ------------------ | ------- |
| zskiplist *zslCreate(void);                                  | 创建跳表           | O(1)    |
| void zslFree(zskiplist *zsl);                                | 释放跳表           | O(1)    |
| zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele); | 插入               | O(logN) |
| int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node); | 删除跳表           | O(logN) |
| unsigned long zslGetRank(zskiplist *zsl, double score, sds ele); | 返回给定节点的排位 | O(logN) |
| zskiplistNode* zslGetElementByRank(zskiplist *zsl, unsigned long rank); | 返回指定排位的节点 | O(logN) |

#### 参考资料

《Redis设计与实现》第5章 跳跃表





