# Redis学习笔记：链表

[TOC]

#### 前言

因为C语言没有内置链表的数据结构，Redis自己实现了链表结构。

链表是Redis**列表键**的底层实现之一；发布与订阅，慢查询，监视器等功能也要用到链表。

#### 链表的数据结构定义

链表节点的数据结构如下：

```C
// 源码参考adlist.h
typedef struct listNode {		
	struct listNode *prev;		// 前驱指针
	struct listNode *next;		// 后继指针
	void *value;				// 节点的值 
}listNode;
```

Redis使用`struct list`结构持有整个链表，数据结构如下：

```C
// 源码参考adlist.h
typedef struct list {
    listNode *head;							// 头节点
    listNode *tail;							// 尾节点
    void *(*dup)(void *ptr);				// 用于复制节点
    void (*free)(void *ptr);				// 用于释放节点
    int (*match)(void *ptr, void *key);		// 用于比较节点值是否相同
    unsigned long len;						// 链表中的节点总数
} list;
```

下图表示一个长度为3的链表：(图片来源 http://redisbook.com/preview/adlist/implementation.html)

![image-20201117225830734](C:\Users\pc\Desktop\image-20201117225830734.png)

##### 可以看出，Redis实现的链表特性如下：

* **双向**、**无环**、且**带表头和表尾指针**。

* 用len属性对链表节点计数，获取链表节点数的复杂度为O(1)。

* 多态，节点值用`void *`类型存储，可保存不同类型的值。

#### 链表相关的API

```C
// 返回链表中的节点个数，复杂度O(1)
#define listLength(l) ((l)->len)
// 创建空链表，复杂度O(1)
list *listCreate(void);
// 新增节点到链表头, 复杂度O(1)
list *listAddNodeHead(list *list, void *value);
// 新增节点到链表尾, 复杂度O(1)
list *listAddNodeTail(list *list, void *value);
// 新增节点到给定节点之前或之后，复杂度O(1)
list *listInsertNode(list *list, listNode *old_node, void *value, int after);
// 查找并返回链表中包含给定值节点，如失败返回NULL，复杂度O(N)
listNode *listSearchKey(list *list, void *key);
// 删除给定节点，复杂度O(1)
void listDelNode(list *list, listNode *node);
// 取出表尾节点，将它移动到表头，成为新的表头节点, 复杂度O(1)
void listRotate(list *list);
// 复制整个链表，复杂度O(N)
list *listDup(list *orig);
// 释放给定链表，以及链表中所有的节点
```

#### 参考资料

《Redis设计与实现》第３章 链表

Redis 3.0 源码注释：https://github.com/huangz1990/redis-3.0-annotated
