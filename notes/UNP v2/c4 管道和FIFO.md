#### 管道

```C
#include<unistd.h>
int pipe(int fd[2]);	//成功返回0，失败返回-1
```

一般为半双工， 其中fd[0]用于读，fd[1]用于写

典型用途：父子进程通信，掌握echo程序写法

#### popen函数与pclose函数

```C
#include<stdio.h>
FILE *popen(const char *cmd, const char *type); //返回:失败返回NULL
int pclose(FILE *stream); // 若成功返回为shell的终止状态，出错返回-1
```

type为r, 调用进程读进cmd的标准输出

type为w,调用进程写到cmd的标准写入

典型用途：shell管道线，popen执行shell命令

#### FIFO

First In First Out, **半双工**数据流，每个FIFO有一个路径名与之关联，从而允许无亲缘关系的进程访问同一个FIFO，也称为**有名管道**，由mkfifo函数创建

```C
#include<sys/type.h>
#include<sys/stat.h>
int mkfifo(const char *pathname,mode_t mode);// 成功返回0，错误返回-1
```

* mkfifo函数已隐含**O_CREAT**|O_EXCL，要么创建一个新的FIFO，要么返回一个EEXIST错

* 半双工，若对FIFO调用lseek，返回ESPIPE错

#define FILE_MODE (S_IRUSR|S_IWUSR|S_IRGRP|S_IROTH)

##### 注意点

若当前没有进程打开某个FIFO来写，打开该FIFO来读的进程将阻塞

**具有随进程的持续性**，写入数据结束进程后，读进程读不出任何数据

#### 设置非阻塞状态

```C
int flags;
if((flags = fcntl(fd, F_GETFL, 0)) < 0) {
	err_sys("F_GETFL error");
}
flags |= O_NONBLOCK;
if(fnctl(fd, F_SETFL, flags) < 0) {
	err_sys("F_SETFL, error");
}
```

#### 关于管道和FIFO的若干规则

若请求读出的数据量多于管道或FIFO中可用数据量，那只返回可用数据

若请求写入数据字节少于PIPE_BUF,write操作保证原子性

FIFO是一种只能在单台主机上使用的IPC形式，尽管在文件系统上有名字，也只能用在本地文件系统上，不能用在通过NFS安装的文件系统上

管道和FIFO均采用**字节流模型**

限制

* OPEN_MAX 一个进程在任意时刻打开的最大描述符数

* PIPE_BUF 可原子性地往一个管道或FIFO的最大数据量



































































