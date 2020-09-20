## UNIX基础知识

#### 内核(kernel)

将操作系统定义为一种软件，它控制计算机资源，提供程序运行环境。通常将这种软件叫内核，因为它相对较小，且处于环境核心

#### shell

命令行解释器，读取用户输入，然后执行命令

#### 文件描述符(file descriptors)

内核用以标识一个特定进程正在访问的文件，通常是一个小的非负整数。当内核打开一个现有文件或创建新文件时，都返回一个文件描述符。

#### 标准输入、标准输出、标准错误

每当运行新程序，shell为其打开三个文件描述符，默认都指向终端

#### 程序(Program)

程序是一个存储在磁盘上某个目录的可执行文件，内核使用exec系列函数，将程序读入内存，执行程序

#### 进程和进程ID(Processes and Process ID)

程序的执行实例称为进程，进程ID指每个进程唯一数字标识符

#### 进程控制函数(Process Control)

fork, exec, waitpid

#### 线程和线程ID(Threads and Threads ID)

一个进程内的多个线程共享同一地址空间，文件描述符，栈等资源

#### 出错处理(Error Handling)

关于errno的两条规则：

* 如没有出错，其值不会被例程清除，因此仅当返回值指明出错时，才检验其值

* 任何函数不会将errno值设为0，且在<errno.h>中定义所有的常量均不为0

```C
#include<string.h>
#include<stdio.h>
char *strerror(int errnum); // 将errno映射程一个出错消息字符串，并返回此字符串的指针
void perror(const char *msg); // 基于errno，在标准错误上产生出错信息，然后返回。一般msg传argv[0]
```

#### 用户标识

root用户的ID为0

通过getuid()查看user ID, getgid()查看group ID

#### 信号

用于通知进程发生了某种情况。

三种处理信号方式：

* 忽略， 不推荐此方式

* 按系统默认方式处理

* 自己编写信号处理函数，捕捉信号

终端键盘上产生信号的方式

* 中断键：Delete键和Ctrl+C键

* 退出键：Ctrl+\键

* 调用kill函数传递信号。向一个进程发送信号时，必须是这个进程的所有者或超级用户，否则报Operation not permitted错

#### 时间值

日历时间： UTC(Coordinated Universal Time协调世界时间)，自1970.1.1 00:00:00以来的秒数

#### 系统调用(system call)

The interface to the kernel is a layer of software called the system calls(内核的接口被称为系统调用)

库函数可能会调用一个或多个系统调用，但不是内核的入口点

#### 参考资料

《UNIX环境高级编程第三版》

[http://www.tutorialspoint.com/c_standard_library/c_function_signal.htm](http://www.tutorialspoint.com/c_standard_library/c_function_signal.htm)

