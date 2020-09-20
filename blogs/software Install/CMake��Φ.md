## CMake源码安装

从官网安装指定版本， 以`3.12.1`版本为例

```shell
wget https://cmake.org/files/v3.12/cmake-3.12.1.tar.gz
tar -zxvf cmake-3.12.1.tar.gz
cd cmake-3.12.1
./bootstrap
make -j8
make install
```

查看cmake版本，检查是否安装成功

```shell
cmake --version
```

