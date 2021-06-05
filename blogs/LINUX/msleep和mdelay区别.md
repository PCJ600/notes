## 内核延时函数msleep和mdelay的区别

`msleep`和`mdelay`都是内核的延时函数，原型如下：

```C
void mdelay(unsigned long msecs); 
void msleep(unsigned int millisecs);
```

#### 区别

`mdelay`是忙等待函数，会占用`CPU`资源，延迟时间是准确的。

`msleep`是休眠函数，不占用`CPU`资源，延迟时间通常高于给定值。

**具体可以参考如下文章：**

[The difference between Mdelay and Msleep in Linux](https://topic.alibabacloud.com/a/the-difference-between-mdelay--and-msleep--in-linux-linux_1_16_20266988.html)

[Linux中内核延时函数](https://www.cnblogs.com/xihong2014/p/6740876.html)

