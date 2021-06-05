## objdump命令和工具

### ar



### objdump

1 打印各个段基本信息

objdump -h *.so 

2 打印各个段的内容

objdump -s *.so

3 反汇编

objdump -d *.soll

4 查看目标文件的重定位表

objdump -r a.o

5  查看函数所在的目标文件

objdump -t *.a



### readelf

readelf -S a.out 显示真正的段表结构

readelf --string-dump=39 a.out   (30指段表字符串表.shstrtab在段表中的下标)

readelf -l *.elf

从装载的角度，描述了ELF文件如何被操作系统映射到进程的虚拟空间。



### name

nm *.o



### GCC编译选项

-fno-builtin

默认情况下，GCC会自作聪明地讲只有一个字符串参数的printf替换成puts函数