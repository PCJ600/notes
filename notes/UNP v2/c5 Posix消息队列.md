### 概述

* 消息队列可认为时一个消息链表，有写权限的进程往队列中放置消息，有读权限的进程往队列中读消息

* 具有**随内核的持续性** 

* 队列中每个消息具有三个属性：优先级、数据长度、数据



### mq_open, mq_close, mq_unlink函数

#### mq_open函数

**功能:** 创建一个新的消息队列或打开一个已存在的消息队列。

```C
#include<mqueue.h>
mqd_t mq_open(const char *name, int oflag, ... /*mode_t, struct mq_attr *);
//若成功返回消息队列描述符，出错返回-1
```

* oflag参数是O_RDONLY, O_WRONLY, O_RDWR之一，可能按位或O_CREAT, O_EXCL, O_NONBLOCK

* 实际操作是新队列时，mode和attr参数必需，若attr为空，采用默认属性

* 返回值称为消息队列描述符，该值用作其余消息队列函数的第一个参数

#### mq_close函数

**功能：**用于关闭一个消息队列

```c
#include<mqueue.h>
int mq_close(mqd_t mqdes); //成功返回0, 出错返回-1
```

调用之后，进程不再使用该描述符，但其消息队列并不删除

#### mq_unlink函数

**功能：**删除用作mq_open第一个参数的某个name

```c
#include <mqueue.h>
int mq_unlink(const char *name); //成功返回0, 出错返回-1
```





























































