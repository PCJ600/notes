## 存储器层次结构

### 随机访问存储器(RAM)

易失的，断电即丢失信息。

* 静态 SRAM     应用于高速缓存存储器

* 动态 DRAM    应用于主存

### 只读存储器(ROM)

掉电不丢失信息, 如闪存(flash)

### 局部性

时间局部性：被引用过一次的内存位置很可能在不远将来再次引用。

空间局部性：如一个内存位置被引用一次，程序很可能在不远将来引用附近的一个内存位置。

**小结：**

重复引用相同变量的程序有良好的时间局部性。

步长越小，空间局部性越好

### 存储器层次结构

![存储器层次结构](C:\Users\pc\Desktop\图片\存储器层次结构.PNG)

存储器层次结构的中心思想，对于每个k, 位于k层的更快更小的设备作为位于k+1层的更大更慢的存储设备的缓存。

#### 相关概念

| 术语 | 解释 |
| ---- | ---- |
|  块    | 数据总是以块大小为传送单元在k和k+1层传输 |
|  缓存命中/不命中    | 程序需要k+1层数据时，首先在k层查找，如刚好在k层查到即命中 |
| 替换/驱逐   | 覆盖一个现存块 |
|  冷缓存，强制性不命中    | 一个空的缓存称为冷缓存，此时任何访问都不命中 |
|  冲突不命中     | 缓存空间足够，但这些对象被映射到同一个缓存块 |

#### 通用高速缓存结构

参考cacheLab实验 + 做题 《原书P425》

为什么用中间位做索引？—— 充分利用缓存

#### 高速缓存行、组、块区别

数据总是以块大小为传送单元在k和k+1层传输

行是高速缓存一个容器，存储块及其他信息（如有效位和标记位)

组是一行或多行的集合。



#### 写

直写(write through) + 写不分配

**回写(write back) + 写分配**

直写法：立即将w高速缓存块写回到紧接着的低一层。缺点是每次写引起总线流量

写回法：尽可能推迟更新，只有替换算法要驱逐这个更新过的块时，才写到低一层。缺点时增加了复杂性，需为每个缓存行维护一个额外的修改位(dirty bit), 表示其是否修改过

写分配法：加载低一层中的块到高速缓存

非写分配法：直接把这个字写到低一层中









