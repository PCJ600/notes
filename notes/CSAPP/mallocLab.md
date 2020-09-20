## mallocLab —— 隐式空闲链表法

### 实验简介

实现自己的动态内存分配器（malloc、free、realloc）

### 知识准备

#### 分配器的要求和目标

##### 要求

* 处理任意请求序列，分配器不可以假设分配和释放请求的顺序
* 立即响应请求, 不允许分配器为了提高性能重新排列或缓冲请求
* 只使用堆
* 对齐块，已保存任何类型的数据对象
* 不修改已分配的块，分配器只能操作和改变空闲块

##### 目标

* 最大化吞吐率 —— 每个malloc, free执行的指令越少，吞吐率会越好
* 最大化内存利用率

#### 碎片

* 内部碎片：已分配块大小和它们有效载荷大小的差值。

* 外部碎片：空闲内存合计起来能满足分配要求，但没有一个单独的足够大的空闲块。

#### 实现问题

需要把握吞吐率和内存利用率之间的平衡

* 空闲块组织 —— 如何记录空闲块
* 放置 —— 如何选一个合适的空闲块来放置一个新分配的块 （首次适配/下次适配/最优适配）
* 分割 —— 将一个新分配块放到某个空闲块后，如何处理这个空闲块的剩余部分
* 合并 —— 如何处理一个刚被释放的块 （立即合并/延迟合并）

### 实验内容

#### 隐式空闲链表法

参考P598 图9-42 隐式链表的恒定形式

##### 为什么叫隐式？

因为空闲块是通过头部中大小字段隐含地连接着，从而间接遍历整个空闲块的集合

#### 具体实现

##### 1. 通用分配器设计

**<font color = 'red'>插图</font>**  隐式空闲链表的恒定形式：P598

```C
static char *mem_heap;					// 指向堆的第一个字节
static char *mem_brk;					// 指向堆的最后一个字节 + 1
static char *mem_max_addr;				// = mem_heap + MAX_HEAP 

// mem_init将可用虚拟内存模型化为一个大数组，mem_heap和mem_brk表示已分配，mem_brk和mem_max_addr表示未分配
void mem_init(void) {
	mem_heap = (char *)Malloc(MAX_HEAP);
   	mem_brk = mem_heap;
    mem_max_addr = mem_heap + MAX_HEAP;
}

// mem_sbrk申请额外的堆内存
void *mem_sbrk(int incr)	// 注意incr单位是1字节
{
    char *old_brk = mem_brk;
    if(incr < 0 || mem_brk + incr > mem_max_addr) {
        return (void *)(-1);
    }
 	mem_brk += incr;   
    return (void *)old_brk;
}
```

##### 2. 创建初始的空闲链表和扩展堆

采用边界标记合并法， 每个块设有头部、脚部、payload, 按8字节对齐原则，一个空闲块最少为16字节

**<font color = 'red'>插图 </font>**先分配16字节，即初始4(对齐块) + 8(序言块) + 4(结尾块) = 16字节

|             |                 |                 |                  |
| ----------- | --------------- | --------------- | ---------------- |
| 4字节对齐块 | 序言块头部(8/1) | 序言块脚部(8/1) | 4字节结尾块(0/1) |

```C
static char *heap_listp;
static char *pre_listp;	// 下一次分配策略，需要在mm_init设置pre_listp的初值为heap_listp

// mm_init创建初始的空闲链表
int mm_init(void) {
    if ((heap_listp = mem_sbrk(4 * WSIZE)) == (void *)-1) {
        return -1;
    }
    PUT(heap_listp, 0);
    PUT(heap_listp + WSIZE, PACK(DSIZE, 1));
    PUT(heap_listp + (2 * WSIZE), PACK(DSIZE, 1));
    PUT(heap_listp + (3 * WSIZE), PACK(0, 1));
    heap_listp += (2 * WSIZE);                      // 指向序言块的脚部
    pre_listp = heap_listp;

    if (extend_heap(CHUNKSIZE / WSIZE) == NULL) {   // 分配4096个字节
        return -1;
    }
    return 0;
}

// extend_heap扩展堆
static void *extend_heap(size_t words) {
    size_t size = (words % 2) ? ((words + 1) * WSIZE) : (words * WSIZE);  //取偶数
    void *p;
    if ((p = mem_sbrk(size)) == (void *)-1) {
        return NULL;
    }
    PUT(HDRP(p), PACK(size, 0));             // 修改老结尾块, 即新分配块头部
    PUT(FTRP(p), PACK(size, 0));             // 设置新空闲块脚部
    PUT(HDRP(NEXT_BLKP(p)), PACK(0, 1));     // 设新的结尾块
    return coalesce(p); // 还要考虑新分配块是否能和前一个块合并
}

```

