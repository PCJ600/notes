## 记一次UDP的sendto函数错误解决

### 错误现象

在编写使用select函数的TCP和UDP回射程序，出现UDP的sendto错误，现象如下：

服务端正常启动后，调用select函数监听TCP和UDP套接字, 可以正常处理TCP请求。

UDP客户端可以连接到服务端，但接收标准输入后无回显，阻塞于recvfrom。

### 解决过程

经排查，发现服务端处理UDP请求的代码有问题，如下:

```C
typedef struct sockaddr SA;
void udp_echo(int udpfd) {
	int n;
	char recvline[MAXLINE];
	struct sockaddr_in cliaddr;
	socklen_t len;
	n = recvfrom(udpfd, recvline, MAXLINE, 0, (SA*)&cliaddr, &len);
	sendto(udpfd, recvline, n, 0, (SA*)&cliaddr, len);
}
```

udp_echo函数，先用recvfrom读取UDP客户端发送的字符串，再使用sendto将该字符串送回客户端。代码中没有判断recvfrom, sendto函数的返回值，为获取出错信息改写如下:

```C
void udp_echo(int udpfd) {
	int n;
	char recvline[MAXLINE];
	struct sockaddr_in cliaddr;
	socklen_t len;
	if((n = recvfrom(udpfd, recvline, MAXLINE, 0, (SA*)&cliaddr, &len)) < 0) {
		err_sys("recvfrom error");
	}
	if(sendto(udpfd, recvline, n, 0, (SA*)&cliaddr, len) != n) {
		printf("error: %s\n", strerror(errno));
		err_sys("sendto error");
    }
}
```

再运行程序，输出如下:

$ error: Invalid argument

$ sendto error

错误原因在于cliaddr参数没有初始化。**sockaddr_in结构体在使用之前，需要先使用bzero/memset函数初始化为0**，否则出现赋值不完整导致参数无效。

#### 总结：

* 变量使用之前最好初始化。

* 注意判断函数的返回值，可以使用包裹函数。

