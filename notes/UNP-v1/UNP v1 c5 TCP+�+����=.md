# 掌握简单的TCP客户\服务器echo程序



## netstat命令

WCHAN字段解析:

Linux在进程阻塞于accept和connect, 输出wait_for_connect

阻塞于socket输入和输出时，输出tcp_data_wait

阻塞于终端I/O时，输出read_chan

### POSIX信号处理

信号:告知某个进程发生了某个事件的通知，称为软中断，通常是异步的。

信号可以由一个进程发送到另一个进程；由内核发给某个进程

SIGCHLD：内核在任何一个进程终止时发给它父进程的一个信号

### Sigaction函数

信号处理函数：void handler(int signo);

两个信号不能被捕获：SIGKILL, SIGSTOP

SIG_IGN，SIG_DFL

Signal函数：void (*singal(int signo, void (*func)(int)))(int);

### 步骤

设置处理函数：sigaction结构的sa_handler成员设置为func参数

设置信号掩码：sa_mask设为空集，表示信号处理函数运行期间不阻塞额外信号。POSIX保证被捕获的信号在其信号处理运行期间总是阻塞的

设置SA_RESTART标志

调用sigaction函数

子进程结束时，向父进程发送SIGCHID信号, 父进程调用wait/waitpid获取子进程信息。

 

#### sigaction函数写法

```C
static void func(int signo) {
    // fuck ...
    return;
}
struct sigaction act, oact;
act.sa_handler = func; // 信号处理函数
sigemptyset(&act.sa_mask);
act.sa_flags = 0;
if(sigaction(signo, &act, &oact) < 0) {
    err_sys("sig err");
}
```

### Wait和waitpid函数

Pid_t wait(int *statloc);

Pid_t waitpid(pid_t pid, int *statloc, int options);

返回值: 已终止子进程的进程ID号，statloc返回子进程终止状态

三个检查终止状态的宏：WIFEXITED, EXITSTATUS

### 注意：

如调用wait的进程没有已终止的进程，不过有一个或多个子进程仍在执行，那么wait将阻塞到现有子进程第一个终止为止。

Waitpid函数，pid指定想等待的进程，-1表示等待第一个终止的子进程。Options选项比如WNOHANG，表示告诉内核没有已终止子进程时不要阻塞。

### 常见问题：

 Fork子进程时，必须捕获SIGCHLD信号

捕获信号时，必须处理被中断的系统调用

SIGCHLD信号处理函数必须正确编写，应使用waitpid函数避免僵死进程

当阻塞于某个慢系统调用的一个进程捕获某个信号且相应信号处理函数返回时，该系统调用可能返回一个EINTR错(比如accept)

#### 服务端终止后，再次打开会出现bind: address already in use

可以设置RESUEADDR(bind之前通过setsockopt设置)，不必等待TIME_WAIT状态就可以重启服务器























