## CSAPP 二进制炸弹实验

### 实验简介

二进制炸弹是一个作为目标代码提供的程序。运行时提示用户输入6个不同的字符串，如其中一个字符串不正确，炸弹会引爆并打印一条错误信息。需要通过反汇编确定输入的6个字符串，从而拆除炸弹。

### 知识点

* 汇编语言基础
* GDB和OBJDUMP工具的使用

### 实验环境

Centos7 x86_64

### 获取二进制炸弹

首先从CSAPP官网获取二进制炸弹`bomb.tar`:  [http://csapp.cs.cmu.edu/3e/labs.html](http://csapp.cs.cmu.edu/3e/labs.html)

在linux下执行`tar xvf bomb.tar`，得到二进制炸弹的文件，文件列表如下：

```
|-- bomb		# 二进制炸弹，x86-64位
|-- bomb.c		# 主程序，逻辑是接受用户输入的6个字符串，并判断每个字符串是否正确。如果正确，调用phase_defused进入下一关，否则调用explode_bomb引爆炸弹
```

### 第一关

#### 1. 首先理解main函数执行过程

反汇编炸弹，使用`odjump -d bomb > bomb.txt`命令，内容如下：

```
0000000000400da0 <main>:
  400da0:   53                      push   %rbx
  ......
  400e19:   e8 84 05 00 00          callq  4013a2 <initialize_bomb>
  400e1e:   bf 38 23 40 00          mov    $0x402338,%edi
  400e23:   e8 e8 fc ff ff          callq  400b10 <puts@plt>
  400e28:   bf 78 23 40 00          mov    $0x402378,%edi
  400e2d:   e8 de fc ff ff          callq  400b10 <puts@plt>
  400e32:   e8 67 06 00 00          callq  40149e <read_line> # 接受用户输入的字符串，保存到%rax寄存器
  400e37:   48 89 c7                mov    %rax,%rdi		  # 用户输入的字符串保存在$rdi寄存器，并作为phase_1函数的第一个入参传递
  400e3a:   e8 a1 00 00 00          callq  400ee0 <phase_1>	  # 调用phase_1(), 执行第一关代码
  400e3f:   e8 80 07 00 00          callq  4015c4 <phase_defused>
```

查看`0x400e32`处的代码： 主程序调用`read_line`接收用户输入的字符串，保存到`$rax`，并且将该字符串作为函数入参传递给`phase_1`，执行第一阶段的代码。

<font color = 'red'>**这里需要理解x86-64的过程调用规则：**</font>（关于过程调用，可参考《CSAPP原书第三版》3.7小节 —— 过程）

* 函数调用中，**利用`%rax`寄存器保存返回值。**
* 关于参数传递 , **如果函数参数不超过6个，会依次通过`%rdi`,` %rsi`, `%rdx`, `%rcx`, `%r8`, `%r9`传递； 如超过6个参数，超出的参数利用栈传递。**
* `%rbx, %rbp, %r12~%15`被划分为**被调用者保存寄存器**；其余的寄存器，除了栈指针`%rsp`，都被划分为**调用者保存寄存器**。

#### 2. 理解phase_1执行过程

查看`phase_1`的代码，如下：

```
0000000000400ee0 <phase_1>:
  400ee0:   48 83 ec 08             sub    $0x8,%rsp
  400ee4:   be 00 24 40 00          mov    $0x402400,%esi 			  # $esi作为strings_not_equal函数的第二个参数传递
  400ee9:   e8 4a 04 00 00          callq  401338 <strings_not_equal> # 调用strings_not_equal函数，比较用户输入的字符串($rdi)和$esi处的字符串是否相等
  400eee:   85 c0                   test   %eax,%eax 				  # 判断%eax的值是否为0
  400ef0:   74 05                   je     400ef7 <phase_1+0x17> 	  # 如%eax等于0，跳转到400ef7,正常退出
  400ef2:   e8 43 05 00 00          callq  40143a <explode_bomb> 	  # 如%eax不等于0，炸弹爆炸
  400ef7:   48 83 c4 08             add    $0x8,%rsp
  400efb:   c3                      retq
```

`phase_1`执行过程如下：

* 首先通过`callq`指令调用`strings_not_equal`函数。根据过程调用的规则，可以确定这个函数接受2个参数。第一个参数是`%rdi`, 上面分析过，是我们拆弹时输入的字符串；第二个参数是`$esi`，值为`0x402400`。顾名思义，`strings_not_equal`函数用来判断两个字符串是否相等。
* 接着用`test`指令判断函数的返回值是否为0。如果为0进行跳转并返回，调用`phase_defused`拆除炸弹，否则通过callq指令调用`explode_bomb`，引爆炸弹。

因此，拆弹的关键在于，确认`0x402400`地址处的字符串是什么。只需要在`0x400ee9`处打个gdb断点就可以了。

#### 3. gdb调试炸弹

执行`gdb bomb`, 设置断点`b *0x400ee9`,  执行`run`。 此时程序会要求用户输入字符串，先随便输一个字符串使程序运行到我们设置的断点 `0x400ee9`处，如下：

```
# gdb bomb		
(gdb) b *0x400ee9							# 设置断点， b相当于break
Breakpoint 1 at 0x400ee9
(gdb) r										# 运行程序， r相当于run
Starting program: /home/pc/CSAPP/bomb/bomb
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
123											# 随便输入一个字符串，让程序运行到断点0x400ee9
Breakpoint 1, 0x0000000000400ee9 in phase_1 ()
(gdb) p (char *)$rdi						# $rdi保存用户输入字符串，为123，与x/s $rdi指令等价
$2 = 0x603780 <input_strings> "123"
(gdb) x/s $esi								# 查看esi处字符串，这就是第一关的答案
$1 = 0x402400 "Border relations with Canada have never been better."
```

查看`%esi`处字符串, 即第一关的答案，如下：

`Border relations with Canada have never been better.`

用`quit`指令退出gdb, 新建一个文件`answer`，将答案写入该文件。重新执行`bomb`程序，指定`answer`文件名作为参数，如下：

```
# ./bomb answer
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Phase 1 defused. How about the next one?
```

显示`Phase 1 defused`, 表示第一关已成功通过。

#### 几个调试汇编代码相关的GDB指令

使用`disassemble`指令查看汇编代码，箭头表示下一步即将执行的汇编指令。

`si`指令用于单步执行汇编代码，相当于`stepi`

`ni`指令以函数调用为单位进行单步执行， 相当于`nexti`

```
(gdb) disassemble
Dump of assembler code for function phase_1:
   0x0000000000400ee0 <+0>:     sub    $0x8,%rsp
   0x0000000000400ee4 <+4>:     mov    $0x402400,%esi
=> 0x0000000000400ee9 <+9>:     callq  0x401338 <strings_not_equal>
   0x0000000000400eee <+14>:    test   %eax,%eax
   0x0000000000400ef0 <+16>:    je     0x400ef7 <phase_1+23>
   0x0000000000400ef2 <+18>:    callq  0x40143a <explode_bomb>
   0x0000000000400ef7 <+23>:    add    $0x8,%rsp
   0x0000000000400efb <+27>:    retq
End of assembler dump.
```

更多用法参考`gdb`手册或《CSAPP 原书第3版》3.10.2小节 —— 使用GDB调试器

#### 拆弹小技巧

* 查看`bomb`符号表或者直接查看汇编代码，会发现有个`explode_bomb`符号，该函数用来引爆炸弹。

* 调试时可以对该函数设置断点，在拆弹失败时暂停运行，不让其爆炸，便于调试。可使用 `b explode_bomb`指令设置断点

### 第二关

先贴出第二关`phase_2`的部分汇编代码：

```
0000000000400efc <phase_2>:
  400efc:   55                      push   %rbp
  400efd:   53                      push   %rbx
  400efe:   48 83 ec 28             sub    $0x28,%rsp
  400f02:   48 89 e6                mov    %rsp,%rsi	# %rsi作为read_six_numbers的第二个入参传递, $rsp既是入参也是出参；第一个入参为$rdi, 即用户输入的字符串
  400f05:   e8 52 05 00 00          callq  40145c <read_six_numbers> # 读六个数字
  ......
```

`phase_2`程序先调用`read_six_numbers`。该函数接受两个参数，第一个参数为`$rdi`, 即我们输入的字符串；第二个参数为`$rsp`, `$rsp`既是入参也是出参，用于保存`read_six_numbers`函数解析`$rdi`后得到的6个整数。想得到这个结论，需要分析`read_six_numbers`代码，如下：

```
(gdb) disassemble read_six_numbers
Dump of assembler code for function read_six_numbers:
   0x000000000040145c <+0>:     sub    $0x18,%rsp
   0x0000000000401460 <+4>:     mov    %rsi,%rdx		# sscanf函数的第1个可变参数，第3个参数，通过%rdx传递
   0x0000000000401463 <+7>:     lea    0x4(%rsi),%rcx	# sscanf函数的第2个可变参数，第4个参数, 通过%rcx传递
   0x0000000000401467 <+11>:    lea    0x14(%rsi),%rax	# sscanf函数的第6个可变参数，第8个参数，通过栈传递
   0x000000000040146b <+15>:    mov    %rax,0x8(%rsp)
   0x0000000000401470 <+20>:    lea    0x10(%rsi),%rax  # sscanf函数的第5个可变参数，第7个参数，通过栈传递
   0x0000000000401474 <+24>:    mov    %rax,(%rsp)
   0x0000000000401478 <+28>:    lea    0xc(%rsi),%r9	# sscanf函数的第4个可变参数，第6个参数, 通过%r9传递
   0x000000000040147c <+32>:    lea    0x8(%rsi),%r8	# sscanf函数的第3个可变参数，第5个参数, 通过%r8传递
   0x0000000000401480 <+36>:    mov    $0x4025c3,%esi	# sscanf函数的第2个参数，通过%esi传递，0x4025c3地址的格式化字符串为"%d %d %d %d %d %d"，表示输入字符串应该为6个整数
   0x0000000000401485 <+41>:    mov    $0x0,%eax
   0x000000000040148a <+46>:    callq  0x400bf0 <__isoc99_sscanf@plt>
   0x000000000040148f <+51>:    cmp    $0x5,%eax		# sscanf读入的可变参数需大于5个，否则爆炸
   0x0000000000401492 <+54>:    jg     0x401499 <read_six_numbers+61>
   0x0000000000401494 <+56>:    callq  0x40143a <explode_bomb>
   0x0000000000401499 <+61>:    add    $0x18,%rsp
   0x000000000040149d <+65>:    retq
```

查看`0x400f02`, `0x401463 ~ 0x40147c`处的代码，我们可以把调用者`phase_2`中的`$rsp`看作一个一维数组的首地址，该数组的长度为6，内容依次为`$rsp`, `$rsp + 4`, `$rsp + 8`, `$rsp + 12`, `$rsp + 16`, `$rsp + 20`，用于 保存`sscanf`函数执行后生成的6个整数。

#### 炸弹用sscanf读取并解析用户输入字符串

注意到`0x40148a`处调用`sscanf`函数，作用是从用户输入的字符串`%rdi`中解析出6个整数，保存到调用者`phase_2`中的`$rsp`

C语言中`sscanf`函数原型如下：

```C
int sscanf(const char *str, const char *format, ...);
```

对照汇编代码， `$rdi`相当于`sscanf`的参数`str`；查看`0x401480`代码，`$esi`相当于`sscanf`的参数`format`， 用gdb查看`0x4025c3`处的字符串，为`"%d %d %d %d %d %d"`， 说明`sscanf`中的可变参数个数为6，且都是指向`int`类型的地址。

```
(gdb) x/s 0x4025c3
0x4025c3:       "%d %d %d %d %d %d"
```

可以看出`sscanf`函数实际上接收8个入参。`read_six_numbers`利用`%rdi`,  ` %rsi`，`%rdx`, `%rcx`, `%r8`, `%r9`分别传递用户输入字符串、格式化字符串、6个整数中的前4个。而第5、6个整数超出了六个参数，需通过栈传递。

**到此，确定了第二关需要输入6个整数, 且这6个整数保存在调用者`phase_2`的`%rsp`中**，用gdb验证这个结论：

```
gdb bomb
(gdb) set args answer 	# 第二行可随便输入6个数，例如1 2 3 4 5 6 
(gdb) b *0x400f0a
(gdb) c
Breakpoint 3, 0x0000000000400f0a in phase_2 ()
(gdb) x/6x $rsp			# 查看$rsp, 和输入字符串中6个数一致
0x7fffffffe420: 0x00000001      0x00000002      0x00000003      0x00000004
0x7fffffffe430: 0x00000005      0x00000006
```

再看下`phase_2`完整代码：

```
0000000000400efc <phase_2>:
  400efc:   55                      push   %rbp
  400efd:   53                      push   %rbx
  400efe:   48 83 ec 28             sub    $0x28,%rsp
  400f02:   48 89 e6                mov    %rsp,%rsi 			# %rsi作为read_six_numbers的第二个入参传递, $rsp既是入参也是出参；第一个入参为$rdi, 即用户输入的字符串
  400f05:   e8 52 05 00 00          callq  40145c <read_six_numbers> # 读六个数字
  400f0a:   83 3c 24 01             cmpl   $0x1,(%rsp) 			# %rsp为调用者保存寄存器，过程调用前后值不变，因此保存的是read_six_numbers输出的6个数，(%rsp)保存的是第一个整数
  400f0e:   74 20                   je     400f30 <phase_2+0x34>
  400f10:   e8 25 05 00 00          callq  40143a <explode_bomb>
  400f15:   eb 19                   jmp    400f30 <phase_2+0x34># 将6个整数看作一个数组
  400f17:   8b 43 fc                mov    -0x4(%rbx),%eax		# 将数组前一个数保存到%eax
  400f1a:   01 c0                   add    %eax,%eax			# 将%eax乘以2
  400f1c:   39 03                   cmp    %eax,(%rbx)			# 判断当前整数是否为前一个数的两倍， 不等则爆炸，相等跳转到400f25，
  400f1e:   74 05                   je     400f25 <phase_2+0x29>
  400f20:   e8 15 05 00 00          callq  40143a <explode_bomb>
  400f25:   48 83 c3 04             add    $0x4,%rbx			# 每次循环，将rbx值加4，即指向数组的下一个元素
  400f29:   48 39 eb                cmp    %rbp,%rbx			# rbp指向数组结尾，标识循环是否结束
  400f2c:   75 e9                   jne    400f17 <phase_2+0x1b>
  400f2e:   eb 0c                   jmp    400f3c <phase_2+0x40>
  400f30:   48 8d 5c 24 04          lea    0x4(%rsp),%rbx 		# 最初rbx指向第二个数，
  400f35:   48 8d 6c 24 18          lea    0x18(%rsp),%rbp  	# %rbp = $rsp + 24
  400f3a:   eb db                   jmp    400f17 <phase_2+0x1b>
  400f3c:   48 83 c4 28             add    $0x28,%rsp
  400f40:   5b                      pop    %rbx
  400f41:   5d                      pop    %rbp
  400f42:   c3                      retq
```

由`0x400f0a: cmpl  $0x1,(%rsp)`可知，第一个整数一定是1，然后跳转到`0x400f30`进入循环。

将这6个数看作一个数组， 由`0x400f30`处代码，`$rbx`可看作这个数组的下标，初始值为1，指向第2个整数；由`0x400f35`处代码，`$rbp`标识着数组的结尾，用于判断循环是否退出。

由`0x400f17` ~ `0x400f1c`代码可知，每次循环判断数组当前元素是否为前一个元素的两倍，不等则爆炸。因此答案为`1 2 4 8 16 32`， 唯一解。

### 第三关

`phase_3`的代码如下：

```
0000000000400f43 <phase_3>:
  400f43:   48 83 ec 18             sub    $0x18,%rsp
  400f47:   48 8d 4c 24 0c          lea    0xc(%rsp),%rcx		# 第二个整数位于$rsp + 12
  400f4c:   48 8d 54 24 08          lea    0x8(%rsp),%rdx		# 第一个整数位于%rsp + 8
  400f51:   be cf 25 40 00          mov    $0x4025cf,%esi		# 查看0x4025cf地址处内存，为"%d %d"，表示接受两个整数作为输入
  400f56:   b8 00 00 00 00          mov    $0x0,%eax
  400f5b:   e8 90 fc ff ff          callq  400bf0 <__isoc99_sscanf@plt>
  400f60:   83 f8 01                cmp    $0x1,%eax			# sscanf返回值需大于1，否则爆炸。说明可变参数个数为2
  400f63:   7f 05                   jg     400f6a <phase_3+0x27>
  400f65:   e8 d0 04 00 00          callq  40143a <explode_bomb>
  400f6a:   83 7c 24 08 07          cmpl   $0x7,0x8(%rsp)		# 第一个数必须小于7，否则爆炸
  400f6f:   77 3c                   ja     400fad <phase_3+0x6a># 比较结果大于0则跳转
  400f71:   8b 44 24 08             mov    0x8(%rsp),%eax
  400f75:   ff 24 c5 70 24 40 00    jmpq   *0x402470(,%rax,8)	# 跳转表结构，对应C语言中的switch语句
  400f7c:   b8 cf 00 00 00          mov    $0xcf,%eax			# %rax = 0，跳转到400f7c
  400f81:   eb 3b                   jmp    400fbe <phase_3+0x7b>
  400f83:   b8 c3 02 00 00          mov    $0x2c3,%eax
  400f88:   eb 34                   jmp    400fbe <phase_3+0x7b>
  400f8a:   b8 00 01 00 00          mov    $0x100,%eax
  400f8f:   eb 2d                   jmp    400fbe <phase_3+0x7b>
  400f91:   b8 85 01 00 00          mov    $0x185,%eax
  400f96:   eb 26                   jmp    400fbe <phase_3+0x7b>
  400f98:   b8 ce 00 00 00          mov    $0xce,%eax
  400f9d:   eb 1f                   jmp    400fbe <phase_3+0x7b>
  400f9f:   b8 aa 02 00 00          mov    $0x2aa,%eax
  400fa4:   eb 18                   jmp    400fbe <phase_3+0x7b>
  400fa6:   b8 47 01 00 00          mov    $0x147,%eax
  400fab:   eb 11                   jmp    400fbe <phase_3+0x7b>
  400fad:   e8 88 04 00 00          callq  40143a <explode_bomb>
  400fb2:   b8 00 00 00 00          mov    $0x0,%eax
  400fb7:   eb 05                   jmp    400fbe <phase_3+0x7b>
  400fb9:   b8 37 01 00 00          mov    $0x137,%eax
  400fbe:   3b 44 24 0c             cmp    0xc(%rsp),%eax		# 判断%eax值与第二个参数是否相等，不等则爆炸
  400fc2:   74 05                   je     400fc9 <phase_3+0x86>
  400fc4:   e8 71 04 00 00          callq  40143a <explode_bomb>
  400fc9:   48 83 c4 18             add    $0x18,%rsp
  400fcd:   c3                      retq
```

与第二关类似，查看`0x400f51`处代码`mov $0x4025cf $esi` , 用gdb打印`0x4025cf`处内存，如下：

```
(gdb) x/s 0x4025cf
0x4025cf:       "%d %d"
```

内容为`"%d %d"`，表示这一关需要输入两个整数。

由`400f47 ~ 400f4c`代码可知，第一个整数位于`$rsp + 8`地址，第二个整数位于`$rsp + 12`地址

#### 确认这两个整数应满足的条件

观察`0x400f6a`处的`cmp`指令。注意比较顺序，是计算`*(%rsp + 8) - 7` 的值，再判断这个值是否大于`0`

```
400f6a:   83 7c 24 08 07          cmpl   $0x7,0x8(%rsp)		# 第一个数必须小于7，否则爆炸
400f6f:   77 3c                   ja     400fad				# 引爆炸弹
```

以上两句汇编等同于 `if (*rsp+8) > 7, 跳转到0x400fad`， 因此第一个数必须不大于7。

`0x400f75`处`jmpq *0x402470(,%rax,8)`是一个间接跳转指令, 可以看出这段代码是典型的switch语句，跳转表就存在于`0x402470`。`%rax`取值为[0, 7]，代表switch语句中8条不同的case。 打印这张跳转表：

```
(gdb) x/8g 0x402470
0x402470:       0x0000000000400f7c      0x0000000000400fb9
0x402480:       0x0000000000400f83      0x0000000000400f8a
0x402490:       0x0000000000400f91      0x0000000000400f98
0x4024a0:       0x0000000000400f9f      0x0000000000400fa6
```

举例，第一个整数取0时，会跳转到`0x400f7c`, 将`0xcf`赋给`%rax`，`0x400fbe`处再判断`$rax`和第二个整数是否相等。因此`0 207`为满足条件的一组解。依次类推，一共得到8组解，答案不唯一，任选一种即可：

```
0 207
1 311
2 707
3 256
4 389
5 206
6 682
7 327
```

switch语句和跳转表内容可参考 《CSAPP 原书第3版》 3.6.8小节 —— switch语句。

### 第四关

`phase_4`的代码如下：

```
000000000040100c <phase_4>:
  40100c:   48 83 ec 18             sub    $0x18,%rsp
  401010:   48 8d 4c 24 0c          lea    0xc(%rsp),%rcx		# 第二个整数，用$rcx保存
  401015:   48 8d 54 24 08          lea    0x8(%rsp),%rdx		# 第一个整数，用%rdx保存
  40101a:   be cf 25 40 00          mov    $0x4025cf,%esi		# %rsi处字符串: "%d %d"
  40101f:   b8 00 00 00 00          mov    $0x0,%eax
  401024:   e8 c7 fb ff ff          callq  400bf0 <__isoc99_sscanf@plt>
  401029:   83 f8 02                cmp    $0x2,%eax			
  40102c:   75 07                   jne    401035 <phase_4+0x29>
  40102e:   83 7c 24 08 0e          cmpl   $0xe,0x8(%rsp)		# 将第一个整数和14比较
  401033:   76 05                   jbe    40103a <phase_4+0x2e># 如果不大于14跳转，否则引爆炸弹
  401035:   e8 00 04 00 00          callq  40143a <explode_bomb>
  40103a:   ba 0e 00 00 00          mov    $0xe,%edx 			# func4函数的第一个入参，初值为14
  40103f:   be 00 00 00 00          mov    $0x0,%esi 			# func4函数的第二个入参，初值为0		
  401044:   8b 7c 24 08             mov    0x8(%rsp),%edi 		# func4函数的第三个入参，初值为输入的第一个整数
  401048:   e8 81 ff ff ff          callq  400fce <func4>		# 调用func4, func4为递归函数
  40104d:   85 c0                   test   %eax,%eax			# func4函数必须返回0，否则爆炸
  40104f:   75 07                   jne    401058 <phase_4+0x4c>
  401051:   83 7c 24 0c 00          cmpl   $0x0,0xc(%rsp)		# 第二个整数必须为0， 否则爆炸
  401056:   74 05                   je     40105d <phase_4+0x51>
  401058:   e8 dd 03 00 00          callq  40143a <explode_bomb>
  40105d:   48 83 c4 18             add    $0x18,%rsp
  401061:   c3                      retq
```

同样的，先确认输入字符串的格式，查看`0x4025cf`处的格式化字符串

```
(gdb) x/s 0x4025cf
0x4025cf:       "%d %d"
```

内容为`"%d %d"`， 说明需要输入两个整数。第一个整数位于`%rsp + 8`， 第二个整数位于`%rsp + 12`

由`0x40102e ~ 0x401033`代码可知，第一个整数必须不大于14， 否则引爆炸弹。

注意到`0x401048`处调用`func4`函数，并判断该函数返回值是否为0，不等于0则引爆炸弹。

由`0x40103a ~ 0x401044`三条语句可知，`func4`函数接受三个入参，且三个参数的初始值从左到右分别为输入的第一个整数,`14`, `0`。下面查看`func4`代码:

```
# int func4(int x, int y, int z);
# x in %edi, y in $esi, z in $edx, ret in $eax
0000000000400fce <func4>:
  400fce:   48 83 ec 08             sub    $0x8,%rsp
  400fd2:   89 d0                   mov    %edx,%eax 			# ret = z				
  400fd4:   29 f0                   sub    %esi,%eax			# ret -= y
  400fd6:   89 c1                   mov    %eax,%ecx			# ecx = ret
  400fd8:   c1 e9 1f                shr    $0x1f,%ecx			# ecx = (ecx >> 31) & 0x1
  400fdb:   01 c8                   add    %ecx,%eax			# ret += ecx
  400fdd:   d1 f8                   sar    %eax					# ret >>= 1
  400fdf:   8d 0c 30                lea    (%rax,%rsi,1),%ecx	# ecx = ret + y
  400fe2:   39 f9                   cmp    %edi,%ecx			
  400fe4:   7e 0c                   jle    400ff2 <func4+0x24>	# if ecx <= x, jump to 0x400ff2
  400fe6:   8d 51 ff                lea    -0x1(%rcx),%edx		# z = rcx - 1
  400fe9:   e8 e0 ff ff ff          callq  400fce <func4>
  400fee:   01 c0                   add    %eax,%eax			# ret *= 2
  400ff0:   eb 15                   jmp    401007 <func4+0x39>	
  400ff2:   b8 00 00 00 00          mov    $0x0,%eax			# ret = 0
  400ff7:   39 f9                   cmp    %edi,%ecx			
  400ff9:   7d 0c                   jge    401007 <func4+0x39>	# if ecx >= x, jump to 0x401007
  400ffb:   8d 71 01                lea    0x1(%rcx),%esi		# y = ecx + 1
  400ffe:   e8 cb ff ff ff          callq  400fce <func4>		# ret = func(x, y, z)
  401003:   8d 44 00 01             lea    0x1(%rax,%rax,1),%eax# ret = 2 * ret + 1
  401007:   48 83 c4 08             add    $0x8,%rsp
  40100b:   c3                      retq
```

由`400fe9`和`400ffe`处的`callq 400fce <func4>`指令可知发生了递归调用。我们可以将`func4`的汇编代码逐句翻译成C语言，将第一个整数从0取到14依次调用`func4`函数，看哪些取值能成功返回`0`。这里需要了解`add`,` sub`,` sar`,` shr`,`lea`,` jle`等指令的用法以及注意操作数的顺序。

`func4`的递归过程，可以转换为如下的C语言函数：

```C
// x in %edi, y in $esi, z in $edx, ret in %eax
int func4(int x, int y, int z) {
    int ecx;
    int ret = z - y;

    if(z < y) {
        ret += 1;
    }
    ret >>= 1;
    ecx = ret + y;

    if(ecx == x) {
        return 0;
    } else if (ecx <= x) {
        return 2 * func4(x, ecx + 1, z) + 1;
    } else {
        return 2 * func4(x, y, ecx - 1);
    }
    return ret;
}
int main() {
    for(int i = 0; i <= 14; ++i)
        if(func4(i, 0, 14) == 0)
            printf("answer: %d\n", i);	// 打印出第一个整数的所有取值
    return 0;
}
```

编译执行C程序，发现第一个整数可以是`0`， `1`， `3`， `7`

由`phase_4`的`0x401051 ~ 0x401056`代码可知，第二个整数必须为`0`，否则引爆炸弹。

因此，一共得到四组解，答案不唯一，任选一种即可：

```
0 0
1 0
3 0
7 0
```

### 第五关

`phase_5`的代码如下, 根据`0x40107f`处的`cmp $0x6, %eax`指令，可确定这关需要输入长度为6的字符串

```
0000000000401062 <phase_5>:
  401062:   53                      push   %rbx
  401063:   48 83 ec 20             sub    $0x20,%rsp
  401067:   48 89 fb                mov    %rdi,%rbx			# rbx保存输入字符串
  40106a:   64 48 8b 04 25 28 00    mov    %fs:0x28,%rax
  401071:   00 00
  401073:   48 89 44 24 18          mov    %rax,0x18(%rsp)
  401078:   31 c0                   xor    %eax,%eax
  40107a:   e8 9c 02 00 00          callq  40131b <string_length>
  40107f:   83 f8 06                cmp    $0x6,%eax			# 输入字符串的长度必须为6,否则爆炸
  401082:   74 4e                   je     4010d2 <phase_5+0x70>
  401084:   e8 b1 03 00 00          callq  40143a <explode_bomb>
  401089:   eb 47                   jmp    4010d2 <phase_5+0x70>
  40108b:   0f b6 0c 03             movzbl (%rbx,%rax,1),%ecx
  40108f:   88 0c 24                mov    %cl,(%rsp)
  401092:   48 8b 14 24             mov    (%rsp),%rdx
  401096:   83 e2 0f                and    $0xf,%edx 			# 将当前字符与上0xf，结果保存在%edx
  401099:   0f b6 92 b0 24 40 00    movzbl 0x4024b0(%rdx),%edx  # 将(0x4024b0+%edx)处的字符保存在%rdx
  4010a0:   88 54 04 10             mov    %dl,0x10(%rsp,%rax,1)
  4010a4:   48 83 c0 01             add    $0x1,%rax			# 每次循环%rax加1
  4010a8:   48 83 f8 06             cmp    $0x6,%rax			# 用%rax循环计数，循环6次
  4010ac:   75 dd                   jne    40108b <phase_5+0x29>
  4010ae:   c6 44 24 16 00          movb   $0x0,0x16(%rsp)
  4010b3:   be 5e 24 40 00          mov    $0x40245e,%esi		# 0x40245e处字符串为flyers
  4010b8:   48 8d 7c 24 10          lea    0x10(%rsp),%rdi		# 这里需要构造输入串，使得(%rsp+0x10)处的串等于"flyers"
  4010bd:   e8 76 02 00 00          callq  401338 <strings_not_equal>
  4010c2:   85 c0                   test   %eax,%eax
  4010c4:   74 13                   je     4010d9 <phase_5+0x77>
  4010c6:   e8 6f 03 00 00          callq  40143a <explode_bomb>
  4010cb:   0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)
  4010d0:   eb 07                   jmp    4010d9 <phase_5+0x77>
  4010d2:   b8 00 00 00 00          mov    $0x0,%eax			# 循环开始，eax初值为0
  4010d7:   eb b2                   jmp    40108b <phase_5+0x29>
  4010d9:   48 8b 44 24 18          mov    0x18(%rsp),%rax
  4010de:   64 48 33 04 25 28 00    xor    %fs:0x28,%rax
  4010e5:   00 00
  4010e7:   74 05                   je     4010ee <phase_5+0x8c>
  4010e9:   e8 42 fa ff ff          callq  400b30 <__stack_chk_fail@plt>
  4010ee:   48 83 c4 20             add    $0x20,%rsp
  4010f2:   5b                      pop    %rbx
  4010f3:   c3                      retq
```

根据`jmp 40108b <phase_5+0x29>`，看出这段代码是循环。`%eax`初始为0， 每次循环将%eax加1，再和`6`进行比较(`0x4010a8`)。循环结束后调用`strings_not_equal`,将`0x40245e`处的字符串和`$rsp + 0x10`比较，两个字符串必须相等，否则爆炸。先查看`0x40245e`处的内容：

```
(gdb) x/s 0x40245e
0x40245e:       "flyers"
```

内容为`flyers`， 再看看`$rsp + 0x10`处的字符串是怎么来的：

从`0x401067: mov %rdi,%rbx`看出，我们输入的字符串位于`%rbx`。这里依次将`%rbx`的每一个字符先与`0xf`做与运算，然后加上`0x4024b0`得到新的地址`x`, 最后取地址`x`处的字符作为输出；循环结束后，输出一个长度为6的字符串，将以上逻辑改写为如下C代码：

```C
int phase_5(char str[6]) {	// str表示用户输入的字符串
	char res[6];
	for(int i = 0; i < 6; ++i) {
		res[i] = *(char *)((str[i] & 0xf) + 0x4024b0);
	}
	return !strcmp(res[i], "flyers");
}
```

因此，我们只需确定`0x4024b0`处的内容，然后对照ASCII码表，即可得到答案。

先查看`0x4024b0`， 只需查看前16个字符

```
(gdb) x/16c 0x4024b0
0x4024b0 <array.3449>  : 109 'm' 97 'a'  100 'd' 117 'u' 105 'i' 101 'e' 114 'r' 115 's'
0x4024b8 <array.3449+8>: 110 'n' 102 'f' 111 'o' 116 't' 118 'v' 98 'b'  121 'y' 108 'l'
```

发现`flyers`中的每个字符都可以找到。根据偏移确定输入的每个字符的ASCII码最低一个字节依次为`0x9, 0xF, 0xE, 0x5, 0x6, 0x7`, 答案不唯一。对照ASCII码表 http://ascii.911cha.com/， 我们找到一组解：`IONUVW`

| 二进制    | 十进制 | 十六进制 | 图形 |
| --------- | ------ | -------- | ---- |
| 0100 1001 | 73     | 49       | I    |
| 0100 1111 | 79     | 4F       | O    |
| 0100 1110 | 78     | 4E       | N    |
| 0101 0101 | 85     | 55       | U    |
| 0101 0110 | 86     | 56       | V    |
| 0101 0111 | 87     | 57       | W    |

### 第六关

`phase_6`的代码如下，非常的长

```
00000000004010f4 <phase_6>:
  4010f4:   41 56                   push   %r14
  4010f6:   41 55                   push   %r13
  4010f8:   41 54                   push   %r12
  4010fa:   55                      push   %rbp
  4010fb:   53                      push   %rbx
  4010fc:   48 83 ec 50             sub    $0x50,%rsp
  401100:   49 89 e5                mov    %rsp,%r13
  401103:   48 89 e6                mov    %rsp,%rsi
  401106:   e8 51 03 00 00          callq  40145c <read_six_numbers> #读6个数，保存到$rsp
# 步骤1：判断输入的每个数是否不超过6，且任意两个数都不相等
  40110b:   49 89 e6                mov    %rsp,%r14	
  40110e:   41 bc 00 00 00 00       mov    $0x0,%r12d			# %r12d = 0
  401114:   4c 89 ed                mov    %r13,%rbp			# 初始$rbp, %r13都指向第一个数
  401117:   41 8b 45 00             mov    0x0(%r13),%eax	
  40111b:   83 e8 01                sub    $0x1,%eax
  40111e:   83 f8 05                cmp    $0x5,%eax			# 每个数必须小于等于6，否则爆炸
  401121:   76 05                   jbe    401128 <phase_6+0x34>
  401123:   e8 12 03 00 00          callq  40143a <explode_bomb>
  401128:   41 83 c4 01             add    $0x1,%r12d			# %12d循环计数，每次加1
  40112c:   41 83 fc 06             cmp    $0x6,%r12d			# 第一重循环，终止条件是%r12d等于6
  401130:   74 21                   je     401153 <phase_6+0x5f>
  401132:   44 89 e3                mov    %r12d,%ebx			# %ebx <- %r12d 
  401135:   48 63 c3                movslq %ebx,%rax
  401138:   8b 04 84                mov    (%rsp,%rax,4),%eax	# 将输入的下一个整数保存到%eax
  40113b:   39 45 00                cmp    %eax,0x0(%rbp)		# 第二重循环，当前整数不能和它后面的任意一个数重复，否则爆炸； 两重循环用于确保输入的6个数没有重复数字，否则引爆炸弹
  40113e:   75 05                   jne    401145 <phase_6+0x51>
  401140:   e8 f5 02 00 00          callq  40143a <explode_bomb>
  401145:   83 c3 01                add    $0x1,%ebx
  401148:   83 fb 05                cmp    $0x5,%ebx
  40114b:   7e e8                   jle    401135 <phase_6+0x41>
  40114d:   49 83 c5 04             add    $0x4,%r13
  401151:   eb c1                   jmp    401114 <phase_6+0x20>
# 步骤2：对于每个输入的整数，做这样的转换：用7减去这个整数的值替换原来的数
  401153:   48 8d 74 24 18          lea    0x18(%rsp),%rsi
  401158:   4c 89 f0                mov    %r14,%rax 			
  40115b:   b9 07 00 00 00          mov    $0x7,%ecx 			# %ecx初值为7
  401160:   89 ca                   mov    %ecx,%edx
  401162:   2b 10                   sub    (%rax),%edx 			# %edx = 7 - (%rax), %rax指向当前整数
  401164:   89 10                   mov    %edx,(%rax) 			# (%rax) = %edx  
  401166:   48 83 c0 04             add    $0x4,%rax   			# 循环每执行一次, %rax指向下一个整数
  40116a:   48 39 f0                cmp    %rsi,%rax
  40116d:   75 f1                   jne    401160 <phase_6+0x6c>
# 步骤3：0x6032d0处表示一个包含6个节点的链表， 对于经过步骤2之后转换的每个整数i, 取链表第i个节点的value，依次保存在(%rsp + 32)处
  40116f:   be 00 00 00 00          mov    $0x0,%esi   # esi设为0
  401174:   eb 21                   jmp    401197 <phase_6+0xa3>
  401176:   48 8b 52 08             mov    0x8(%rdx),%rdx		# 访问链表
  40117a:   83 c0 01                add    $0x1,%eax
  40117d:   39 c8                   cmp    %ecx,%eax
  40117f:   75 f5                   jne    401176 <phase_6+0x82>
  401181:   eb 05                   jmp    401188 <phase_6+0x94>
  401183:   ba d0 32 60 00          mov    $0x6032d0,%edx 		# 0x6032d0处为链表，包含6个节点
  401188:   48 89 54 74 20          mov    %rdx,0x20(%rsp,%rsi,2) #每次取链表中第%ecx个节点的值，保存到$rsp + 0x20 + 2 * $rsi处， %ecx表示每个
  40118d:   48 83 c6 04             add    $0x4,%rsi
  401191:   48 83 fe 18             cmp    $0x18,%rsi
  401195:   74 14                   je     4011ab <phase_6+0xb7>
  401197:   8b 0c 34                mov    (%rsp,%rsi,1),%ecx	# ecx初始值为%rsp, 指向第一个数
  40119a:   83 f9 01                cmp    $0x1,%ecx
  40119d:   7e e4                   jle    401183 <phase_6+0x8f>
  40119f:   b8 01 00 00 00          mov    $0x1,%eax
  4011a4:   ba d0 32 60 00          mov    $0x6032d0,%edx		# 0x6032d0处为链表，包含6个节点
  4011a9:   eb cb                   jmp    401176 <phase_6+0x82>
  4011ab:   48 8b 5c 24 20          mov    0x20(%rsp),%rbx 		# 保存6个节点的值
  4011b0:   48 8d 44 24 28          lea    0x28(%rsp),%rax
  4011b5:   48 8d 74 24 50          lea    0x50(%rsp),%rsi
  4011ba:   48 89 d9                mov    %rbx,%rcx
  4011bd:   48 8b 10                mov    (%rax),%rdx
  4011c0:   48 89 51 08             mov    %rdx,0x8(%rcx)
  4011c4:   48 83 c0 08             add    $0x8,%rax
  4011c8:   48 39 f0                cmp    %rsi,%rax
  4011cb:   74 05                   je     4011d2 <phase_6+0xde>
  4011cd:   48 89 d1                mov    %rdx,%rcx
  4011d0:   eb eb                   jmp    4011bd <phase_6+0xc9>
# 4.判断（%rsp + 32）处的6个整数是否为降序排列，如不满足条件引爆炸弹
  4011d2:   48 c7 42 08 00 00 00    movq   $0x0,0x8(%rdx)
  4011d9:   00
  4011da:   bd 05 00 00 00          mov    $0x5,%ebp
  4011df:   48 8b 43 08             mov    0x8(%rbx),%rax		#将链表下一个节点地址给rax
  4011e3:   8b 00                   mov    (%rax),%eax			#eax为链表下一个节点的值
  4011e5:   39 03                   cmp    %eax,(%rbx)			# 比较前后两个节点的值
  4011e7:   7d 05                   jge    4011ee <phase_6+0xfa>#前一个数要大于后一个，否则炸弹爆炸。即必须为降序排列
  4011e9:   e8 4c 02 00 00          callq  40143a <explode_bomb>
  4011ee:   48 8b 5b 08             mov    0x8(%rbx),%rbx
  4011f2:   83 ed 01                sub    $0x1,%ebp
  4011f5:   75 e8                   jne    4011df <phase_6+0xeb>
  4011f7:   48 83 c4 50             add    $0x50,%rsp
  4011fb:   5b                      pop    %rbx
  4011fc:   5d                      pop    %rbp
  4011fd:   41 5c                   pop    %r12
  4011ff:   41 5d                   pop    %r13
  401201:   41 5e                   pop    %r14
  401203:   c3                      retq
```

`40111b ~ 40111e`： 将每个输入的整数和6比较，如存在某个数大于6，引爆炸弹。

`40110b ~ 401153`： 双重循环，用于判断输入的6个数字中是否存在两个数相同。如果存在，引爆炸弹。

举例：用户可以输入`1,2,3,4,5,6`，记为**序列0**，满足以上两个条件。

`401160 ~ 40116d`： 一重循环，对于**序列0**中的每个整数，做这样的转换：用`7`减去这个整数的结果替换原来的数，即得到**序列1**： `6,5,4,3,2,1`。

注意到`%edx`初值为`0x6032d0`， 打印这块内存，发现这是一条链表, 包含6个节点。

```
(gdb) x/24x 0x6032d0
0x6032d0 <node1>:       0x0000014c      0x00000001      0x006032e0      0x00000000
0x6032e0 <node2>:       0x000000a8      0x00000002      0x006032f0      0x00000000
0x6032f0 <node3>:       0x0000039c      0x00000003      0x00603300      0x00000000
0x603300 <node4>:       0x000002b3      0x00000004      0x00603310      0x00000000
0x603310 <node5>:       0x000001dd      0x00000005      0x00603320      0x00000000
0x603320 <node6>:       0x000001bb      0x00000006      0x00000000      0x00000000
```

`40116f ~ 4011d0`：遍历转换后的**序列1**，对于每个整数`i`, 取第`i`个`node`节点的值，依次存储到`%rsp + 32`处，本例中存储到`%rsp + 32`的6个数为`0x1bb`,`0x1dd`,`0x2b3`, `0x39c`, `0xa8`, `0x14c`,  记为**序列2**。

`4011d2 ~ 4011f5`：判断**序列2**是否为降序排列，本例中的**序列2**不满足条件。因此我们需要回过头，调整输入的6个整数的顺序，使得序列2为降序排列，过程如下：

* 链表中6个节点降序排列应为： `0x39c`,`0x2b3`,`0x1dd`,`0x1bb`,`0x14c`, `0xa8`

* 对应的6个节点序列为：`node3`,`node4`,`node5`,`node6`,`node1`,`node2`

* 推导出序列1: `3,4,5,6,1,2`

* 根据序列1逆推出输入：`7-3, 7-4, 7-5, 7-6, 7-1, 7-2` -> `4, 3, 5, 6, 1, 2` 

最终得到这一关的答案为`4,3,5,6,1,2`， 唯一解。

### 隐藏关

在汇编文件中搜`secret_phase`，发现`defused_defused`调用了它。先看看如何触发隐藏关，`phase_defused`代码如下：

```
00000000004015c4 <phase_defused>:
  4015c4:   48 83 ec 78             sub    $0x78,%rsp
  4015c8:   64 48 8b 04 25 28 00    mov    %fs:0x28,%rax
  4015cf:   00 00
  4015d1:   48 89 44 24 68          mov    %rax,0x68(%rsp)
  4015d6:   31 c0                   xor    %eax,%eax
  4015d8:   83 3d 81 21 20 00 06    cmpl   $0x6,0x202181(%rip)        # 603760 <num_input_strings>	仅当第6关通过后，不进行跳转，进入隐藏关
  4015df:   75 5e                   jne    40163f <phase_defused+0x7b>
  4015e1:   4c 8d 44 24 10          lea    0x10(%rsp),%r8
  4015e6:   48 8d 4c 24 0c          lea    0xc(%rsp),%rcx
  4015eb:   48 8d 54 24 08          lea    0x8(%rsp),%rdx
  4015f0:   be 19 26 40 00          mov    $0x402619,%esi	# 格式为"%d %d %s"
  4015f5:   bf 70 38 60 00          mov    $0x603870,%edi	# 0x603870处保存第四关输入的答案， 可通过对phase_defused打断点，或者对0x603870打数据断点确认
  4015fa:   e8 f1 f5 ff ff          callq  400bf0 <__isoc99_sscanf@plt>
  4015ff:   83 f8 03                cmp    $0x3,%eax					# 需要输入3个参数，才能触发隐藏关，否则跳转0x401635
  401602:   75 31                   jne    401635 <phase_defused+0x71>
  401604:   be 22 26 40 00          mov    $0x402622,%esi	# %esi处字符串"DrEvil"
  401609:   48 8d 7c 24 10          lea    0xls
  40160e:   e8 25 fd ff ff          callq  401338 <strings_not_equal>
  401613:   85 c0                   test   %eax,%eax
  401615:   75 1e                   jne    401635 <phase_defused+0x71>
  401617:   bf f8 24 40 00          mov    $0x4024f8,%edi
  40161c:   e8 ef f4 ff ff          callq  400b10 <puts@plt>
  401621:   bf 20 25 40 00          mov    $0x402520,%edi
  401626:   e8 e5 f4 ff ff          callq  400b10 <puts@plt>
  40162b:   b8 00 00 00 00          mov    $0x0,%eax
  401630:   e8 0d fc ff ff          callq  401242 <secret_phase>
  401635:   bf 58 25 40 00          mov    $0x402558,%edi
  40163a:   e8 d1 f4 ff ff          callq  400b10 <puts@plt>
  40163f:   48 8b 44 24 68          mov    0x68(%rsp),%rax
  401644:   64 48 33 04 25 28 00    xor    %fs:0x28,%rax
  40164b:   00 00
  40164d:   74 05                   je     401654 <phase_defused+0x90>
  40164f:   e8 dc f4 ff ff          callq  400b30 <__stack_chk_fail@plt>
  401654:   48 83 c4 78             add    $0x78,%rsp
  401658:   c3                      retq

```

首先查看`4015ff: cmp $0x3,%eax`， 说明`sscanf`需接受3个变参才能触发隐藏关。参数格式依次为`%d, %d, %s`

```
(gdb) x/s 0x402619
0x402619:       "%d %d %s"
```

接下来确定`0x603870`处字符串怎么来的。在`0x603870`处设置gdb数据断点, 发现`0x603870`处内容依次变为`7`, `7, 0`，然后程序退出。而`7 0`恰好是我们第四关输入的答案。 说明我们只需在第四关后添加一个合适的字符串，作为第3个参数，即可触发隐藏关。

```
(gdb) watch *0x603870							# 设置内存断点
Hardware watchpoint 5: *0x603870
(gdb) r
...
Hardware watchpoint 5: *0x603870
Old value = 0
New value = 55
0x00007ffff7aa1b53 in __memcpy_sse2 () from /lib64/libc.so.6
(gdb) x/s 0x603870
0x603870 <input_strings+240>:   "7"
(gdb) c
Continuing.
Hardware watchpoint 5: *0x603870
Old value = 55
New value = 3153975
0x00007ffff7aa1b64 in __memcpy_sse2 () from /lib64/libc.so.6
(gdb) x/s 0x603870
0x603870 <input_strings+240>:   "7 0"
```

根据`401604: mov $0x402622,%esi`， 确认输入的字符串为`"DrEvil"`

```s
(gdb) x/s 0x402622
0x402622:       "DrEvil"
```

因此，只需要将第四关答案改为`7 0 DrEvil`， 即可触发隐藏关。

#### 查看隐藏关代码

```txt
0000000000401242 <secret_phase>:
  401242:   53                      push   %rbx
  401243:   e8 56 02 00 00          callq  40149e <read_line>
  401248:   ba 0a 00 00 00          mov    $0xa,%edx		# strtol的第三个参数，base等于10
  40124d:   be 00 00 00 00          mov    $0x0,%esi		# strtol的第二个参数, endptr='\0'
  401252:   48 89 c7                mov    %rax,%rdi		# strtol的第一个参数，str=用户输入字符串
  401255:   e8 76 f9 ff ff          callq  400bd0 <strtol@plt>
  40125a:   48 89 c3                mov    %rax,%rbx		# 将用户输入的整数保存到%rbx
  40125d:   8d 40 ff                lea    -0x1(%rax),%eax 	# %eax = %rax - 1
  401260:   3d e8 03 00 00          cmp    $0x3e8,%eax		# 0x3e8 = 1000					
  401265:   76 05                   jbe    40126c <secret_phase+0x2a>
  401267:   e8 ce 01 00 00          callq  40143a <explode_bomb>
  40126c:   89 de                   mov    %ebx,%esi	 	# fun7的第二个参数, %rbx
  40126e:   bf f0 30 60 00          mov    $0x6030f0,%edi	# fun7 	
  401273:   e8 8c ff ff ff          callq  401204 <fun7>
  401278:   83 f8 02                cmp    $0x2,%eax		# fun7的返回值必须为2，否则引爆
  40127b:   74 05                   je     401282 <secret_phase+0x40>
  40127d:   e8 b8 01 00 00          callq  40143a <explode_bomb>
  401282:   bf 38 24 40 00          mov    $0x402438,%edi
  401287:   e8 84 f8 ff ff          callq  400b10 <puts@plt>
  40128c:   e8 33 03 00 00          callq  4015c4 <phase_defused>
  401291:   5b                      pop    %rbx
```

`0x401255`处调用了strtol函数，函数声明如下：

```C
long int strtol(const char *str, char **endptr, int base);
```

查看`0x401248 ~ 0x401267`，发现最后一关需要输入一个整数，并且该整数不能超过1001。

查看`0x40126c ~ 0x40127b`， 发现调用了`func7`函数， 该函数返回值必须为2;`func7`接受两个入参。

打印`fun7`的第一个参数, 发现这是一棵二叉树。

```
(gdb) x/120x 0x6030f0
0x6030f0 <n1>:  0x00000024      0x00000000      0x00603110      0x00000000
0x603100 <n1+16>:       0x00603130      0x00000000      0x00000000      0x00000000
0x603110 <n21>: 0x00000008      0x00000000      0x00603190      0x00000000
0x603120 <n21+16>:      0x00603150      0x00000000      0x00000000      0x00000000
0x603130 <n22>: 0x00000032      0x00000000      0x00603170      0x00000000
0x603140 <n22+16>:      0x006031b0      0x00000000      0x00000000      0x00000000
0x603150 <n32>: 0x00000016      0x00000000      0x00603270      0x00000000
0x603160 <n32+16>:      0x00603230      0x00000000      0x00000000      0x00000000
0x603170 <n33>: 0x0000002d      0x00000000      0x006031d0      0x00000000
0x603180 <n33+16>:      0x00603290      0x00000000      0x00000000      0x00000000
0x603190 <n31>: 0x00000006      0x00000000      0x006031f0      0x00000000
0x6031a0 <n31+16>:      0x00603250      0x00000000      0x00000000      0x00000000
0x6031b0 <n34>: 0x0000006b      0x00000000      0x00603210      0x00000000
0x6031c0 <n34+16>:      0x006032b0      0x00000000      0x00000000      0x00000000
0x6031d0 <n45>: 0x00000028      0x00000000      0x00000000      0x00000000
0x6031e0 <n45+16>:      0x00000000      0x00000000      0x00000000      0x00000000
0x6031f0 <n41>: 0x00000001      0x00000000      0x00000000      0x00000000
0x603200 <n41+16>:      0x00000000      0x00000000      0x00000000      0x00000000
0x603210 <n47>: 0x00000063      0x00000000      0x00000000      0x00000000
0x603220 <n47+16>:      0x00000000      0x00000000      0x00000000      0x00000000
0x603230 <n44>: 0x00000023      0x00000000      0x00000000      0x00000000
0x603240 <n44+16>:      0x00000000      0x00000000      0x00000000      0x00000000
0x603250 <n42>: 0x00000007      0x00000000      0x00000000      0x00000000
0x603260 <n42+16>:      0x00000000      0x00000000      0x00000000      0x00000000
0x603270 <n43>: 0x00000014      0x00000000      0x00000000      0x00000000
0x603280 <n43+16>:      0x00000000      0x00000000      0x00000000      0x00000000
0x603290 <n46>: 0x0000002f      0x00000000      0x00000000      0x00000000
0x6032a0 <n46+16>:      0x00000000      0x00000000      0x00000000      0x00000000
0x6032b0 <n48>: 0x000003e9      0x00000000      0x00000000      0x00000000
0x6032c0 <n48+16>:      0x00000000      0x00000000      0x00000000      0x00000000
```

把这棵二叉树画出来, 发现有4层，共15个节点：

```txt
                                        0x24
                              /                       \								
                 0x8                                            0x32
            /           \                                   /           \
      0x6                   0x16                    0x2d                    0x6b
    /    \                 /     \                 /     \                 /     \
0x1        0x7        0x14        0x23        0x28        0x2f        0x63        0x3e9
```

`func7`代码如下，可以看出这是个递归函数。

```
0000000000401204 <fun7>:
  401204:   48 83 ec 08             sub    $0x8,%rsp
  401208:   48 85 ff                test   %rdi,%rdi
  40120b:   74 2b                   je     401238 <fun7+0x34>
  40120d:   8b 17                   mov    (%rdi),%edx		# 首次调用fun7时， rdi指向二叉树的根节点
  40120f:   39 f2                   cmp    %esi,%edx	# 如当前节点值小于等于用户输入的数，跳转到0x401220 
  401211:   7e 0d                   jle    401220 <fun7+0x1c>
  401213:   48 8b 7f 08             mov    0x8(%rdi),%rdi	# 取当前节点的左子女
  401217:   e8 e8 ff ff ff          callq  401204 <fun7>
  40121c:   01 c0                   add    %eax,%eax
  40121e:   eb 1d                   jmp    40123d <fun7+0x39>
  401220:   b8 00 00 00 00          mov    $0x0,%eax
  401225:   39 f2                   cmp    %esi,%edx
  401227:   74 14                   je     40123d <fun7+0x39>
  401229:   48 8b 7f 10             mov    0x10(%rdi),%rdi	# 取当前节点的右子女
  40122d:   e8 d2 ff ff ff          callq  401204 <fun7>
  401232:   8d 44 00 01             lea    0x1(%rax,%rax,1),%eax # %eax = 2 * $rax + 1
  401236:   eb 05                   jmp    40123d <fun7+0x39>
  401238:   b8 ff ff ff ff          mov    $0xffffffff,%eax	# 当前节点为NULL，返回-1
  40123d:   48 83 c4 08             add    $0x8,%rsp
  401241:   c3                      retq
```

将递归逻辑转换为C语言， 如下：

```C
// 二叉树的数组表示，树高4层, 共15个节点
const int arrTree[] = { 0x24,				
                        0x8, 0x32,
                        0x6, 0x16, 0x2d, 0x6b,
                        0x1, 0x7 , 0x14, 0x23, 0x28, 0x2f, 0x63, 0x3e9 };
// idx表示数组arrTree索引，userInput即用户输入的答案
int fun7(int idx, int userInput) {	
    if(idx < 0 || idx > 14) {
        return -1;  // 如当前节点为NULL, 返回-1
    }
    int ret = 0;
    if(arrTree[idx] < userInput) {
        ret = 2 * fun7(2 * idx + 2, userInput) + 1; // 取当前节点的右子女
    } else if(arrTree[idx] > userInput) {
        ret = 2 * fun7(2 * idx + 1, userInput); 	// 取当前节点的左子女
    }
    return ret;
}

int main() {
    for(int i = 0; i < 1002; ++i)
        if(fun7(0, i) == 2) // idx初值为0, 表示从二叉树的根开始递归
            printf("answer: %d\n", i);
    return 0;
}
```

执行C程序，打印如下，说明答案有两组, 输入`20`或`22`都可以。

```txt
answer: 20
answer: 22
```

### 总结

所有关卡的答案保存在`answer`文件中，内容如下：

```txt
# cat answer
Border relations with Canada have never been better.
1 2 4 8 16 32
6 682
7 0 DrEvil
IONUVW
4 3 2 1 6 5
20
```

执行结果如下：

```txt
# ./bomb answer
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Phase 1 defused. How about the next one?
That's number 2.  Keep going!
Halfway there!
So you got that one.  Try this one.
Good work!  On to the next...
Curses, you've found the secret phase!
But finding it and solving it are quite different...
Wow! You've defused the secret stage!
Congratulations! You've defused the bomb!
```

### 参考资料

《深入理解计算机系统 原书第3版》





























































































































































