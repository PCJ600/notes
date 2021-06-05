## Redis学习笔记：整数集合

[TOC]

### 前言

Redis中，整数集合是集合键的底层之一。

当**一个集合只包含整数元素，且这个集合中元素个数不多**的情况下，Redis就会使用整数集合作为集合键的底层实现。

### 整数集合的实现

#### 0. 数据结构设计

以Redis6.0源码为例，整数集合由`intset.c/intset`结构定义，数据结构设计如下：

```C
typedef struct intset {
    uint32_t encoding;			// 编码方式
    uint32_t length;			// 集合中的元素个数
    int8_t contents[];			// 用于保存集合中的元素
} intset;
```

* length属性表示整数集合中元素的个数。
* 集合中所有元素以**有序、无重复**的方式保存在contents数组。
* **虽然contents是int8_t类型，但它本身不保存int8_t类型的元素！**实际上，contents数组保存的元素类型由encoding属性决定，encoding属性有以下3种取值：

```C
// intset.c
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```

举例，如encoding取值为`INTSET_ENC_INT32`，表示contents数组保存的每个元素都是int32_t类型的整数。

下图表示一个包含3个类型为uint16_t的元素的整数集合：

![image-20201206160908210](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\image-20201206160908210.png)当创建空的整数集合时，为了节约内存，默认的encoding类型为`INTSET_ENC_INT16`(参考`intset.c/intsetNew`的实现)

#### 1. 升级操作

考虑插入新元素a到整数集合S的场景：如果新元素所占字节大小大于整数集合中现有的任意一个元素所占字节大小(即`sizeof(a) > S.encoding`），就需要先对整数集合执行**升级(upgrade)**操作后，再执行插入操作。

升级操作的要点如下：

* 根据新元素类型，计算扩展后的整数集合需要分配的空间大小，并调用realloc分配空间。
* 将原整数集合中所有元素类型转换为和新元素相同，并将所有转化后的元素有序地放到正确的位置上。
* 将新元素添加到整数集合。**升级后新元素放置的位置要么在最开头，要么在最结尾**，其原因在于：**能引发升级的新元素，它的值要么小于先前整数集合中的所有元素，要么大于所有元素。**

升级操作的源码实现参考`intset.c/intsetUpgradeAndAdd`, 如下：

```C
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding);
    uint8_t newenc = _intsetValueEncoding(value);
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize */
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

另外，Redis中的整数集合不支持降级操作。

### 整数集合的API

| API                                                          | 功能                                       | 复杂度  |
| ------------------------------------------------------------ | ------------------------------------------ | ------- |
| intset *intsetNew(void);                                     | 创建空的整数集合                           | O(1)    |
| intset *intsetAdd(intset *is, int64_t value, uint8_t *success); | 插入新元素到整数集合                       | O(N)    |
| intset *intsetRemove(intset *is, int64_t value, int *success); | 删除整数集合中的指定元素                   | O(N)    |
| uint8_t intsetFind(intset *is, int64_t value);               | 查询元素是否在整数集合中。用二分查找法实现 | O(logN) |


### 参考资料

【1】《Redis设计与实现》 —— 第6章 整数集合

