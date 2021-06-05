## 文件I/O

#### 常用五个文件I/O函数

open , read, write, lseek, close

#### unbuffered I/O

指每一次read, write进入内核，执行一次系统调用

#### 文件描述符

当打开一个现有文件或创建新文件时，内核向进程返回一个文件描述符，是一个非负整数

幻数0,1,2分别表示STDIN_FILENO, STDOUT_FILENO, STDERR_FILENO

文件描述符变化0~OPEN_MAX-1

#### open函数

```C
#include<fnctl.h>
// 成功返回fd, 失败返回-1
int open(const char *path, int oflag, ... /* mode_t mode */);
int openat(int fd, const char *path, int oflag, ... /* mode_t mode */)
```

path参数表示打开或创建文件的名字

oflags参数将下列多个常量进行或运算构成oflag参数， 以下是常用参数

O_CREAT 若此文件不存在则创建它

O_EXCL 若同时指定O_CREAT，而文件已存在，则出错。可以测试一个文件是否存在

O_RDONLY 只读打开，O_WRONLY只写打开，O_RDWR读写打开

O_APPEND 每次写追加到文件尾端

O_TRUNC 如此文件存在，且为只写或读写成功打开，将其长度截断为0

O_NONBLOCK 非阻塞

O_NOFOLLOW 如引用的为符号链接，则出错

#### creat函数

```C
#include<fcntl.h>
int creat(const char *path, mode_t mode); // 成功返回fd, 失败返回-1
从函数等效于open(path, O_WRONLY|O_CREAT|O_TRUNC, mode);
```

creat函数以**只写形式**打开文件, 不足之处在于如果创建临时文件，并要先写后读，则必须先调用creat,close，然后调用open， 现在可以使用open(path, O_RDWR|O_CREAT|O_TRUNC, mode)替代creat函数

#### close函数

```c
#include<fcntl.h>
int close(int fd); // 成功返回0，失败返回-1
```

关闭一个文件还将释放该进程在文件上的所有记录锁

当进程终止时，自动关闭它所有打开的文件，无需显式指定close

#### lseek函数

```C
#include<unistd.h>
off_t lseek(int fd, off_t offset, int whence); //成功返回新的偏移量，失败返回-1
```

off_t：32位long类型，64位long unsigned int类型

whence三个常值:

SEEK_SET，将偏移量置为据文件开始的offset个字节

SEEK_CUR，将偏移量置为其当前值加offset, offset可正可负

SEEK_END， 将偏移量设为文件长度加offset，offset可正可负

lseek仅将当前文件的偏移量记录在内核中，不引起I/O操作，该偏移量用于下一次读写

#### read函数

```C 
#include<unistd.h>
ssize_t read(int fd, void *buf, size_t nbytes);
// 从打开文件中读数据，成功返回读取的字节数，已到文件尾返回0，失败返回-1
```

什么情况读到字节数少于要求读的字节数

* 读普通文件时，在读到要求字节数之前已到文件尾
* 从终端设备读时，通常一次读一行
* 从网络读时，网络中缓冲机制造成返回字节数小于要求读的字节数
* 信号造成中断，已读了部分数据量时

#### write函数

```C
#include<unistd.h>
ssize_t write(int fd, const void *buf, size_t nbytes);
// 若成功，返回已写的字节数，失败返回-1
```

出错常见原因：磁盘已满或超过一个给定进程的文件长度限制

#### 文件共享

理解进程表项、文件表项、v结点表项

![image-20200412150042151](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\image-20200412150042151.png)

##### pread和pwrite函数

```C
#include <unistd.h>
ssize_t pread(int fd, void *buf, size_t nbytes, off_t offset);
ssize_t pwrite(int fd, void *buf, size_t nbytes, off_t offset);
```

调用pread相当于先调lseek，后调read，但区别如下:

* pread调用时，无法中断其定位和读操作
* 不更新当前文件偏移量

#### dup和dup2函数

用来复制一个现有文件描述符

```C
#include <unistd.h>
int dup(int fd);
int dup2(int fd, int fd2);
// 若成功，返回新的文件描述符，出错返回-1
```

dup返回的一定是当前可用文件描述符中的最小数值

对于dup2，用fd2指定新描述符值

dup函数返回值与参数fd共享同一个文件表项

dup(fd) 等效于 fcntl(fd, F_DUPFD, 0); dup2函数一定是原子操作，不等同于close+fcntl

#### sync, fsync, fdatasync函数

传统UNIX系统实现在内核中设有缓冲区高速缓存或页高速缓存。**向文件写数据时，内核先将数据复制到缓冲区，然后排入队列，晚些时候写入硬盘，这种方式称为延迟写**

内核会在合适时机把脏页的数据写到磁盘中，以保持高速缓存中的数据和磁盘中数据的一致性，用到以下三个函数

```C
#include<unistd.h>
int fsync(int fd);		//若成功返回0，错误返回-1
int fdatasync(int fd);  //若成功返回0，错误返回-1
void sync(void); 
```

sync只将所有修改过的块缓冲区排入写队列，然后返回，并不等待实际写磁盘操作结束

fsync只对fd指定的那一个文件一起做，并等待实际写磁盘操作结束才返回

fdatasync功能与fsync类似，但它只影响文件数据部分，除数据外，fsync会同步更新文件的属性

#### fcntl函数

用于改变已经打开文件的属性

```C
#include<fcntl.h>
int fcntl(int fd, int cmd, ... /* arg */); // 出错返回-1
```

常用功能：

复制已有fd: cmd=F_DUPFD

获取/设置fd标志 cmd=F_GETFD, F_SETFD

获取/设置文件状态 cmd=F_GETFL, F_SETFL

举例，setfl函数实现

```C
void set_fl(int fd, int flags) {
	int val;
	if((val = fcntl(fd, F_GETFL, 0)) < 0) {
		err_sys("fcntl F_GETFL error");
	}
	val |= flags;
	if(fcntl(fd, F_SETFL, val) < 0) {
		err_sys("fcntl F_SETFL error");
	}
}
```

#### ioctl函数 —— I/O操作的杂物箱

对设备进行操作的方法

```C
#include<sys/ioctl.h>
int ioctl(int fd, int request, ...); //出错返回-1
```

#### 习题3.5

```shell
./a.out > outfile  2>&1 #stderr, stdout均重定向到outfile
./a.out  2>&1 > outfile #2重定向到终端，1重定向到outfile
```

文件描述符1与标准输出关联，文件描述符2与标准出错关联

1> 可以简写为>， 两种差异根据内核打开文件的数据结构区分(进程表项，文件表项，v-node)

前者设置标准输出outfile, 然后执行dup将标准输入复制到2上，结果将标准输出和标准出错设置为同一个文件

后者首先执行dup，2成为终端，标准输出重定向到outfile ，结果是1指向outfile，2指向终端



























































#### <font color='red'>TOCTTOU </font>

time-of-check-to-time-of-use错误

基本思想：如有两个基于文件的函数调用，第二个调用依赖第一个调用，那么程序是脆弱的，因为这两个调用并不是原子操作，两个调用之间文件改变了，会造成第一个调用结果不再有效





























