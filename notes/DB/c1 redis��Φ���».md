## redis安装与启动

### 一、源码安装redis

```shell
# 0.官网下载最新redis源码包
wget http://download.redis.io/releases/redis-5.0.5.tar.gz
# 1.将redis.tar.gz拷贝到/usr/local目录并解压
mkdir -p /usr/local/ && cd /usr/local
cp /path/to/redis-5.0.5.tar.gz .
tar -zxvf redis-5.0.5.tar.gz
mv redis-5.0.5 redis && cd redis
# 2.编译、测试、安装
make -j4 						# 编译
make test						# 测试，显示All tests passed without errors
make install					# 安装
```

### 二、启动和停止redis

#### 通过命令行启动redis

```shell
redis-server 								#直接启动
redis-server --port 6380					#自定义端口号启动，--port指定端口为6380
redis-server /usr/local/redis/redis.conf    #启动redis时指定配置文件
```

#### 通过初始化脚本启动redis

* 在redis源码路径的utils目录中找到初始化脚本redis_init_script, 将该脚本复制到/etc/init.d，文件名改为redis_端口号(如redis_6379)，脚本中修改REDISPORT变量为该端口号(如6379)

* 建立需要的目录， 新键/etc/redis目录用于存放redis配置文件，新建/var/redis/端口号(如/var/redis/6379)目录存放redis持久化文件

* 修改配置文件，将redis.conf复制到/etc/redis下，并改名为端口号.conf(如6379.conf)，需要修改部分参数，如下表

| 参数      | 值   | 说明                |
| --------- | ---- | ------------------- |
| daemonize | yes  | 以守护模式运行redis |
| pidfile | /var/run/redis_端口号.pid | 设置redis的PID文件位置 |
| port | 端口号 | 设置redis监听的端口号，如6379 |
| dir | /var/redis/端口号 | 设置持久化文件存放位置 |

#### 设置redis随系统自启动(Centos)

```shell
chkconfig --add redis_6379 		#增加redis服务，并通过chkconfig管理
chkconfig redis_6379 on 		#开启服务
chkconfig --list 				#查看redis服务级别，默认2，3，4，5为ON表示成功开启
```

#### 停止redis

强行终止redis可能导致数据丢失，正确停止redis方式是发送SHUTDOWN命令

```
redis-cli SHUTDOWN
```

redis收到SHUTDOWN命令后，会先断开所有客户端连接，然后根据配置执行持久化，最后退出

redis可以妥善处理SIGTERM信号，所以使用kill redis进程的PID也可以正常结束redis

#### 参考资料

[1]《Redis入门指南 第2版》 

[2] redis安装教程：<a href="https://blog.csdn.net/qq_36737803/article/details/90578860">https://blog.csdn.net/qq_36737803/article/details/90578860</a>





































































