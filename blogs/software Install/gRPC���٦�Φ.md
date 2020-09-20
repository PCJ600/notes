## gRPC编译安装

```shell
sudo apt-get install pkg-config
sudo apt-get install autoconf automake libtool make g++ unzip
sudo apt-get install libgflags-dev libgtest-dev
sudo apt-get install clang libc++-dev

git clone https://github.com/grpc/grpc.git
git submodule update --init

grpc/third_party/protobuf/
git submodule update --init --recursive #更新第三方源码
sudo ./autogen.sh   #生成配置脚本
sudo ./configure    #生成Makefile文件，默认路径为/usr/local/
sudo make
sudo make install 
sudo ldconfig       #更新共享库缓存
protoc --version
```

