## CSAPP c4

### 对程序只使用安全的优化

* 内存别名使用
* 函数调用

```C
void t1(long *xp, long *yp) {
	*xp += *yp;
	*xp += *yp;
}
void t2(long *xp, long *yp) {
	*xp += 2 * (*yp);
}
```

当yp等于xp时，两者并不等价！

```C
long f();
long func1() {
	return f() + f() + f() + f();
}
long func2() {
	return 4 * f();
}
```

编译器会假设最糟的情况，保持函数调用不变

### 优化方法

* 消除循环的低效率， strlen的例子

* 减少过程调用

* 消除不必要的内存引用