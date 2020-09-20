## ShellLab 

### 实验目标

实现一个带有作业控制的Unix shell，包括Ctrl + C, Ctrl + Z，实现jobs, fg, bg命令。通过实验了解Unix的进程控制、并发、信号处理

### 预备知识

* Ctrl + C, Ctrl + Z

* waitpid、kill、sigprocmask



### 实验准备



### 实验内容

基于框架代码 完成几个函数的实现：

* eval
* builtin_cmd
* do_bgfg
* waitfg

* sigchld_handler, sigint_handler, sigtstp_handler



#### dup2函数

功能：复制一个现有的文件描述符

```C
int dup2(int fd, int fd2);
dup2(1, 2);	 							//将stderr也重定向到stdout
```



### TIPS

子进程退出都进入中断处理程序sigchld_handler, 如是前台程序直接sleep等待OK

sigchld_handler中waitpid选项

WNOHANG表示立即返回

WUNTRACED表示获取停止状态进程信息



































