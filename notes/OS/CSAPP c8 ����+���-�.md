## 异常控制流

### 异常

异常是异常控制流中一种形式，一部分由硬件实现，一部分由操作系统实现。

控制流中的突变，响应处理器状态的某些变化。

异常处理程序完成处理后，根据引起事件的异常和类型，发生如下3种情况中的一种：

* 处理程序将控制返回给当前指令
* 处理程序将控制返回给下一条指令

* 处理程序终止被中断的程序

#### 异常处理

系统中可能的每种类型的异常都分配了一个唯一的非负整数的异常号。

其中一些号码由处理器的设计者分配，如被零除，缺页，内存访问违例。

其他号码由操作系统内核的设计者分配，如系统调用，外部I/O设备的信号。

异常处理程序运行在内核模式下，意味着它们对所有的系统资源都有完全的访问权限。

#### 异常类别

| 类别 | 原因              | 异步/同步 | 返回行为           |
| ---- | ----------------- | --------- | ------------------ |
| 中断 | 来自I/O设备的信号 | 异步      | 总返回到下一条指令 |
| 陷阱 | 有意的异常        | 同步      | 总返回到下一条指令 |
| 故障 | 潜在可恢复的错误  | 同步      | 可能返回到当前指令 |
| 终止 | 不可恢复的错误    | 同步      | 不会返回           |

##### 中断

是异步发生的，这里异步的意思是中断并不是当前指令执行的结果，而是来自处理器外部I/O设备信号的结果

##### 陷阱

陷阱是有意的异常，是执行一条指令的结果。用户程序经常需要向内核请求服务，为允许对内核服务的受控访问，处理器提供一条特殊的`syscall`指令

##### 故障

如处理程序能修正这个错误，就将控制返回到引起故障的指令从而重新执行它；否则处理程序返回到内核的abort例程，abort例程会终止引起故障的应用程序。

一个经典的故障示例：缺页异常。指令引用一个虚拟地址，而与该地址相对应的物理页面不在内存中，因此必须从磁盘取出时，就发生故障。缺页处理程序从磁盘加载适当的页面，然后将控制返回给引起故障的指令。当指令再次执行时，响应物理页面已在内存中，指令就可以无故障地完成了。

##### 终止

不可恢复的致命错误造成的结果，通常是硬件错误，如DRAM或SRAM位被损坏

#### syscall汇编

所有Linux系统调用参数通过通用寄存器而不是栈传递

按惯例，%rax包含系统调用号，`%rdi,%rsi,%rdx,%r10,%r8,%r9`包含最多6个参数。示例：

```assembly
.section .data
string:
	.ascii "hello, world\n"
string_end:
	.euq len, string_end - string
.section .text
.globl main
main:
	movq $1, %rax		# write系统调用号为1
	movq $1, %rdi		# 文件描述符，标准输出
	movq $string, %rsi	# 存储字符串
	movq $len, %rdx		# 保存字符串长度
	syscall
	# Next, call _exit(0)
	movq $60, %rax		# _exit系统调用号为60
	movq $0, %rdi		# exit返回0
	syscall
```

### 进程

经典定义是一个执行中程序的实例。进程提供给应用程序的关键抽象：

* 一个独立的逻辑控制流，提供一个假象：程序独占地使用处理器
* 一个私有的地址空间，提供一个假象：程序独占地使用内存系统

#### waitpid和wait

调用wait等价于调waitpid(-1, &status, 0)

#### 进程休眠

sleep函数将一个进程挂起一段指定时间

```C
unsigned int sleep(unsigned int secs);
```

如果请求时间量已到, sleep返回0，否则返回剩余要休眠的秒数。(因为sleep函数可能被一个信号中断而过早地返回)

pause函数让调用进程休眠，直到进程收到一个信号

```C
int pause(void);
```

#### 加载并运行程序

execve函数在当前进程地上下文中加载并运行一个新程序

```C
int execve(const char *filename, const char *argv[]. const char *envp[]);
```

只有出现错误时，`execve`返回到调用程序，execve仅调用一次且从不返回

#### 程序与进程

程序是一段代码和数据，可以作为目标文件放在磁盘上，或作为段存在于地址空间

进程是执行中程序的一个具体的实例，程序总运行在某个进程的上下文中

##### fork与execve

