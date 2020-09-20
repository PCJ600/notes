## Linux 常见控制字符

### ctrl + c 

中断键，给前台进程组中所有进程发送`SIGINT`信号，并终止进程。

### ctrl + z 

挂起键，给前台进程组中所有进程发送`SIGTSTP`信号,  并挂起进程。被挂起的进程并没有真正结束，可以使用`fg`或`bg`命令恢复被挂起的进程。

* fg —— 将后台作业放到前台终端运行。例如用VIM编辑文件时，需要敲shell命令。可以先用`Ctrl + Z`挂起VIM，敲完shell命令后再使用`fg`命令恢复VIM继续编辑，好处是不用退出VIM程序。
* bg —— 恢复后台被挂起的作业，变成在后台继续执行。例如前台启动一个程序时，不希望一直等待程序运行结束，可以先用`Ctrl+Z`挂起进程，再使用`bg`命令后台恢复程序的执行，好处是不用终止程序。
* jobs —— 显示当前shell中后台正在运行或被挂起的任务列表。

### ctrl + d

表示一个特殊二进制值`EOF`，表示已到达文件末尾(`end of file`), 可以用来快速退出终端。

### ctrl + s

中断控制台的输出。有时终端卡死了，敲什么都没反应，很可能是敲了`Ctrl + S`，可以接着敲`Ctrl + q`恢复。

### ctrl + \

终止进程，并向进程发送`SIGQUIT`信号，默认会产生`coredump`文件。

### ctrl + l

清屏， 相当于终端里敲`clear`。

### 参考资料

[Linux常见信号大全](https://www.jianshu.com/p/730989a7302e)

[Linux中ctrl-c, ctrl-z, ctrl-d区别](https://blog.csdn.net/mylizh/article/details/38385739?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-7.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-7.channel_param)

