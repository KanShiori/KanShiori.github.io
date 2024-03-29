# 认证机制 Authentication


> 文章内容来自于[**《凤凰架构》**](https://github.com/fenixsoft/awesome-fenix)，推荐阅读。

[认证]^(Authentication) 用来解决系统对用户身份认证的问题，例如用户密码就是最简单的认证机制。用户不仅仅是人，也可能指外部的代码。

从网络资源传输流程上，有着主要三个认证方式：

* 通信信道上的认证：建立网络连接时，进行身份的认证。典型是基于 SSL/TLS 的传输层认证。
* 通信协议上的认证：获取资源时，由协议进行身份的认证。典型是基于 HTTP 协议的认证。
* 通信内容上的认证：获取资源时，由业务进行身份认证。典型是基于 Web 内容的认证。

## 1 传输层的认证

### 1.1 摘要、加密与签名

[摘要]^(digest) 通过哈希算法的易变性与不可逆性，用于判断数据的真伪。
* 易变性 - 如果输入端发生任何一点变动，哈希结果都会产生极大变化。通过对比摘要，可以判断收到的数据与源数据是否是一致的。
* 不可逆行 - 从摘要无法还原出源数据，保证公开的摘要不会泄漏源数据。

[加密]^(encryption) 与摘要的本质区别是，加密是可逆的，通过解密可以得到原数据。根据加密与解密是否使用同一个密钥，可以分为 对称加密算法 和 非对称加密算法。

* 对称加密算法 - 加密和解密时使用相同的密钥
  
  性能比非对称加密高，但是面临着密钥传输难题（无法保证密钥在传输过程中不泄露）。

* 非对称加密算法 - 密钥分为公钥与私钥，私钥由用户保管，公钥可以传输给对端。公钥加密数据，只有私钥能解密。私钥加密数据，只有公钥能解密。
  
  因为加解密使用不同密钥，公钥就可以通过不安全的信道传输，解决了密钥传输难题。但是，非对称加密性能比对称加密低几个数量级。

目前在加密方面，现在一版本会结合对称与非对称加密的优点：用非对称加密传输对称加密的密钥，然后采用对称加密来进行通信。

因为非对称加密的加密算法，可以提供两种功能：

* 公钥加密，私钥解密 - 这就是加密。A 将公钥传输给 B，B 使用公钥加密数据传输给 A，A 使用私钥解密。整个过程是即使公钥或者加密数据泄漏，通信还是安全的。
* 私钥加密，公钥解密 - 这就是签名。因为私钥加密后，只有公钥可以解密，那么公钥持有者就可以通过解密来验证私钥是否正确。

[签名]^(signature) 就是类似于客户端通过证书里的公钥，来检查服务端的私钥是否正确。

### 1.2 数字证书

前面看到，“签名” 的方法是让公钥持有者验证私钥持有者的身份，而关键就在于公钥持有者保存的公钥是否是受信任的。

* 内部服务，公钥可以来自于自签名，那么公钥肯定是受信任的，因为是开发者自行保存的。
* 公共服务，公钥需要一个权威的公证人进行一次签名，例如证书机制中的 CA 存在。所有的用户都信任公证人，那么就可以认为公证人签名后的公钥是可以信任的。

第二种方式引出了公钥可信分发的标准，称为 [公开密钥基础设置]^(Public Key Infrastructure)，简称 PKI。其中，数字证书认证中心 CA 就是权威的机构，可以签发证书。

[证书]^(Certificate) 是权威 CA 对特定公钥的签名证书，可以理解 CA 将某个公钥使用了 CA 的私钥进行加密。

在认证时，用户使用内置在机器的根证书对证书进行认证，本质上就是使用根证书中的公钥解密证书。认证（解密）后，就可以认为证书是合法的了，而后续就可以使用证书中的公钥进行通信了。

{{< admonition note Note>}}
可以看到，证书就是 CA 对公钥签名，而 CA 的证书（公钥）内置在用户机器上。
{{< /admonition >}}

PKI 中的证书格式是 X.509 格式，除了公钥内容外还包含了用户信息：

* 版本号 Version - 证书使用哪个版本的 X.509 标准，目前版本为 3。
  
  ```
  Version: 3 (0x2)
  ``` 

* 序列号 Serial Number - 给证书分配的唯一标识符。
  
  ```
  Serial Number: 04:00:00:00:00:01:15:4b:5a:c3:94
  ```

* 签名算法标识符 Signature Algorithm ID - 签名证书算法标识，包含非对称加密算法与摘要算法。
  
  ```
  Signature Algorithm: sha1WithRSAEncryption
  ```

* 认证机构的数字签名 Certificate Signature - 证书颁发者使用私钥对证书的签名。

* 认证机构 Issuer Name - 证书颁发者的名字。
  
  ```
  Issuer: C=BE, O=GlobalSign nv-sa, CN=GlobalSign Organization Validation CA - SHA256 - G2
  ```

* 有效期限 Validity Period - 证书起始日期和时间，以及终止日期和时间。
  
  ```
  Validity
	Not Before: Nov 21 08:00:00 2020 GMT
	Not After : Nov 22 07:59:59 2021 GMT
  ```

* 主题信息 Subject - 证书持有人的唯一标识符。
  
  ```
  Subject: C=CN, ST=GuangDong, L=Zhuhai, O=Awosome-Fenix, CN=*.icyfenix.cn
  ```

* 公钥信息 Public Key - 证书持有者的公钥，算法标识符和其他密钥参数。


## 2 HTTP 认证

RFC 7235 中定义了 HTTP 协议的通用认证框架：在未授权用户访问保护区域的资源时，服务器应返回 401 Unauthorized 的状态码，同时在回复 Header 中附带以下两个 Header 之一，告知客户端采取何种方式进行认证。

* `WWW-Authenticate: <认证方案> realm=<保护区域描述信息>`

* `Proxy-Authenticate: <认证方案> realm=<保护区域描述信息>`

客户端收到回复后，遵循服务器指定的认证方案，在请求的 Header 中加入身份凭证信息。服务端验证通过后返回资源，否则将返回 403 Forbidden 错误。请求 Header 中包含以下 Header 之一：

* `Authorization: <认证方案> <凭证内容>`
* `Proxy-Authorization: <认证方案> <凭证内容>`

{{< image src="img1.png" height=350 >}}

以简单的家有路由器为例，登陆路由器时，浏览器会收到服务端基于 HTTP Basic 认证的回复：

```HTTP
HTTP/1.1 401 Unauthorized
Date: Mon, 24 Feb 2020 16:50:53 GMT
WWW-Authenticate: Basic realm="example from icyfenix.cn"
```

接着浏览器会弹出 HTTP Basic 认证对话框，要求用户输入账户与密码。用户输入后，浏览器将信息编码然后发送给服务端：
```HTTP
GET /admin HTTP/1.1
Authorization: Basic aWN5ZmVuaXg6MTIzNDU2
```

服务端收到请求后，检查用户账户密码是否正确。

除了 Basic 认证外，还定义了许多用于实际生产环境的认证方案，例如：
* Digest - RFC 7616，HTTP 摘要认证，视为 Basic 认证的改良版本。
* Bearer - RFC 6750，基于 OAuth2 来完成认证。
* HOBA - RFC 7486，一种基于自签名证书的认证方案。

HTTP 认证框架提供了基本的框架，允许自行扩展，并不要求使用 RFC 规范来定义，只要客户端与服务端能够识别并处理私有的认证方案即可。


## 3 Web 认证

对于非浏览器的用户认证场景中，通常是希望后端系统来实现认证方式，而不是直接使用 HTTP 协议的认证方案。这些依靠内容而不是传输协议来实现的认证方式，统称为 “Web 认证”。

Web 认证没有什么标准，直到 2019 年 3 月，WebAuthn 出现。WebAuthn 不采用传统的密码登陆，而是直接采用生物识别（指纹、人脸、虹膜、声纹）或者实体密钥（USB、蓝牙、NFC 连接的物理密钥容器）作为身份凭证。
