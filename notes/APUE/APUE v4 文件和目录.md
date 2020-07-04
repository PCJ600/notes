

## 文件和目录

#### stat, fstat, fstatat, lstat (man fstat)

```C
#include<sys/stat.h>
int stat(const char *restrict pathname, struct stat *restrict buf);
int lstat(const char *restrict pathname, struct stat *restrict buf);
```

stat函数返回文件信息结构，lstat类似于stat，但当一个文件为软链接时返回该符号链接信息而不是引用文件信息

#### 文件类型

普通文件 - 数据本身是文本还是二进制，对内核而言并无区别，对普通文件解释由处理该文件应用程序进行!

类型：目录文件，块特殊文件，字符特殊文件，FIFO，套接字，符号链接

文件信息类型在st_mode字段中，<sys/stat.h>中文件类型宏如下：

| 宏        | 文件类型 |
| --------- | -------- |
| S_ISREG() | 普通文件 |
| S_ISDIR() | 目录文件 |
| S_ISLNK() | 符号链接文件 |

#### 设置用户ID和设置组ID

与一个进程有关的ID有6个或更多，如图所示
| 实际用户ID，实际组ID | 我们实际上是谁 |
| 有效用户ID，有效组 | 用于文件访问权限检查 |
| 保存的设置用户ID，保存的设置组ID | 由exec函数保存 |

* 通常实际用户ID等于有效用户ID，实际组ID等于有效组

* 每个文件有一个所有者和组所有者，所有者由stat结构的**st_uid**指定， 组所有者由**st_gid**指定

* 可以在st_mode中设置特殊标志，含义是当执行此文件时，将进程的有效用户ID设置为文件所有者的用户ID(st_uid)
* stat函数中，设置用户ID位和设置组ID位都包含在st_mode中，两位分别用**常量**S_ISUID, S_ISGID测试
  * 设置 st_mode | S_ISUID ，获取 st_mode & S_ISUID，清零 st_mode | ~S_ISUID

#### 文件访问权限



















#### <font color = 'red'>问题</font>

与一个进程关联的6个ID含义、区别？























































#### <font color='red'>TOCTTOU </font>

time-of-check-to-time-of-use错误

基本思想：如有两个基于文件的函数调用，第二个调用依赖第一个调用，那么程序是脆弱的，因为这两个调用并不是原子操作，两个调用之间文件改变了，会造成第一个调用结果不再有效





























