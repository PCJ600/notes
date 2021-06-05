# Redis学习笔记： 简单动态字符串

#### 前言

Redis不直接使用C风格字符串，而是使用**SDS(Simple Dynamic Strings, 简单动态字符串)**表示字符串。

SDS主要用于保存数据库中的字符串值，也可用作缓冲区。

Redis通过C语言实现，源码参考[https://github.com/redis/redis](https://github.com/redis/redis)

#### SDS的数据结构定义

 Redis3.0版本，SDS的数据结构设计如下：

```C
struct sdshdr {
	int len;		// 字符串长度
	int free;		// buf数组中的未使用字节数
	char buf[];		// 用于存储字符串，柔性数组(sizeof(struct sdsshr) = 8)
};
```

一个包含"Redis"字符串的SDS示例如下：

![image-20201114134140301](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\image-20201114134140301.png)

图片来源：http://redisbook.com/preview/sds/implementation.html

* buf是一个char类型数组，前5个字节保存"Redis"串，第6个字节保存'\0'
* len的值为5，表示"Redis"串的长度
* free值为5,  表示buf中有5个已分配的字节未使用

注意：**SDS以空字符'\0'结尾，保存空字符的1字节空间不计算在SDS的len属性中**

#### 为什么使用SDS而不是C风格字符串？

##### 1 获取字符串长度的时间复杂度为O(1)

只需访问SDS的len属性就可以获得字符串长度，相比C风格字符串，复杂度从O(N)降至O(1)。

##### 2. 杜绝缓冲区溢出

SDS的API在执行字符串拼接前，会先检查SDS的空间是否足够，如不足会先扩展SDS的空间后才进行拼接操作，这杜绝了缓冲区溢出的可能。（参考`sdscat`的实现）

##### 3. 减少字符串长度修改带来的内存重分配次数

SDS通过**空间预分配**和**惰性空间释放**策略，减少内存重分配次数。

###### 3.1 空间预分配

当SDS空间需扩展时，Redis不仅分配保存新字符串所必须的空间，还会分配额外的未使用空间。

3.0版本中，额外分配的未使用空间大小策略如下：参考`sds.c/sdsMakeRoomFor`

* 如扩展后SDS的长度(即len属性)将小于1024KB，那么redis分配和len属性同样大小的未使用空间。举例，如扩展后，SDS的len将变成100字节, 那么会分配100字节的未使用空间，SDS的buf数组实际长度为100 + 100 + 1 = 201字节。

* 如扩展后SDS的长度(即len属性)超过1024KB，那么redis就分配1024KB的未使用空间。

这种预分配策略，使SDS将连续增长N次字符串所需的内存重分配次数从N次降低为最多N次。

###### 3.2 惰性空间释放

当SDS空间需收缩时，Redis不会立即释放多余的内存，而是仅用free属性记录未使用空间，以便后续使用。

同时，SDS也提供了机制，可以在需要时真正释放SDS的未使用空间，不必担心内存浪费问题。

##### 4. 二进制安全

C风格字符串以'\0'结尾，字符串中间不能包含'\0'，这导致C字符串只能保存文本数据，不能保存二进制数据。

SDS的buf属性也叫字节数组，可保存任意格式的二进制数据。

#### SDS相关API

```C
// 创建一个包含init指向的C风格串的SDS, initlen表示init串的长度
sds sdsnewlen(const void *init, size_t initlen); 
// 创建一个不包含任何内容的空SDS
sds sdsempty(void);
// 真正释放给定的SDS
void sdsfree(sds s);
// 返回SDS的已使用空间的字节数，通过len属性直接获得，复杂度O(1)
static inline size_t sdslen(const sds s);
// 返回SDS的未使用空间的字节数，通过free属性直接获得，复杂度O(1)
static inline size_t sdsavail(const sds s);
// 创建给定SDS的一个副本, 复杂度O(N)
sds sdsdup(const sds s);
// 惰性释放SDS，复杂度O(1)
void sdsclear(sds s);
// 将C风格串拼接到sds末尾, 复杂度O(N)
sds sdscat(sds s, const void *t);
// 将SDS串拼接到另一个sds末尾，复杂度O(N)
sds sdscatsds(sds s, const sds t);
// 将C风格串复制到SDS，复杂度O(N)
sds sdscpy(sds s, const char *t);
// 对比两个SDS串是否相同
sds sdscmp(const sds s1, const sds s2);
// 在s中溢出所有在cset中出现的字符，复杂度O(len(s) * len(cset)) 
sds sdstrim(sds s, const char *cset);
// 判断两个sds串是否相同
int sdscmp(const sds s1, const sds s2);
```

#### Redis 6.0中的SDS实现

在3.0的实现中，SDS结构的len和free属性各占4字节空间。而实际应用中，Redis存储的字符串可能很短，根本不需要用4字节来表示长度, 造成了空间的浪费。

为了进一步节约空间，6.0版本对SDS又细分了5种类型，数据结构设计如下:

```C
typedef char *sds;
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

* 每种类型新增一个8位的`flags`字段，其中低3位表示sds类型，高5位仅在SDS类型为`sdshdr5`时有意义，表示字符串长度。

* 使用`__attribute__ ((__packed__))`，结构体变成按1字节对齐。好处是以时间换空间，并且能通过`buf[-1]`的方式直接访问sds的内部成员flags。

#### 参考资料

《Redis设计与实现》第2章 简单动态字符串

 [http://redisbook.com](http://redisbook.com)

[深挖Redis 6.0源码 —— SDS](https://juejin.im/post/6868450395006599181)

[深入浅出C语言中的柔性数组](https://blog.csdn.net/ce123_zhouwei/article/details/8973073)

[__attribute__ ((__packed__))关键字](https://blog.csdn.net/weixin_39533180/article/details/76207099)

