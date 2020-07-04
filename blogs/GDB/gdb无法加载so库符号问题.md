## gdb加载so库符号失败的解决方法

#### 问题现象

gdb调试core文件或进程时，出现加载so库符号失败，错误信息如下

```shell
warning: Could not load shared library symbols for ../libadd.so
Do you need "set solib-search-path" or "set sysroot"?
```

执行**info sharedlibrary**，查看Syms Read字段为No,  表示对应so库符号加载失败。

```shell
$ pwd /home/gdb
(gdb) info sharedlibrary
From                To                  Syms Read   Shared Object Library
0x00007fba2c572570  0x00007fba2c57267b  No          ../libadd.so
0x00007fba2c370570  0x00007fba2c37066b  No          ../../var/libsub.so
(gdb) bt
#0  0x00007fba2c57266b in ?? ()
#1  0x00007ffc6f703ff0 in ?? ()
```

#### 解决方法

设置gdb的**solib-search-path**选项， 指定加载失败的so的搜索路径即可。

solib-search-path可以指定多个路径。在linux上，路径之间用冒号分隔，命令如下:

```
(gdb) set solib-search-path /var:/home
Reading symbols from ../libadd.so...done.
Loaded symbols for ../libadd.so
Reading symbols from ../../var/libsub.so...done.
Loaded symbols for ../../var/libsub.so
(gdb) info sharedlibrary
From                To                  Syms Read   Shared Object Library
0x00007fba2c572570  0x00007fba2c57267b  Yes         ../libadd.so
0x00007fba2c370570  0x00007fba2c37066b  Yes         ../../var/libsub.so
(gdb) bt
#0  0x00007fba2c57266b in add (a=1, b=2) at add.c:5
#1  0x0000000000400600 in ?? ()
```

或者设置gdb的**solib-absolute-prefix**选项，指定被搜索so文件路径的前缀， 与solib-search-path区别在于solib-absolute-prefix只能有一个，使用如下gdb指令：

```shell
(gdb) set solib-absolute-prefix /		
(gdb) set sysroot /					# sysroot是solib-absolute-prefix的别名
```

#### 参考资料

[https://visualgdb.com/gdbreference/commands/set_solib-search-path](https://visualgdb.com/gdbreference/commands/set_solib-search-path)

