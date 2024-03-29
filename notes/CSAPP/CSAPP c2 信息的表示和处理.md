### 信息的表示和处理

### 2.1 信息存储

字节，最小的可寻址存储器单位，8位

字长， 指明整数和指针数据的标称大小，对于w位机器，虚拟地址0~2

掌握十六进制、二进制、十进制相互转化

#### 数据大小

| C声明 | 32位机器 | 64位机器 |
| ----- | -------- | -------- |
| char | 1 | 1 |
| short | 2 | 2 |
| int | 4 | 4 |
| long int | 4 | 8 |
| long long int | 8 | 8 |
| char * | 4 | 8 |
| float | 4 | 4 |
| double | 4 | 8 |
| size_t | 4 | 8 |
| ssize_t | 4 | 8 |

#### 寻址和端序

假设x类型为int, 地址0x100, 值为0x01234567

大端法排列:

| 0x100 | 0x101 | 0x102 | 0x103 |
| ----- | ----- | ----- | ----- |
| 01    | 23    | 45    | 67    |

小端法排列

| 0x100 | 0x101 | 0x102 | 0x103 |
| ----- | ----- | ----- | ----- |
| 67    | 45    | 23    | 01    |

#### 如何判断端序

```C
union {
	char a;
	int b;
}u;
int main() {
	u.b = 0x12345678;
	if(u.a == 0x78) {
		printf("little endian!\n");
	} else {
		printf("big endian!\n");
	}
    return 0;
} 
```

#### 打印程序对象的字节表示

```C
typedef unsigned char *byte_order;
void show_bytes(byte_pointer start, int len) {
	int i;
	for(int i = 0; i < len; ++i) {
		printf("%.2x", start[i]);
	}
}
```

上面的强制类型转换不会改变真实的指针，只告诉编译器以新的数据类型看待被指向的数据

#### %p, %x的区别

```C
*p用于输出指针，%x输出16进制数
当机器为64位时，%p输出指针结果正确，%x有问题 			//为什么？
```

#### 字符串表示

C语言中字符串被编码为一个以Null字符结尾的数组，比如：

* show_bytes打印字符串"12345"，得到30 31 32 33 34 35 00

* 十进制x的ASCII码正好对应0x3x，终止字节十六进制为0x00

#### 交换两个数的几种方法

```c
// x和y地址相同时，会失效，将内容置为0
void mySwap(int *x, int *y) {
	*y = *x ^ *y;
	*x = *x ^ *y;
	*y = *x ^ *y;
}
```

#### 掌握C语言位级运算

写出变量x的C表达式，代码需要对任何字长w >= 8都能工作, 如x=0x87654321, w = 32

* x的最低有效字节，其他位均为0  0x21 —— x & 0xFF

* 除x最低有效字节，其他位取补，最低有效字节保持不变 0x789ABC21 —— x ^ ~0xFF

* x的最低有效字节设置为全1，其他字节保持不变 0x876543FF —— x | 0xFF

#### C语言逻辑运算

区分位级运算，仅返回TRUE， FALSE； 逻辑运算有短路求值

#### C语言移位运算

左移： x向左移k位，丢弃最高k位，并在右端补0

右移：对于无符号数据总是逻辑右移，有符号数据：算术和逻辑都可以

#### 对于一个w位的数据类型，如果移动k >= w位有什么结果?

不确定，可能全为0,  移位数量应该小于字长

```C
printf("%x\n", 0xFEDCBA98 >> 4); 				//输出0xFEDCBA9， 逻辑右移
printf("%x\n", (int)0xFEDCBA98 >> 4);			//输出0xFFEDCBA9， 算术右移
```

#### 整数表示范围

#### 整数编码

* 无符号整数  B2U
* 补码     B2T
* 反码， 原码 (对于数字0有两种不同编码方式)

反码: 最高有效位是-(2w-1 - 1)而不是-2w-1

原码: 最高有效位是符号位

#### 有符号数和无符号数转换

声明像12345,0x1A2B这样的常量时，这个值被认为是有符号的

要创建无符号常量，必须加上后缀字符'U'或者'u'

```C
short int v = -12345;						// 0xCFC7
unsigned short uv = (unsigned short) v;
printf("v = %d, uv = %u\n", v , uv);	// v = -12345, uv = 53191;
```

转换原则是底层的位表示不变，只改变解释方式。

表达式中整数默认int类型，浮点数默认double类型。

#### INT_MIN 为什么写成 -INT_MAX - 1

直接写成-2147483648，该数为long long类型，与int数比较会发生隐式类型转换

#### 扩展一个数字的位表示

* 零扩展
* 符号扩展

