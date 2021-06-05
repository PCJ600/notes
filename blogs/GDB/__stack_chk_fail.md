## __stack_chk_fail 栈溢出问题定位

### 一、问题描述

进程收到`SIGABRT`信号异常退出，异常调用栈显示`__stack_chk_fail`

### 二、原因分析和定位思路

**原因分析：** `__stack_chk_fail`说明**发生了缓冲区溢出，canary被破坏**。这说明代码设置GCC编译选项**fstack-protector**，开启了[**栈保护机制canary**](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/mitigation/canary-zh/)

**定位思路：** 

* 先通过反汇编找到`canary`在栈上的存放地址。
* 用GDB对`canary`的存放地址打数据断点，定位出导致栈破坏的指令，再结合C代码具体分析。

### 三、定位过程

以下给出一个简化案例：一个可执行程序test, 依赖两个.so：`libcomp1.so`, `libcomp2.so`。执行`test`程序后会异常退出，调用栈显示`__stack_chk_fail`

```txt
├── CMakeLists.txt			—— 可执行程序 test, 依赖libcomp1.so, libcomp2.so
├── comp1					
│   ├── CMakeLists.txt		—— libcomp1.so
│   ├── comp1.c
│   ├── lua.h
├── comp2
│   ├── CMakeLists.txt		-- libcomp2.so
│   ├── comp2.c
│   ├── lua.h
├── main.c
```

C代码如下：

```C
// main.c
extern void func1();	// defined in libcomp1.so
void main() {
    func1();
}

// comp1/comp1.c
#include "lua.h"
extern func2(struct lua_Debug *a, char *b, int c); // defined in libcomp2.so
void func1() {
    struct lua_Debug a = {0};
    char *b = 0x12345678;
    int c = 0xFFFFFFFF;
    func2(&a, b, c);
    return;
}

// comp2/comp2.c
#include "lua.h"
void func2(struct lua_Debug *a, char *b, int c) {
    a->i_ci = 1;
    return;
}
```

用GDB调试test程序，出现如下的异常调用栈

```
(gdb) bt
#0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:50
#1  0x00007ffff7e1c535 in __GI_abort () at abort.c:79
#2  0x00007ffff7e73508 in __libc_message (action=<optimized out>, fmt=fmt@entry=0x7ffff7f7e07b "*** %s ***: %s terminated\n")
    at ../sysdeps/posix/libc_fatal.c:181
#3  0x00007ffff7f0480d in __GI___fortify_fail_abort (need_backtrace=need_backtrace@entry=false,
    msg=msg@entry=0x7ffff7f7e059 "stack smashing detected") at fortify_fail.c:28
#4  0x00007ffff7f047c2 in __stack_chk_fail () at stack_chk_fail.c:29
#5  0x00007ffff7fca189 in func1 () at /home/pc/LUA/comp1/comp1.c:13
#6  0x0000555555555143 in main () at /home/pc/LUA/main.c:7
```

第4帧出现`__stack_chk_fail`，这表示程序出现了栈溢出。定位思路如下：

#### 1. 先通过反汇编找到`canary`在栈上的存放地址。

用gdb查看出现`__stack_chk_fail`的前一个函数帧，即第5帧的汇编代码。通过`frame`命令切换函数帧，`disassemble`查看反汇编代码。

