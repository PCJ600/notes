## 名字与地址转换

#### 域名系统

主要用于主机名字与IP地址之间的映射。主机名既可以是一个简单的名字，也可以是一个全限定域名

DNS中的条目称为资源记录(RR)，类型如下：

A记录: 把一个主机名映射成32位IPV4地址

AAAA记录: 把一个主机名映射成128位地址

PTR记录：把IP地址映射成主机名

MX， CNAME(规范名字)

#### 了解客户、解析器、名字服务器之间的典型关系



#### DNS替代方法

不适用DNS也可能获取名字和地址信息，常用替代方法有:

静态主机文件: /etc/hosts

#### gethostbyname函数

````C
#include <netdb.h>
struct hostent *gethostbyname(const char *hostname);//成功返回非空指针，出错为NULL,设h_errno，返回结构体中包含所查找主机的所有IPV4地址
struct hostent {
    char *h_name;	// 所查询主机的规范名字 如www.a.shifen.com
    char **h_aliases; // 主机所有别名 如www.baidu.com
    int h_addrtype;	// 协议类型，如AF_INET
    int h_length;	// 地址长度, 常见值4
    char **h_addr_list;	// 查询主机的多个IPV4地址(需inet_ntop转换为点分十进制)
};
printf("%s\n", hstrerror(h_errno));
````

返回值：发生错误时不设置errno变量，而是将全局变量h_errno设置为在头文件<netdb.h>中定义的常值；

多数解析器提供名为hstrerror的函数，以某一个h_errno值作为唯一参数，返回一个const char* 的指针。

#### gethostbyaddr函数

```C
#include <netdb.h>
struct hostent *gethostbyaddr(const char *addr, socklen_t len, int family);
```

addr参数不是char*类型，是一个指向IPV4地址的某个in_addr结构的指针，len参数是这个结构的大小。

按DNS的说法，gethostbyaddr在in_addr.arpa域中向一个名字服务器查询PTR记录

#### 注意事项

应该用getaddrinfo取代gethostbyname, 前者支持IPV4和IPV6，后者只能返回V4地址

#### getservbyname和getservbyport函数

在程序代码中通过名字而不是端口指代一个服务，将从名字到端口号的映射关系保存在一个文件(/etc/services)，即使端口号发生变动，需修改的仅仅是/etc/services文件中的某一行，不必重新编译

#### getservbyname函数

```C
#include<netdb.h>
struct servent *getservbyname(const char *servname, const char *protoname);// 若成功返回非空指针，失败返回NULL
struct servent {
    char *s_name;	// official service name
    char **s_aliases;
    int s_port;		// **以网络字节序返回**
    char *s_proto;  // 值可以是"tcp", "udp"
}
getservbyname("ftp", "tcp");
```

/etc/services文件 grep -e ^ftp -e ^domain /etc/services

#### getservbyport函数

```C
#include<netdb.h>
struct servent *getservbyport(int port, const char *protoname);
getservbyport(htons(21), "tcp"); // FTP using tcp
```

port必须为网络字节序

有些端口号在TCP上用于一种服务，UDP上用于完全不同的服务(grep 514 /etc/services)

#### getaddrinfo函数

```C
#include<netdb.h>
int getaddrinfo(const char *hostname, const char *service, const struct addrinfo *hints,0 struct addrinfo **result);

// UNP-v1 c16
struct addrinfo {
    int ai_flags;
    int ai_family;
    int ai_socktype;
    int ai_protocol;
    socklen_t ai_addrlen;
    char *ai_canonname;
    struct sockaddr *ai_addr;
    struct addrinfo *ai_next;    
};
```

能够处理名字到地址， 服务到端口这两种转换

hostname参数是主机名或地址串, service参数是一个服务名或十进制端口号数串

hints参数可以是空指针

##### 注意事项

* 如客户清楚自己只处理一种套接字，应吧把hints结构设为SOCK_STREAM或SOCK_DGRAM

* 典型服务器进程只指定service不指定hostname, hints结构中指定AI_PASSIVE

* 如服务器清楚自己只处理一种类型套接字，应把hints结构的ai_socktype成员设置为SOCK_STREAM或SOCK_DGRAM

#### gai_strerror函数

```C
#include<netdb.h>
const char *gai_strerror(int error);
```

#### freeaddrinfo函数

```C
#include<netdb.h>
void freeaddrinfo(struct addrinfo *ai); //ai参数指向getaddrinfo返回的第一个addrinfo结构
```

##### 说明

* 由getaddrinfo返回的所有存储空间是动态获取的，包括addrinfo, ai_addr, ai_canonname，存储空间通过freeaddrinfo返还给系统

* 如果我们为保存其信息仅复制这个addrinfo结构，然后调用freeaddrinfo会出错。(浅复制)

































































