fork在新的子进程中运行相同的程序，新的子进程是父进程的一个复制品

execve函数在当前进程上下文加载并运行一个新程序，覆盖当前进程的地址空间，并不创建新进程，并继承已打开的文件描述符

### 信号

* 一种软件形式的异常，运行进程和内核中断其他进程。

* 一个信号就是一条小消息，通知进程系统中发生某个类型的事件。

#### 查看信号

```shell
kill -l
```

SIGKILL, SIGSTOP不能被捕获，不能被忽略

传送一个信号到目的进程分两步：

* 发送信号, 原因可以是1)内核检测到一个系统事件 2)一个进程调用了`kill`函数
* 接收信号，目的进程被内核强迫以某种方式对信号的发送做出反应，它就接收了信号。进程可以忽略这个信号，终止或通过执行一个称为信号处理程序的用户层函数捕获这个信号。基本思想如下：

#### 信号是不排队的

* 一个发出而没有被接收的信号叫待处理信号。任何时刻，一种类型至多有一个待处理信号。

* 如一个进程有一个类型为k的待处理信号，那么任何接下来发送到这个进程的类型为k的信号都不会排队等待；它们只是简单被丢弃。

* 一个进程可以有选择地阻塞接收某种信号，当一种信号被阻塞时，它仍可以被发送，但产生地待处理信号不会被接收，直到进程取消对这种信号地阻塞。

* 内核为每个进程在pending位向量中维护着待处理信号集合，在blocked位向量中维护被阻塞的信号集合。只要传送了一个类型为k的信号，内核就设置pending中的第k位，只要接收一个类型为k的信号，内核就会清除pending中的第k位。

#### 进程组

UNIX系统提供了大量向进程发送信号的机制，所有机制基于进程组的概念

##### 如何查看进程的进程组

```shell
ps axo stat,euid,ruid,tty,tpgid,sess,pgrp,ppid,pid,pcpu,comm
```

```C
pid_t getpgrp(void);	// 返回调用进程的进程组ID
```

默认，父子进程同属一个进程组，一个进程可通过setpgid函数改变自己或其他进程的进程组

```C
int setpgid(pid_t pid, pid_t pgid);
```

将进程pid的进程组改为pgid

* 如pid是0，使用当前进程的PID

* 如pgid为0，使用pid指定的进程PID

例如进程12345是调用进程，调用setpgid(0, 0)会创建一个新进程组，其进程组ID是12345，并把进程12345加入到这个新进程中

##### shell发送信号

```shell
kill -9 123				#发送信号9给进程123
kill -9 -123			#一个负的PID会导致信号发送给进程组123中的每个进程
```

##### kill函数发送信号

进程通过调用kill函数发送信号给其他进程(包括自己)

```C
#include <sys/types.h>
#include <signal.h>

int kill(pid_t pid, int sig);			// 返回：若成功返回0，错误返回-1
```

* pid > 0，kill函数发送sig信号给进程pid

* pid == 0, kill函数发送sig信号给调用进程所在进程组中的每个进程
* pid < 0, kill函数发送sig

死机： `kill -19 -1`

##### alarm函数发送信号

```C
unsigned int alarm(unsigned int secs);
```

alarm函数安排内核在secs秒后发送一个SIGALRM信号给调用进程

#### 阻塞和解除阻塞信号

* 隐式阻塞机制 —— 假设程序捕获了信号s, 当前正在运行处理程序S。如果发送给该进程另一个信号s, 那直到处理程序S返回，s会变成待处理而没有被接收
* 显式阻塞机制 —— 应用程序使用sigprocmask函数和它的辅助函数，明确阻塞和解除阻塞选定的信号

```C
#include <signal.h>
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set, int signum);
int sigdelset(sigset_t *set, int signum);
int sigisnumber(const sigset_t *set. int signum);
```

sigprocmask函数改变当前阻塞的信号集合，具体行为依赖how的值：

* SIG_BLOCK —— 把set中的信号添加到blocked中 (blocked = blocked | set)

* SIG_UNBLOCK —— 从blocked中删除set的信号(blocked = blocked &~ set)

* SIG_SETMASK —— blocked=set

如果oldset非空，blocked位向量之前的值保存在oldset中

#### 示例： 使用sigpromask临时阻塞接收SIGINT信号

