## HTTPS

#### HTTP的缺点

* 通信使用明文，内容可能被窃听
* 不验证通信方身份，可能遭遇伪装
* 无法证明报文的完整性，可能已遭篡改

#### HTTPS( HTTP Secure)

* 通信的加密

通过和SSL(Secure Socket Layer，安全套接层)或TLS(Transport Layer Security, 安全层传输协议)的组合使用，加密HTTP的通信内容。

#### 加密技术

SSL采用一种叫做**公开密钥加密**(Public-key cryptography)的方式，被叫做对称密钥加密

* 共享密钥加密 —— 加密和解密同用一个密钥的方式

* 公开密钥加密 —— 使用一对非对称的密钥，一把私有密钥，一把公有密钥

#### 为什么不一直使用HTTPS ?

* 加密通信消耗资源，导致处理请求能力下降
* 节约购买证书的开源

