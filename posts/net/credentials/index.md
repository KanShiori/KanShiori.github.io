# 凭证机制 Credential


在授权过程中，基本每种授权模式目的就是为了得到一个令牌，令牌就是 [凭证]^(Credential)。目前主流有两种令牌实现方式：Cookie-Session 和 JWT。

## 1 Cookie-Session

HTTP 是无状态协议，实现了简单的 Web 资源交换功能。但是一些情况下也需要一些手段，让服务器接收 HTTP 请求时能够得用户知相关的上下文。为此，RFC 6265 定义了 HTTP 的状态管理机制，在 HTTP 协议中增加了 Set-Cookie 指令。

Set-Cookie 指令的含义是：以键值对的方式向客户端发送一组信息，该信息将在此后每次 HTTP 请求中，以名为 Cookie 的 Header 发送给服务端。服务端就可以根据 Cookie 字段来区分客户端。
```
Set-Cookie: id=icyfenix; Expires=Wed, 21 Feb 2020 07:28:00 GMT; Secure; HttpOnly
```

收到该指令后，客户端再次访问时会带有 Cookie Header，例如：
```
GET /index.html HTTP/2.0
Host: icyfenix.cn
Cookie: id=icyfenix
```

这样，每次请求传到服务器的 Cookie，服务端就能够识别出是哪个用户。因为 Cookie 是 HTTP Header，所以通常 Cookie 的值为一个 session id。服务端根据 session id 在保存的信息中检索出用户对应的上下文信息。再加上一些超时自动清理的管理措施。

这种服务端的状态机制就称为 Session。Cookie-Session 就是目前最传统也最常用的客户端服务端状态管理机制。

优点：
* 安全性优势，因为状态信息都存储于服务端；
* 服务端管理状态信息，能够实现灵活的管理；

缺点：
* 由于 Session 存储在服务器内存中，Cookie-Session 当遇到多节点环境时，各个服务器之间的状态信息同步就是个问题，类似于分布式的状态问题；

通常，Cookie-Session 遇到多节点的解决方案有：
* 负载均衡 - 每个特定用户请求分配到特定的某个服务器；
* 状态同步 - 每个服务器将 Session 变动同步到其他所有服务器；
* 单点 - 将 Session 集中放在一个单点服务器，所有服务器无状态，访问单点 Session 服务器读取状态；

## 2 JWT

与 Cookie-Session 保存状态信息的思路相反，当服务器存在多个时，一个状态信息对应客户端只有一个，可以把状态信息保存在客户端，每次随着请求发送给服务端。但是，这个思路的缺点是无法携带大量信息，而且有着泄漏和篡改的风险。

JWT（JSON Web Token）是目前广泛使用的令牌格式，能够确保传输信息不被中间人篡改。

{{< admonition note Note>}}
JWT 默认是不加密的，因为仅仅解决防止篡改问题，不解决泄漏问题。
{{< /admonition >}}

JWT 最常见的使用方式是作为名为 Authorization 的 Header 发送服务端。其值前缀定义为 Bearer，类似于：
```HTTP
GET /restful/products/1 HTTP/1.1
Host: icyfenix.cn
Connection: keep-alive
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJpY3lmZW5peCIsInNjb3BlIjpbIkFMTCJdLCJleHAiOjE1ODQ5NDg5NDcsImF1dGhvcml0aWVzIjpbIlJPTEVfVVNFUiIsIlJPTEVfQURNSU4iXSwianRpIjoiOWQ3NzU4NmEtM2Y0Zi00Y2JiLTk5MjQtZmUyZjc3ZGZhMzNkIiwiY2xpZW50X2lkIjoiYm9va3N0b3JlX2Zyb250ZW5kIiwidXNlcm5hbWUiOiJpY3lmZW5peCJ9.539WMzbjv63wBtx4ytYYw_Fo1ECG_9vsgAn8bheflL8
```

### 2.1 JWT 的格式

JWT 传输时是以 Base64 编码后传输，其原文格式以 JSON 格式。总结划分为三个部分，每个部分间用点号 “.” 分隔。
{{< find_img "img1.png" >}}

第一部分是 Header：
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
* alg - 签名的算法
* typ - 令牌类型

第二部分是 Payload，即令牌真正传输的信息，用户随意定制。
```json
{
  "username": "icyfenix",
  "authorities": [
    "ROLE_USER",
    "ROLE_ADMIN"
  ],
  "scope": [
    "ALL"
  ],
  "exp": 1584948947,
  "jti": "9d77586a-3f4f-4cbb-9924-fe2f77dfa33d",
  "client_id": "bookstore_frontend"
}
```

JWT RFC 中推荐了七项内容：
* iss（Issuer）- 签发人；
* exp（Expiration Time）- 令牌过期时间；
* sub（Subject）- 主题；
* aud（Audience）- 令牌受众；
* nbf（Not Before）- 令牌生效时间；
* iat（Issued At）- 令牌签发时间；
* jti（JWT ID）- 令牌 ID；

第三部分是 Signature，通过 Header 中定义算法，通过特定密钥（服务器保管）对前面两部分进行加密计算。服务端在接收到 JWT 后，只要内容被篡改过，其签名就无法通过检查。

优点：
* 服务端无状态

缺点：
* 令牌难以主动失效。令牌签发后，理论上和认证服务器没有关联了，难以主动让某个令牌失效。通常，会使用 Session 机制在服务端维护一个黑名单，过滤失效的令牌。
* 容易收到重放攻击。
* 只能携带优先的数据。
* 必须考虑令牌在客户端如何存储。