```
(gdb) frame 5
#5  0x00007ffff7fca189 in func1 () at /home/pc/LUA/comp1/comp1.c:13
13      }
(gdb) disassemble
Dump of assembler code for function func1:
   0x00007ffff7fca115 <+0>:     push   %rbp
   0x00007ffff7fca116 <+1>:     mov    %rsp,%rbp
   0x00007ffff7fca119 <+4>:     sub    $0x90,%rsp				# 栈上共分配144字节空间: (8字节canary + 120字节lua_Debug + 8字节char * + 4字节int, 4字节对齐 = 144字节)
   0x00007ffff7fca120 <+11>:    mov    %fs:0x28,%rax			# %fs:0x28说明，canary值是通过段寻址方式从内存中读入的
   0x00007ffff7fca129 <+20>:    mov    %rax,-0x8(%rbp)			# 将canary存储到栈中，位于$rbp - 0x8处
   0x00007ffff7fca12d <+24>:    xor    %eax,%eax
   0x00007ffff7fca12f <+26>:    lea    -0x80(%rbp),%rdx			# comp1.c/func1(): struct lua_Debug a = {0};
   0x00007ffff7fca133 <+30>:    mov    $0x0,%eax
   0x00007ffff7fca138 <+35>:    mov    $0xf,%ecx
   0x00007ffff7fca13d <+40>:    mov    %rdx,%rdi
   0x00007ffff7fca140 <+43>:    rep stos %rax,%es:(%rdi)
   0x00007ffff7fca143 <+46>:    movq   $0x12345678,-0x88(%rbp)	# comp1.c/func1(): char *b = 0x12345678;
   0x00007ffff7fca14e <+57>:    movl   $0xffffffff,-0x8c(%rbp)	# comp1.c/func1(): int c = 0xFFFFFFFF;
   0x00007ffff7fca158 <+67>:    mov    -0x8c(%rbp),%edx			# c
   0x00007ffff7fca15e <+73>:    mov    -0x88(%rbp),%rcx			# b
   0x00007ffff7fca165 <+80>:    lea    -0x80(%rbp),%rax			# a
   0x00007ffff7fca169 <+84>:    mov    %rcx,%rsi				
   0x00007ffff7fca16c <+87>:    mov    %rax,%rdi				
   0x00007ffff7fca16f <+90>:    callq  0x7ffff7fca040 <func2@plt>
   0x00007ffff7fca174 <+95>:    nop
   0x00007ffff7fca175 <+96>:    mov    -0x8(%rbp),%rax
   0x00007ffff7fca179 <+100>:   xor    %fs:0x28,%rax			 # 从段寄存器取出canary的值，和栈上$rbp - 0x8处比较，如果发现canary被修改, 调用__stack_chk_fail进行错误处理
   0x00007ffff7fca182 <+109>:   je     0x7ffff7fca189 <func1+116>
   0x00007ffff7fca184 <+111>:   callq  0x7ffff7fca030 <__stack_chk_fail@plt>
=> 0x00007ffff7fca189 <+116>:   leaveq # movq %rbp %rsp, popq %rbp
   0x00007ffff7fca18a <+117>:   retq
End of assembler dump.
```

`0x00007ffff7fca119：sub $0x90,%rsp`：表示栈上分配了**144**个字节，依次存放三个局部变量a, b, c

`0x00007ffff7fca120 <+11>: mov  %fs:0x28,%rax`：其中指令参数`$fs:0x28`说明`canary`通过段寻址从内存中读入。canary值存放在一个特殊的段中，标志为**只读**，这样攻击者就无法覆盖canary的值。

`0x00007ffff7fca129 <+20>: mov %rax,-0x8(%rbp)`：说明canary在栈上存放地址是`$rbp - 0x8`。

根据**x86_64过程调用的函数参数传递规则**，可确定栈上所有局部变量和`canary`是如何存放的。`func1`的栈组织如下：

![image-20201216230603660](C:\Users\pc\Desktop\image-20201216230603660.png)

#### 2. 用GDB对`canary`的存放地址打数据断点，定位出导致栈破坏的指令

对`func1`打断点，重新执行程序，接着用`watch`命令，对`canary`的存放地址打数据断点，操作如下：