short 转换为 unsigned int时，<font color='red'>先改变大小，再完成从有符号到无符号的转换</font>

```shell
short sx = -12345;	// 0xcfc7
unsigned uy = sx;	// 应该输出0xffffcfc7， 而不是0x0000cfc7
```

#### 截断数字

将w位数截断为一个ke位数字，直接丢弃高位即可

```
int x = 53191;	// 0x0000cfc7
short sx = (short) x;	// 0xcfc7, -12345
int y = sx;				// 0xffffcfc7, -12345
```

#### 无符号运算注意问题

##### 1. 字符串比较发生的错误

```C
size_t strlen(const char *s);
int strlonger(char *s, char *t) {
	return strlen(s) - strlen(t) > 0;	//左边运算结果是无符号整型，右边是有符号数
}
```

##### 2. getpeername函数的安全漏洞

```C
int copy_from_kernel(void *user_dst, int maxlen) {
    int len = KSIZE < maxlen ? KSIZE : maxlen;
    memcpy(user_dst, kbuf, len); //如果maxlen传入-1, 由于size_t在32位机器上是unsigned int, memcpy将其解析为非常大的正整数，导致错误
    return len;
}
```

#### 整数运算

#### 无符号加法

##### 判断两个无符号加法是否溢出

```C
int uadd_ok(unsigned x, unsigned y) {
	unsigned sum = x + y;
	return sum >= x;
}
```

#### 补码加法

##### 判断两个有符号数加法是否溢出

```C
int tadd_ok(int x, int y) {
    int sum = x+y;    
    int neg_over = x < 0 && y < 0 && sum >= 0;
    int pos_over = x >= 0 && y >= 0 && sum < 0;
    return !neg_over && !pos_over;
}
// 正正得负，负负得正，也就发生了溢出
```

#### 整数减法

参照加法，只是把减数取相反数，两种方法取相反数：

* 各位取反加1
* 从右至至左找到第一个1，将这个1左边所有位取反

##### 判断两个与有符号数减法是否溢出

```C
int tsub_ok(int x, int y) {
	int sum = x - y;
	int neg_over = (x < 0 && y >= 0) && sum > 0;
	int pos_over = (x >= 0 && y < 0) && sum <= 0;
	return !(neg_over || pos_over);
}
```

#### 无符号乘法

C语言中无符号乘法定义为产生w位值，而计算结果可能需要2w位表示，需要截断

#### 补码乘法

先按照无符号乘法规则得到w位结果，再将该结果转换为补码

#### 判断两个有符号整数乘法是否溢出

```C
int tmul_ok(int x, int y) {
	int p = x * y;
	return !x || p/x == y;
}
```

#### 移位代替乘除法

补码除法

```C
(x < 0 ? (x + 1 << k - 1) : x) >> k
```

实现x除以16

```C
int div16(int x) {
	int bias = (x >> 31) & 0xF;
	return (x + bias) >> 4;
}
```

#### IEEE浮点数表示

IEEE浮点数用V = (-1)^s*M*2^E表示, 由三部分组成

* 符号，首位，1表示负数，0表示正数
* 阶码，对浮点数加权, 值为e - Bias，Bias值单精度127， 双精度1023(2^k-1)
* 尾数，二进制小数

![浮点数表示法](C:\PC\learning\notes\CSAPP\images\浮点数表示法.jpg)

给定位表示，根据exp值，被编码的值分为三种情况

| 类型       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| 规格化     | exp值不为全0，也不为全1，此时阶码值E = e - Bias, Bias是一个2^(k-1)-1的偏置值，单精度127，双精度1023，尾数M可看作1 + f |
| 非规格化   | 阶码域全为0, 此时阶码值为1 - Bias，尾数看作f，不包含隐藏的1。用途在于表示0和非常接近于0.0的数 |
| 无穷大/NaN | 阶码全为1。 当小数域全为0表示无穷，非零时表示非数"NAN"       |

#### 练习：12345.0为什么是0x4640E400

* 求12345的二进制表示 0x3039 -> 0011 0000 0011 1001b = 1.1000000111001 * 2^13 

* 求得阶码13 + 127 = 140 -> 1000 1100b, 符号位为0， 尾数从小数点开始，后位补0
* 0100 0110 0100 0000 1110 0010 0000 0000b -> 0x4640E400

#### 浮点运算

浮点加法可交换，不可结合

(3.14+1e10)-1e10 ，由于溢出求值为0

#### 课后作业

位运算实验 http://csapp.cs.cmu.edu/3e/labs.html

#### 参考资料

深入理解计算机系统(原书第3版)










































































































