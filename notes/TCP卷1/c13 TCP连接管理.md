#### 13.TCP连接管理

#### 初始序列号

一个32位的计数器，每4微妙加1

目的在于为一个连接的报文段安排序列号，以防出现与其他连接的序列号重叠的情况

#### 建立连接超时

telnet一个同一子网不存在的主机，不修改ARP表情况下返回 unable to connect to host

如在ARP表添加一条记录，系统无需ARP请求立即尝试建立连接，命令如下：

```
arp -s 192.168.43.241 00:00:1a:1b:1c:1d
telnet 192.168.43.241
```

发生连接超时，**指数回退**，抓包如下：

```shell
root@ubuntu1:/home/panchuang# tcpdump -i enp0s3 'port 23' -S
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s3, link-type EN10MB (Ethernet), capture size 262144 bytes
22:30:15.449352 IP 192.168.43.240.53628 > 192.168.43.241.telnet: Flags [S], seq 3829841400, win 29200, options [mss 1460,sackOK,TS val 227805 ecr 0,nop,wscale 7], length 0
22:30:16.447338 IP 192.168.43.240.53628 > 192.168.43.241.telnet: Flags [S], seq 3829841400, win 29200, options [mss 1460,sackOK,TS val 228055 ecr 0,nop,wscale 7], length 0
22:30:18.452395 IP 192.168.43.240.53628 > 192.168.43.241.telnet: Flags [S], seq 3829841400, win 29200, options [mss 1460,sackOK,TS val 228556 ecr 0,nop,wscale 7], length 0
22:30:22.460544 IP 192.168.43.240.53628 > 192.168.43.241.telnet: Flags [S], seq 3829841400, win 29200, options [mss 1460,sackOK,TS val 229558 ecr 0,nop,wscale 7], length 0
22:30:30.468308 IP 192.168.43.240.53628 > 192.168.43.241.telnet: Flags [S], seq 3829841400, win 29200, options [mss 1460,sackOK,TS val 231560 ecr 0,nop,wscale 7], length 0
```

#### 最大段选项MSS

最大段大小默认536字节，刚好组成20 + 20 + 536 = 576字节的v4数据报

v4典型值1460，20IP首部+20TCP首部，共1500字节，为以太网中最大传输单元和互联网路径最大传输单元的典型值

v6为1440字节，因为v6首部比v4多20字节

**注:**最大段大小不是TCP通信双方的协商结果，而是一个限定的数值。当通信的一方将自己的最大段大小选项发送给对方时，表示自己不愿意接收任何大于该尺寸的报文段。

#### 选择确认选项

#### 窗口缩放选项

#### 时间戳选项

#### 状态转换图

