##### 3.释放和合并块

**<font color = 'red'>插图 </font>**调用mm_free释放块，这里采用立即合并策略， 如图：

* 修改当前块的头部和脚部分配位为0

* 将这个块与它前后的空闲块进行合并

```C
void mm_free(void *ptr)
{
    // 头部和脚部分配位清0, 并尽可能合并空闲块
    int size = GET_SIZE(HDRP(ptr));
    PUT(HDRP(ptr), PACK(size, 0));
    PUT(FTRP(ptr), PACK(size, 0));
    coalesce(ptr);
}
```

调用coalesce合并前后的合并块，分四种情况：

* 前面的块和后面的块都已分配 —— 不可能合并
* 前面的块已分配，后面的块空闲 —— 用当前块和后面块的大小之和更新当前块的头部和后面块的脚部
* 前面的块是空闲的，后面的块是分配的 —— 用两块大小之和更新前面块的头部和后面块的脚部
* 前面和后面的块都是空闲的 —— 用三个块大小之和更新前面块的头部和后面块的脚部

**<font color = 'red'>插图</font>**

**注意如采用下次适配策略，在3、4种情况必须更新pre_listp指针， 否则会payload overlap错**

```C
static void *pre_listp;
static void *coalesce(void *p)
{
    int prev_flag = GET_ALLOC(FTRP(PREV_BLKP(p)));
    int next_flag = GET_ALLOC(HDRP(NEXT_BLKP(p)));
    int size = GET_SIZE(HDRP(p));

    if (prev_flag && next_flag) {
        ;
    } else if (prev_flag && !next_flag) {
        size += GET_SIZE(HDRP(NEXT_BLKP(p)));
        PUT(HDRP(p), PACK(size, 0));
        PUT(FTRP(p), PACK(size, 0));

    } else if (!prev_flag && next_flag) {
        size += GET_SIZE(HDRP(PREV_BLKP(p)));
        PUT(FTRP(p), PACK(size, 0));
        PUT(HDRP(PREV_BLKP(p)), PACK(size, 0));
        p = PREV_BLKP(p);
    } else {
        size += GET_SIZE(HDRP(PREV_BLKP(p))) + GET_SIZE(FTRP(NEXT_BLKP(p)));
        PUT(HDRP(PREV_BLKP(p)), PACK(size, 0));
        PUT(FTRP(NEXT_BLKP(p)), PACK(size, 0));
        p = PREV_BLKP(p);
    }
    pre_listp = p;  // 如采用下次适配，这里注意更新pre_listp的值，以免出错
    return p;
}
```

##### 4.分配块

注意：

1. 需最少分配16字节，因为其中8字节满足对齐要求，另8字节用来放头部和脚部
2. 对于超过8字节的请求，需向上舍入到8的整数倍

```C
void *mm_malloc(size_t size)
{
    if (size == 0) {
        return NULL;
    }

    // 最少分配16字节，8字节满足对齐，另8字节用来放头部和脚部
    int asize;
    if (size <= DSIZE) {
        asize = 2 * DSIZE;
    } else {
        asize = DSIZE * ((size + (DSIZE) + (DSIZE-1)) / DSIZE);
    }

    // 根据请求的size搜索空闲链表，寻找一个合适的空闲块，放置这个请求块并可选地分隔空闲块
    void *p;
    if ((p = fit(asize)) != NULL) {
        place(p, asize);
        return p;
    }

    // 如分配器不能发现一个空闲块，只能扩展堆并分配一个新的空闲块
    if ((p = extend_heap(MAX(CHUNKSIZE, asize) / WSIZE)) == NULL) {
        return NULL;
    }
    place(p, asize);
    return p;
}
```

