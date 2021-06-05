### HTTP首部字段结构

#### HTTP首部字段重复的场景

规范没有明确，有些浏览器会优先处理第一次出现的首部字段。有些会优先处理最后出现的首部字段。

#### 4种HTTP首部字段类型

| 类型         | 说明                                  |
| ------------ | ------------------------------------- |
| 通用首部字段 | 请求和响应报文两方都会使用的首部      |
| 请求首部字段 |                                       |
| 响应首部字段 |                                       |
| 实体首部字段 | 针对请求/响应报文的实体部分使用的首部 |

#### 1. 通用首部字段

##### no-cache指令

```
Cache-Control: no-cache
```

使用no-cache指令的目的是为了防止从缓存中返回过期的资源

* 如客户端发送的请求中包含no-cache指令，表示客户端不会接收缓存过的相应。于是“中间”的缓存服务器必须把客户端请求转发给源服务器。
* 如服务器返回的响应中包含no-cache指令，那么缓存服务器不能对资源进行缓存

##### max-age指令

```
Cache-Control: max-age=604800
```

* 如客户端发送请求中包含max-age指令，且缓存资源的缓存时间比指定值校，就接收缓存的资源。另外，当指定max-age值为0，那么缓存服务器通常需要将请求转发给源服务器。
* HTTP/1.1版本的缓存服务器遇到同时存在Expires首部字段的情况时，会优先处理max-age指令，忽略掉Expires首部字段。HTTP/1.0恰好相反

##### min-fresh指令

```
Cache-Control: min-fresh=60(单位：秒)
```

min-fresh指令要求缓存服务器返回至少还未过指定时间的缓存资源。

也就是说，当min-fresh为60秒后，过了60秒的资源都无法作为响应返回。

##### max-stale 指令

```
Cache-Control: max-stale=3600(单位：秒)
```

使用max-stale可指示缓存资源，即使过期也照常接收。

如果指定了具体值，那么即使过期，只要仍处于max-stale指定时间内，仍旧会被客户端接收。

#### Connection

Connection首部字段的两个作用：

* 控制不再转发给代理的首部字段
* 管理持久连接

```
Connection: 不再转发的首部字段名
```

![Connection](C:\Users\pc\Desktop\learning\notes\notes\NetWork\images\Connection.PNG)

HTTP/1.1版本的默认连接都是持久链接。当服务器端想明确断开连接时，则指定Connection首部字段的值为Close； 旧版本上维持持续链接，需指定Connection首部字段的值为Keep-Alive

#### Transfer-Encoding字段

规定了传输报文主体时采用的编码方式， HTTP/1.1的传输编码方式仅对分块传输编码有效。

#### Via字段

使用via字段追踪客户端和服务器之间的请求和响应报文的传输路径。

报文经过代理或网关时，会先在首部字段VIA中附加该服务器的信息，然后再进行转发。

首部字段via不仅用于追踪报文的转发，还可避免请求回环的发生。

#### 2. 请求首部字段

##### accept字段

* 文本文件 text/html, text/plain, text/css, ....
* 图片文件 image/jpeg, image/gif, image/png
* 视频文件 video/mpeg, video/quicktime
* 应用程序使用的二进制文件 application/octet-stream, application/zip

当服务器提供多种内容时，将会首先返回权重值最高的媒体类型

##### accept-charset字段

可用于通知服务器用户代理支持的字符集及字符集的相对优先顺序

```
Accept-Charset: iso-8859-5, unicode-1-1;q=0.8
```

##### accept-encoding字段

用于告知服务器用户代理支持的内容编码及内容编码的优先级顺序，可一次性指定多种内容编码	

##### host字段

```
Host: www.hackr.jp
```

当虚拟主机运行在同一个IP上，需要用Host字段加以区分。

##### If-Match字段

![If-Match](C:\Users\pc\Desktop\learning\notes\notes\NetWork\images\If-Match.PNG)

首部字段If-Match，会告知服务器匹配资源所用的实体标记ETag的值，服务器会比对If-Match的字段值和资源的ETAG值，仅当两者一致，才会执行请求；反之，返回状态码 412 Precondition Failed响应

### Cookie服务的首部字段

| 首部字段名 | 说明                           | 首部类型     |
| ---------- | ------------------------------ | ------------ |
| Set-Cookie | 开始状态管理所使用的Cookie信息 | 响应首部字段 |
| Cookie     | 服务器接收到的Cookie信息       | 请求首部字段 |

```
Set-Cookie: status=enable; expires=Tue, 05 Jul 2011 07:26:31
```

Cookie的expires属性指定浏览器可发送Cookie的有效期。

当省略expires属性时，其有效期仅限于维持浏览器会话时间段内，通常限于浏览器应用被关之前

服务端通过覆盖已过期的Cookie，实现对客户端Cookie的实质性删除操作。

Cookie的HttpOnly属性是Cookie的扩展功能，它使JS脚本无法获得Cookie，防止跨站脚本攻击，对Cookie的信息窃取



