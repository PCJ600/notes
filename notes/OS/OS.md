### OS

计算模型 ——> 关注指针IP及其指向的内容

实模式寻址CS:IP (CS左移4位 + IP)

* x86 PC刚开机时CPU处于实模式
* 开机时，CS=0xFFFF; IP=0x0000
* 寻址0xFFFF0 (ROM BIOS映射区)
* 检查RAM，键盘，显示器，软硬磁盘 -> 这步为止仅涉及硬件，不涉及操作系统

* 将磁盘0磁道0扇区读入0x7c00处
* 设置CS=0x07c0, ip=0x0000

### 0x7c00处存放的代码

就是从磁盘引导扇区读入的512个字节

引导扇区代码： bootsect.s 

![image-20210118225820128](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\image-20210118225820128.png)



`jmpi go INITSEG`： 段间跳转 go -> IP, INITSEG -> CS

### 系统调用的实现

CPL=3 DPL=0 -> 通过int 0x80中断 CPL=3, DPL=3  -> 进入内核态system_call CPL= 0

### 多进程的组织：PCB+状态+队列

就绪态，运行态，阻塞态

进程交替：FIFO,  Priority,  将CPU信息保存到PCB

进程同步：合理的推进顺序，上锁

#### 如何形成多进程图像：

* 读写PCB
* 操作寄存器完成切换
* 调度程序
* 进程同步与合作
* 地址映射

### 用户级线程

两个线程共用一个栈为什么不行？

![image-20210131130138221](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\image-20210131130138221.png)



![image-20210131131227087](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\image-20210131131227087.png)