##### 适配算法

可以采用首次适配或下一次适配

* 首次适配从头搜索空闲链表，选择第一个合适地空闲块

* 下一次适配从上次查询结束的地方开始搜索空闲链表

```C
// 首次适配法
static void *first_fit(int asize)
{
    // 结尾块大小位0, 分配位为1, 表示已经结束
    for (void *ptr = heap_listp; GET_SIZE(HDRP(ptr)) > 0; ptr = NEXT_BLKP(ptr)) {
        if (!GET_ALLOC(HDRP(ptr)) && asize <= GET_SIZE(HDRP(ptr))) {
            return ptr;
        }
    }
    return NULL;
}
        
// 下一次适配法
static void *next_fit(int asize) {
    // 结尾块大小位0, 分配位为1, 表示已经结束
    for (void *ptr = pre_listp; GET_SIZE(HDRP(ptr)) > 0; ptr = NEXT_BLKP(ptr)) {
        if (!GET_ALLOC(HDRP(ptr)) && asize <= GET_SIZE(HDRP(ptr))) {
            pre_listp = ptr;
            return ptr;
        }
    }
    for (void *ptr = heap_listp; ptr != pre_listp; ptr = NEXT_BLKP(ptr)) {
        if (!GET_ALLOC(HDRP(ptr)) && asize <= GET_SIZE(HDRP(ptr))) {
            pre_listp = ptr;
            return ptr;
        }
    }
    return NULL;
}
```

##### place函数

**<font color = 'red'>插图</font>**

如果分割后剩下的块大于或等于16字节，就必须执行分割操作

```C
static void place(void *p, size_t asize)
{
    int osize = GET_SIZE(HDRP(p));
    if (osize - asize < 2 * DSIZE) {
        PUT(HDRP(p), PACK(osize, 1));
        PUT(FTRP(p), PACK(osize, 1));
        return;
    }
   	// 如果分割后剩下的块大于或等于16字节，就必须执行分割操作
    PUT(HDRP(p), PACK(asize, 1));
    PUT(FTRP(p), PACK(asize, 1));
    p = NEXT_BLKP(p);
    PUT(HDRP(p), PACK(osize - asize, 0));
    PUT(FTRP(p), PACK(osize - asize, 0));
}
```

##### mm_realloc函数

可以通过简单调用mm_malloc和mm_free实现, 要求参考writeup

```C
void *mm_realloc(void *ptr, size_t size)
{
    void *oldptr = ptr;
    void *newptr;
    size_t copySize;

    newptr = mm_malloc(size);
    if (newptr == NULL) {
		return NULL;
    }
    copySize = *(size_t *)((char *)oldptr - SIZE_T_SIZE);
    if (size < copySize) {
    	copySize = size;
    }
    memcpy(newptr, oldptr, copySize);
    mm_free(oldptr);
    return newptr;
}
```

#### 实验结果

```txt
# ./mdriver -t traces/ -v
Results for mm malloc:
trace  valid  util     ops      secs  Kops
 0       yes   90%    5694  0.001137  5009
 1       yes   93%    5848  0.000733  7975
 2       yes   94%    6648  0.001964  3386
 3       yes   96%    5380  0.001920  2802
 4       yes   66%   14400  0.000225 64114
 5       yes   89%    4800  0.003227  1487
 6       yes   87%    4800  0.003000  1600
 7       yes   55%   12000  0.008883  1351
 8       yes   51%   24000  0.004652  5159
 9       yes   26%   14401  0.050291   286
10       yes   34%   14401  0.001779  8094
Total          71%  112372  0.077811  1444

Perf index = 43 (util) + 40 (thru) = 83/100
```



#### trace文件格式

mdriver.c: read_trace函数解析

每个trace文件每行作用如下：

```
4000000						# 表示建议堆大小，没有用
2690						# id索引个数
5380						# 操作数， 等于文件非空行数 - 4
1							# 表示trace文件权重，没有用
a 0 2040					# alloc, index = 0, 分配2040字节,
f 208						# free, index = 208
```