```txt
(gdb) disassemble
Dump of assembler code for function func1:
   0x00007ffff7fca115 <+0>:     push   %rbp
   0x00007ffff7fca116 <+1>:     mov    %rsp,%rbp
   0x00007ffff7fca119 <+4>:     sub    $0x90,%rsp
   0x00007ffff7fca120 <+11>:    mov    %fs:0x28,%rax
   0x00007ffff7fca129 <+20>:    mov    %rax,-0x8(%rbp)
=> 0x00007ffff7fca12d <+24>:    xor    %eax,%eax	
   0x00007ffff7fca12f <+26>:    lea    -0x80(%rbp),%rdx
   ......
End of assembler dump.
(gdb) p $rbp - 0x8
$8 = (void *) 0x7fffffffe4a8
(gdb) watch *0x7fffffffe4a8							# 此处对canary在栈中存放的地址打数据断点！
Hardware watchpoint 2: *0x7fffffffe4a8
(gdb) c
Continuing.

Hardware watchpoint 2: *0x7fffffffe4a8
Old value = 1303192064
New value = 1

(gdb) disassemble
Dump of assembler code for function func2:
   0x00007ffff7fc50f5 <+0>:     push   %rbp
   0x00007ffff7fc50f6 <+1>:     mov    %rsp,%rbp
   0x00007ffff7fc50f9 <+4>:     mov    %rdi,-0x8(%rbp)     # 第一个入参，lua_Debug *a
   0x00007ffff7fc50fd <+8>:     mov    %rsi,-0x10(%rbp)
   0x00007ffff7fc5101 <+12>:    mov    %edx,-0x14(%rbp)
   0x00007ffff7fc5104 <+15>:    mov    -0x8(%rbp),%rax	   # 第一个入参，lua_Debug *a
   0x00007ffff7fc5108 <+19>:    movq   $0x1,0x78(%rax)     # comp2.c/func2(): a->i_ci = 1; 这句导致canary被破坏
=> 0x00007ffff7fc5110 <+27>:    nop 	# 箭头指向的指令表示下一步即将执行的指令，也就是说是上一条指令movq $0x1,0x78(%rax)触发的数据断点!
   0x00007ffff7fc5111 <+28>:    pop    %rbp
   0x00007ffff7fc5112 <+29>:    retq
End of assembler dump.
```

数据断点被触发，定位出`0x00007ffff7fc5108: $0x1,0x78(%rax)`导致`canary`破坏，结合汇编上下文和C码确定，这条指令关联的C语句是`a->i_ci = 1;` 

#### 为什么对lua_Debug结构体的成员赋值会导致栈溢出？

由`$0x1,0x78(%rax)`可以确定`i_ci`成员距离结构体首地址的偏移为0x78，即120字节。事实上，在`func2`的函数帧中，`lua_Debug`结构体大小为128字节，而`func1`中的`lua_Debug`结构体只有120字节。

结合`func1`的栈帧发现栈溢出的直接原因：`a->i_ci = 1;`这句执行后，恰好导致canary的值被改写为1。

通过`gdb`也能发现两个.so中的结构体大小不一致问题：

```
(gdb) f 1			
#1  0x00007ffff7fca174 in func1 () at /home/pc/LUA/comp1/comp1.c:11
11          func2(&a, b, c);
(gdb) p sizeof(struct lua_Debug)
$24 = 120							# func1中结构体大小为120
(gdb) f 0
#0  func2 (a=0x7fffffffe430, b=0x12345678 <error: Cannot access memory at address 0x12345678>, c=-1) at /home/pc/LUA/comp2/comp2.c:5
5           return;
(gdb) p sizeof(struct lua_Debug)
$27 = 128							# func2中结构体大小为128
```

`lua_Debug`结构体在`lua.h`定义的，这个错误原因是**由于两个.so编译使用的lua.h头文件不一致，导致栈溢出问题。**以下分别给出两个.so的`lua.h`：

![捕获](C:\Users\pc\Desktop\捕获.PNG)

根据64位**结构体对齐规则**，左边结构体大小为120，右边结构体大小为128，多出的这8个字节恰好覆盖了`canary`，导致栈溢出。

### 总结

* 公司的C代码中，多个.so会依赖相同的开源头文件，如果不能保证每个.so各自依赖的头文件版本一致，就可能出现上述的栈溢出问题。

* 通过设置GCC的编译选项"fstack-protector"开启栈保护机制，便于问题定位。

### 参考资料

《深入理解计算机系统 原书第3版》 3.10.4.2 栈破坏检测



