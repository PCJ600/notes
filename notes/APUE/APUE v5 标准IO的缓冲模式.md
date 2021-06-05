# 标准I/O库的缓冲模式

## 问题描述

有时候，代码中明明执行了printf语句打印到终端，却没有看到输出的内容。

写文件的时候，明明成功执行了fwrite, fprintf语句，文件却没有写入相应的内容。

想搞清楚这些问题产生的原因，需要了解标准I/O库的缓冲模式。

## 标准I/O与unbuffered I/O

linux对I/O文件操作分为不带缓存I/O(unbuffered I/O)和带缓存I/O(即标准I/O)

《APUE》中对术语unbuffered的定义: "The term *unbuffered* means that each read or write invokes a system call in the kernel"

这篇文章讲了unbuffered I/O和标准I/O的区别https://blog.csdn.net/qq_33366098/article/details/77923722，以下引用其中的描述：

不带缓存I/O，是指**每次read, write都会进入内核，执行一次系统调用，不带缓存不是指直接对磁盘进行读写**。比如read,write函数，它们属于系统调用，在用户态没有缓存，但是在内核是有缓存器的。如内核缓存未满，写入的数据还是在内核缓存，并没有真正写入硬盘。需要等待缓存写满或者内核需要重用该缓存以存放其他磁盘块数据时，才进行实际硬盘读写，这种方式被称为延迟写(delayed write)

带缓存I/O也叫标准I/O。标准I/O会在用户态建立一个缓存区，以尽可能减少read和write调用的次数，提高效率。

unbuffered I/O操作数据流向：数据->内核缓存区->磁盘

标准I/O操作数据流向：数据->流缓存区->内核缓存区->磁盘

## 缓冲

标准I/O库提供三种模式的缓冲: 全缓冲、行缓冲、不带缓冲

### 全缓冲(fully buffered)

这种情况下，在填满标准I/O缓冲区后才进行实际I/O操作。对于驻留在磁盘上的文件通常用全缓冲。

### 行缓冲(line buffered)

这种情况下，当输入或输出遇到换行符，或者缓冲区已满时进行实际I/O操作。当流涉及一个终端时，通常使用行缓冲。

### 不带缓冲(unbuffered)

这种情况下，标准I/O库不对字符进行缓冲存储。例如标准出错流stderr通常是不带缓冲的，这使得出错信息可以尽快显示。

### 注：

1.这里的实际I/O操作**不是指读写硬盘操作，而是指执行read, write系统调用**。

2.**缓冲类型与具体的标准I/O函数无关，与读写的文件类型有关**。

## 举例说明

### 例1 全缓冲

```
#include<stdio.h>
#include<unistd.h>
int main() {
	char str[] = "hello world";
	FILE *fp = fopen("./text", "w+");	// 省略了判空操作^_^
	fprintf(fp, "%s\n", str);
    // fflush(fp); 
	for( ; ; ) {						
		sleep(10);
	}
	return 0;
}
```

编译并运行程序，进入死循环后，观察text文件发现内容为空，hello world字符串并没有写入。

这是因为对于磁盘上的文件默认是全缓冲的。因为写入的字符串长度小于缓冲区大小(我的ubuntu机器上，为4096字节)，所以不会直接写入文件。

如需要立即输出，可以在for循环之前调用fflush函数，将缓冲区的内容写入磁盘。

这解释了为什么有时明明成功执行了fwrite,fprintf语句，文件却没有写入相应的内容。

### 例2 行缓冲

```
#include<stdio.h>
#include<unistd.h>
int main() {
	fprintf(stdout, "hello world");
	for( ; ; ) {						
		sleep(10);
	}
	return 0;
}
```

编译并运行程序，发现终端没有输出，即使fprintf已经执行。

这是因为涉及终端的流默认是行缓冲的，当输入或输出遇到换行符时才进行实际I/O操作。

如果需要执行fprintf后立即打印，只需在"hello world"后添加换行符'\n'

这个例子解释了为什么有时代码中执行了printf语句打印到终端，却没有看到输出的内容。

### 例3 无缓冲

```
#include<stdio.h>
#include<unistd.h>
int main() {
	fprintf(stderr, "hello world");
	for( ; ; ) {						
		sleep(10);
	}
	return 0;
}
```

编译并运行程序,  终端立即输出hello world。可以看出标准错误是不带缓冲的。目的是使出错信息可以尽快显示。

### 例4 缓冲类型与读写的文件类型有关

```
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<sys/wait.h>

int main()
{
	pid_t pid;
	printf("before fork\n");

	if((pid = fork()) < 0) {	// fork失败直接返回-1
		exit(-1);
	} else if(pid > 0) {		// 父进程
		wait(NULL);
	}
	printf("pid = %d, hello\n", getpid());
	return 0;
}
```

编译并执行程序, 得到：

$ **./a.out**						 
before fork
pid = 6033, hello
pid = 6032, hello
$ **./a.out > output.txt**		#将输出重定向到output.txt文件
$ cat output.txt
before fork
pid = 6077, hello
before fork
pid = 6076, hello

发现两次输出的内容不同，将输出重定向到文件时，会多打印一行"before fork"，原因如下：

**如果标准输出连到终端设备，默认是行缓冲的**。"before fork"只输出一次，原因是调用第一个printf后，标准输出缓冲区由换行符冲洗，“before fork”被立即打印。

**如果将标准输出重定向到文件，默认是全缓冲的**。"befork fork"会输出两次，原因是调用第一个printf后数据“before fork”仍旧在缓冲区，然后调用fork函数，将父进程数据空间复制到子进程，**此时该缓冲区也被复制到子进程**。最后当父子进程终止时，各自冲洗其缓冲区的副本。

这个例子说明，缓冲类型与读写的文件类型有关，与具体I/O函数无关。

##  其他说明

以下内容直接摘抄自APUE:

### ISO C要求的缓冲特征

当且仅当标准输入和标准输出不涉及交互设备时，才是全缓冲的。

标准出错不可能是全缓冲的。

但标准中没有规定，当标准输入和输出涉及交互设备时，到底是行缓冲的还是不带缓冲的；也没有规定标准出错是不带缓冲的还是行缓冲的。很多系统默认使用下列类型的缓冲：

标准出错是不带缓冲的。

若涉及终端设备的其他流，则是行缓冲的，否则是全缓冲的。

### 标准I/O库的不足之处

效率不高。当使用fgets, fputs函数，通常需要复制两次数据：一次是内核和标准I/O缓冲之间，一次是标准I/O缓冲和用户程序的行缓冲区之间。



## 参考资料

《UNIX环境高级编程第二版》

https://blog.csdn.net/qq_33366098/article/details/77923722

https://www.yanbinghu.com/2019/12/01/27836.html



