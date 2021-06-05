## c2 简单的HTTP协议

### HTTP/1.1

* **GET：**获取资源，资源可以是文本或CGI(Common Gateway Interface)

| GET /index.html HTTP/1.1 Host: www.hackr.jp | 请求 |
| ------------------------------------------- | ---- |
| 返回index.html的页面资源                    | 相应 |

* **POST：**传输实体主体，功能与GET类似

| 请求 | POST /submit.cgi HTTP/1.1 Host: www.hackr.jp Content-Length:1560 |
| ---- | ------------------------------------------------------------ |
| 响应 | 返回submit.cgi接收的数据和处理结果                           |

* **PUT:** 传输文件

| 请求 | PUT /example.html HTTP/1.1 Host: www.hacker.jp Content-type:text/html Content-Lengtjh:1560 |
| ---- | ------------------------------------------------------------ |
| 响应 | 返回204 No Content                                           |

* **HEAD：**获得报文首部

与GET方法类似，只是不返回报文主体部分。用于确认URI有效性及资源更新的日期时间

| 请求 | HEAD /index.html HTTP/1.1 Host: www.hacker.jp |
| ---- | --------------------------------------------- |
| 响应 | 返回index.html有关的响应首部                  |

* **DELETE：** 删除文件

DELETE方法用来删除文件，与PUT相反

| 请求 | DELETE /example.html HTTP/1.1 |
| ---- | ----------------------------- |
| 响应 | 响应返回状态码 204 No Content |

* **OPTIONS:** 询问支持的方法

查询针对请求URI指定的资源支持方法

| 请求 | OPTIONS * HTTP/1.1                            |
| ---- | --------------------------------------------- |
| 响应 | HTTP/1.1 200 OK Allow: GET, POST,HEAD,OPTIONS |

* **CONNECT: ** 要求用隧道协议连接代理

CONNECT方法要求在与代理服务器通信时建立隧道，实现用隧道协议进行TCP通信。主要使用SSL(Secure Sockets Layer，安全套接字)和TLS(Transport Layer Security, 传输层安全)协议，把通信内容加密后经网络隧道传输

```
CONNECT 代理服务器名:端口号 HTTP版本
```

CONNECT方法请求、响应的例子

| 请求 | CONNECT proxy.hackr.jp:8080 HTTP/1.1 |
| ---- | ------------------------------------ |
| 响应 | HTTP/1.1 200 OK                      |

#### HTTP/1.0 和 HTTP/1.1 支持的方法

| 方法 | 说明 | 支持的HTTP协议版本 |
| ---- | ---- | ------------------ |
|  GET    | 获取资源     | 1.0, 1.1                   |
|  POST    | 传输实体主体     | 1.0, 1.1                   |
|  PUT    | 传输文件     |   1.0, 1.1                 |
|  HEAD    | 获得报文首部     |  1.0, 1.1                  |
|  DELETE    |  删除文件    |   1.0, 1.1                 |
|  OPTIONS    | 询问支持的方法     |  1.1                  |
|  TRACE    |  追踪路径    |   1.1                 |
|  CONNECT    |  要求用隧道协议连接代理    |  1.1                  |
|  LINK    | 建立和资源之间的联系     |  1.0                  |
|  UNLINE    | 断开连接关系     | 1.0                   |

#### 持久连接 (HTTP Presistent Connections)

* 也称为(HTTP keep-alive 或 HTTP connection reuse),  特点是只要任意一端没有明确提出断开链接,则保持TCP连接状态

* HTTP/1.1中, 所有的连接默认都是持久连接, 但HTTP/1.0未标准化

#### 管线化

无需等待就能直接发送下一个请求. 例如请求一个包含10张图片的HTML Web页面

#### 使用Cookie的状态管理

让服务器管理全部客户端状态会成为负担, 引入cookie计数

![Cookie](C:\Users\pc\Desktop\learning\notes\notes\NetWork\images\Cookie.PNG)



































