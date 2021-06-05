### POSIX信号量

semaphore用于提供**不同进程间或给定一个进程不同线程之间**同步手段的原语

一个进程可以在信号量上执行的三种操作：

* **create** - 创建信号量，要求创建者指定初始值

* **wait** - 测试信号量值，若小于等于0就等待 P操作 代表荷兰语proberen，称为递减或上锁，这里使用**等待**术语

* **post** - 将信号量加1 V操作 代表荷兰语verhogen，成为递增或解锁， 这里使用**挂出**术语

#### 信号量与互斥锁的区别

互斥锁总是由锁住它的线程解锁，信号量挂出不必由执行过它的等待操作的同一线程执行

#### Posix信号量类型

* 有名信号量:  sem_open, sem_close, sem_unlink
* 基于内存的信号量:  sem_init, sem_destroy
* 共有函数: sem_wait, sem_trywait, sem_post, sem_getvalue

#### **sem_open函数**

功能：创建一个新的有名信号量或打开已存在的有名信号量，可用于进程或线程间同步

```C
#include<semaphore.h>
sem_t *sem_open(const char *name, int oflag, /* mode_t mode, unsigned int value */);
// 成功返回指向信号量指针，否则返回SEM_FAILED
// value表示信号量的初值
```

#### sem_close函数

功能：关闭打开着的有名信号量

```C
#include<semaphore.h>
int sem_close(sem_t *sem); // 成功返回0，出错返回-1
```

一个进c程终止时，内核对其上仍然打开的所有有名信号量自动执行信号量关闭操作

关闭一个信号量并没有将它从系统中删除，Posix有名信号量至少随内核持续的； 即使没有当前进程打开着某个信号量，它的值仍然保持。

#### sem_unlink函数

功能：从系统中删除有名信号量

```C
#include <semaphore.h>
int sem_unlink(const char *name); // 成功返回0，出错返回-1
```

引用计数大于0时，name能从文件系统中删除，然而信号量的析构要等到最后一个sem_close发生为止

#### sem_wait函数

功能：测试信号量的值，**若该值大于0，将它立即减1并返回，若等于0则休眠，直到该值大于1，再将其减1并返回**

```c
#include <semaphore.h>
int sem_wait(sem_t *sem);
int sem_trywait(sem_t *sem); // 信号量为0时，不休眠，直接返回EAGAIN
// 成功返回0，出错则为-1
```

sem_wait可被信号中断，而过早返回，此时错误为EINTR

#### sem_post函数

功能：一个线程使用完信号量后，需要调用sem_post, 将信号量加1**，唤醒正在等待该信号量变为正数的任意线程**

```C
#include <semaphore.h>
int sem_post(sem_t *sem);
int sem_getvalue(sem_t *sem, int *valp); //若成功返回0, 出错返回-1
```



### 基于内存的信号量

由应用程序分配信号量的内存空间

```C
#include<semaphore.h>
int sem_init(sem_t *sem, int shared, unsigned int value);
int sem_destroy(sem_t *sem);				// 错误返回-1
```

shared为0时，信号量在同一进程的各个线程共享，否则该信号量是在进程间的信号量。

shared为非0时，该信号量必须放在某种类型的共享内存区中，用sem_init初始化

使用完一个信号量之后，我们调用sem_destroy摧毁它

- 如果某个基于内存的信号量是由单个进程的线程间共享的，那么该信号具有随进程的持续性。
- 如果某个基于内存的信号量是由多个进程共享的，只要该信号量所在的共享内存区存在，则该信号量就继续存在

进程间共享信号量的规则：

* **匿名信号量** 本身驻留在共享内存中，且sem_init的第二个参数为1。

* **有名信号量** 总是可由不同进程访问。

















































