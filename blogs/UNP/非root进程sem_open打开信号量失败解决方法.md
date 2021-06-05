# 非root进程sem_open打开信号量失败解决方法

### 问题描述

root进程调`sem_open(XXX, O_CREAT, 0666, 1)`创建信号量后，非root进程使用`sem_open`打开同一个信号量失败，报`Permission Denied`错

### 原因分析

非root进程调用`sem_open`， 以`O_CREAT`方式打开信号量，需要同时有对该信号量文件的读权限 + 写权限。

`ll /dev/shm/sem.semname` 查看信号量文件权限，发现权限为0644，缺少其他用户写权限。这个权限与sem_open中指定的权限值0666不一致。

#### 为什么sem_open中mode参数指定的权限(0666)和创建文件的实际权限(0644)不一致？

首先了解Linux中umask的概念。umask为用户文件创建掩码，是一种进程属性。当进程创建文件或目录时，该属性用于指明应屏蔽的权限位。大多数Linux系统的默认掩码为022，可在shell中通过umask命令查看。umask作用如下：

* 若没有文件掩码，则创建文件的默认权限为0666, 创建目录的默认权限为0777
* 若使用默认掩码022, 则创建文件的权限为0666 - 0022 = 0644, 创建目录的权限为 0777 - 0022 = 0755

### 解决方法

可以在进程调用`sem_open`之前，修改umask值为0，再创建有其他用户写权限的信号量即可。

系统调用umask()可以将进程的umask值改为mask参数所指定的值

```C
#include <sys/stat.h>
mode_t umask(mode_t mask); //调用总是成功，返回值为进程的前一个umask的值
```

写法参考如下：

```C
sem_t *SemOpen() {
	mode_t mask = umask(0);							// 取消屏蔽的权限位
	sem_t *sem = sem_open(XXX, O_CREAT, 0666, 1);	// 创建权限0666的二值有名信号量
	umask(mask);									// 恢复umask的值
	return sem;
}
```

### 参考资料

《Linux/UNIX系统编程手册(上)》 —— 15.4.6 进程的文件模式创建掩码



























