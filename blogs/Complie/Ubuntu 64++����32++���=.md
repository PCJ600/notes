### 64位linux使用gcc编译32位程序

#### Ubuntu

安装以下软件包

```shell
apt-get install build-essential module-assistant gcc-multilib g++-multilib
```

#### Centos

安装以下软件包

```shell
yum install glibc-devel.i686 libstdc++-devel.i686 
```

gcc编译添加-m32参数，如

```shell
gcc -m32 main.c
```

#### 参考链接

[http://notes.maxwi.com/2017/12/06/compile-x32-executable-at-x64-linux-system/](URL http://notes.maxwi.com/2017/12/06/compile-x32-executable-at-x64-linux-system/)



