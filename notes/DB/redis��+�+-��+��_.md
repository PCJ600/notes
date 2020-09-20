#### 写入数据失败

MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk

原因：究其原因是因为强制把redis快照关闭了导致不能持久化的问题

解决方法：

命令行：

127.0.0.1:6379> config set stop-writes-on-bgsave-error no

改配置文件：

 修改redis.conf文件：vi打开redis-server配置的redis.conf文件，然后使用快捷匹配模式：/stop-writes-on-bgsave-error定位到stop-writes-on-bgsave-error字符串所在位置，接着把后面的yes设置为no即可。