```
#include<stdio.h>
#include<stdlib.h>
#include<sys/wait.h>

int main(int argc, char **argv)
{
    sigset_t set, old_set;
    sigemptyset(&set);
    sigaddset(&set, SIGINT);
    sigprocmask(SIG_BLOCK, &set, &old_set);
    sleep(5);   // 此时SIGINT信号被阻塞
    sigprocmask(SIG_SETMASK, &old_set, NULL);
    return 0;
}
```

#### 编写信号处理程序原则

* 处理程序尽可能简单 —— 如只是简单设全局标志并返回；
* 处理程序中只调用异步信号安全的函数，要么是可重入的，要么不能被信号处理程序中断
  * 异步信号安全的函数: `man 7 signal`

* 保存和恢复`errno`

* 阻塞所有信号，保护对共享全局数据结构的访问
* volatile声明全局变量。 
* sig_atomic_t声明标志， 保证读和写是原子操作

#### 正确信号处理

**不可以用信号对其他进程中发生的事件计数, 信号是不排队的**

比如回收多个子进程时，对SIGCHLD信号的处理，必须循环waitpid回收进程

#### 可移植的信号处理

Unix信号处理的缺陷在于不同系统上有不同的信号处理语义

* signal函数的语义各有不同，有些老的UNIX系统在信号k被处理程序捕获后就把对信号k的反应恢复到默认值。这些系统上，每次运行之后，处理程序必须调signal函数，显式地重新设置它自己。
* 系统调用可以被中断，某些老版本UNIX系统中，处理程序捕获到信号时，被中断地慢速系统调用(read, write, accept)在信号处理程序返回时不再继续, 而是立即返回给用户一个错误条件，并将errno设置为`EINTR`。这些系统上用户必须手动重启被中断的系统调用代码

```C
int sigaction(int signum, struct sigaction *act, struct sigaction *oldact);
```

#### Signal包装函数

```C
typedef void handler_t(int);
void handler_sigint(int sig) {
    write(STDOUT_FILENO, "SIGINT", 6);
}
handler_t *Signal(int signum, handler_t *handler) {
	struct sigaction act, oact;
	act.sa_handler = handler;
    sigemptyset(&act.sa_mask);
	act.sa_flags = SA_RESTART;
    if(sigaction(signum, &act, &oact) < 0)
        unix_error("Signal error");
    return oact.sa_handler;
}
```

#### sigsuspend函数

```C
int sigsuspend(const sigset_t *mask);
```

sigsuspend函数暂时用mask替换当前的阻塞集合，然后挂起该进程，直到收到一个信号，其行为要么是运行一个处理程序，要么是终止该进程。

* 如行为是终止，该进程不从sigsuspend返回就直接终止
* 如行为是运行一个处理程序，那么sigsuspend从处理程序返回，恢复调用sigsuspend时原有的阻塞集合。

该函数等价于下述代码的原子版本, 原子属性保证第1行和第2行(pause)的调用是一起发生的

```C
sigprocmask(SIG_SETMASK, &mask, &prev);
pause();
sigprocmask(SIG_SETMASK, &prev, NULL);
```

好处在于避免循环的浪费，避免引入pause带来的竞争，又比sleep更有效率，举例

```C
int main() {
    Signal(SIGINT, sigint_handler);
    Signal(SIGCHLD, sigchld_handler);

    sigset_t set, oset;
    sigemptyset(&set);
    sigaddset(&set, SIGCHLD);
    sigprocmask(SIG_SETMASK, &set, &oset);
    if (fork() == 0) {
        exit(0);
    }
    pid = 0;
    sigprocmask(SIG_SETMASK, &oset, NULL);
    while(!pid) {
        sigsuspend(&oset);	// 暂时取消SIGCHLD信号阻塞， 挂起进程，直到SIGCHLD信号处理程序结束后继续运行。
    }
    return 0;
}
```

#### 理解高级语言try, catch机制 —— 非本地跳转

将控制直接从一个函数转移到另一个当前正在执行的函数，而不需要经过正常的调用-返回序列

```C
#include <setjmp.h>
int setjmp(jmp_buf env);		// 调用一次，返回多次。调用时返回0
int sigsetjmp(sigjmp_buf env, int savesigs);
void longjmp(jmp_buf env, int retval);	// 被调用一次，但从不返回
void siglongjmp(sigjmp_buf env, int retval);
```































































