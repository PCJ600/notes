## Socket编程简介

**套接字地址结构**

```C
#include<netinet/in.h>
typedef uint32_t in_addr_t;
struct in_addr {
    in_addr_t s_addr;      // 32为IPV4地址
};
struct sockaddr_in {
    uint8_t sin_len;         // 16字节，4字节对齐
	sa_family_t sin_family;  // 16位协议族 AF_INET
	in_port_t sin_port;      // 16位端口号，网络字节序
	struct in_addr sin_addr; // 32位IPV4地址
	char sin_zero[8];        // 未使用字段
};
```

**IPV4通用套接字地址结构**

```C
#include<sys/socket.h>
struct sockaddr {
    uint8_t sin_len;
    sa_family_t sa_family;
    char sa_data[14];
};
```

**IPV6套接字地址结构**

```C
struct in6_addr {
	uint8_t s6_addr[16];   // 128位IPV6地址
};

struct sockaddr_in6 {
	uint8_t sin6_len;           // 4字节对齐
	sa_family_t sin6_family;    // AF_INET6 地址族
	in_port_t sin6_port;        // 端口 
	uint32_t sin6_flowinfo;     // 低序20位流标，高序12位保留
	struct in6_addr sin6_addr;  // 128位IPV6地址
	uint32_t sin6_scope_id;     
};
```

进程->内核传递socket地址结构，bind, connect, sendto,

内核->进程传递socket地址结构，accept, recvfrom, getpeername, getsockname

**端序及转换函数**

低序字节存储在起始地址 -> 小端序

Htons, h代表host, n代表network, s代表short, l代表32位 

网际协议采取**大端字节序**

**字节操作函数：**

Void bzero(void *dst, size_t nbytes);   // memset替代

Void bcopy(const void *src, void *dst, size nbytes) // memcpy替代

Int bcmp(const void *ptr1, const void *ptr2, size_t nbytes)    // memcmp替代

**地址转换函数**

不建议使用的函数：

```C
#include <arpa/inet.h>
int inet_aton(const char *strptr, struct in_addr *addrptr) //将C字符串strptr转换为32位网络字节序的二进制
    char *inet_ntoa(struct in_addr inaddr); // 将32位网络字节序二进制地址转换为点分十进制，不可重入。注意该函数直接以结构为参数
```

建议使用的函数：

```C
#include <arpa/inet.h>
int net_pton(int family, const char *strptr, void *addrptr);
const char *inet_ntop(int family, const void *addrptr, char *strptr, int len);
```

P表示表达式，n表示数值

inet_pton函数成功返回1，表达式无效返回0，出错返回-1

inet_ntop函数成功返回指向结果的指针，出错则返回NULL

#### readn, writen, readline函数

对于TCP，字节流套接字上调用read或write输入或输出的字节数可能比请求的数量少，这并非出错，原因是内核中用于套接字的缓冲区可能已达到极限。此时所需的是调用者再次调用read或write函数，以输入或输出剩余的字节









