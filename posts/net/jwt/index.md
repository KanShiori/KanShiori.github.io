# JSON Web Token


**JWT** 是目前广泛使用的令牌格式，**能够确保传输信息不被中间人篡改。**

JWT 一般用于 Identity Provider 与 Service 之间传递认证后的用户信息。

## 1 基于 Session 的认证

HTTP 是无状态的，为了让 Server 接收 HTTP 请求时知晓相关上下文，RFC 在 HTTP 协议中增加了 `Set-Cooike` 指令。

`Set-Cooike` 指令的含义是：Server 以 K/V 方式向 Client 发送一组信息。后续 Client 在 HTTP 请求中将信息放在 `Cooike` 的 Header 中，Server 可以根据 `Cookie` 来区分 Client 的身份。

{{< image src="img0.png" height=400 >}}

1. Client 向 Server 第一次发起请求，Server 为其创建了一个 Session。
2. Server 给 Client 的回复中，通过 `Set-Cookie` 提供给了 Client 它的 Session ID，以及超时信息。
    
    ```http
    Set-Cookie: id=1678; Expires=Wed, 21 Feb 2020 07:28:00 GMT; Secure; HttpOnly
    ```
    
3. Client 再次访问 Server 时，在 `Cookie` Header 带上了 Session ID。
    
    ```http
    GET /index.html HTTP/2.0
    Cookie: id=1678
    ```
    
4. Server 根据读到的 Session ID，从 DB 中读取用户信息。

## 2 基于 JWT 的认证

JWT 传输时以 `Base64` 编码传输，原文是 JSON 格式。总共分为三个部分，每个部分都过 `.` 号分隔。

{{< image src="img1.png" height=400 >}}

包含：

- `Header` - 表明 JWT 使用的协议信息
    
    ```json
    {
      "alg": "RS256",  // 签名的算法
      "typ": "JWT"     // Token 类型
    }
    ```
    
- `Payload` - Token 传输的内容
    
    ```json
    {
    	// well known fields (optional)
      "iss": "https://dxxxx.auth0.com/",  // issuer - 签发者     
      "sub": "xxxx",                      // subject - Token 请求者 
      "aud": "/api/order",                // audience - Token 受众
      "iat": 1680703634,                  // issued at - Token 签发时间
      "exp": 1680790034,                  // expired at - Token 过期时间
    	"nbf": 1680703634,                  // not before at - Token 生效时间
      "jti": "xxxx",                      // jwt token id - Token ID
    	// customized fields
      "permissions": []
    }
    ```
    
- `Signature` - Token 签名
    
    通过 `Header` 中定义的算法，通过 Key 或 Secret（Auth Server 保管）对 `Header` 与 `Payload` 进行签名。
    
    例如使用 RS256 算法（非对称加密），Auth Server 会使用 Private Key 对 `Header` 与 `Payload` 进行加密。应用收到 Token 后就可以使用 Public Key 来解密验证 `Signature` 。
    
    例如使用 HS256 算法（对称加密），Auth Server 会使用 Secret 对 `Header` 与 `Payload` 进行加密。应用需要先从 Auth Server 获取 Secret，然后收到 Token 后解密来验证（或者调用 Auth Server 接口验证）。
    

## 2 认证流程

使用 JWT Token 的一般认证流程如下：

{{< image src="img3.png" height=300 >}}

1. User（或者 Client） 携带 Credentials 来请求 Auth Server，以获取一个 Access Token（JWT 格式）。
    
    这个 Credentials 可能是多种方式：
    
    - 账号密码
    - 邮箱验证码
    - 等

2. Auth Server 验证 Credentials 是否正确，成功后返回一个 Access Token。
3. Client 每次访问 Resource 时，都会携带 Access Token。
4. Resource Server 验证 Access Token，成功后返回 Resource。

### 2.1 Client 携带 JWT Token 的方式

通常，Client 会在 HTTP Header **`Authorization`** 包含 JWT Token 内容，其前缀为 `Bearer` 。

```http
GET /restful/products/1 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJpY3lmZW5peCIsInNjb3BlIjpbIkFMTCJdLCJleHAiOjE1ODQ5NDg5NDcsImF1dGhvcml0aWVzIjpbIlJPTEVfVVNFUiIsIlJPTEVfQURNSU4iXSwianRpIjoiOWQ3NzU4NmEtM2Y0Zi00Y2JiLTk5MjQtZmUyZjc3ZGZhMzNkIiwiY2xpZW50X2lkIjoiYm9va3N0b3JlX2Zyb250ZW5kIiwidXNlcm5hbWUiOiJpY3lmZW5peCJ9.539WMzbjv63wBtx4ytYYw_Fo1ECG_9vsgAn8bheflL8
```

### 2.2 如何验证 JWT Token

以使用 RS256 算法为例，Auth Server 会使用 Private Key 来将 JWT Token 内容签名，也就是 JWT Token 中的 **`Signature`** 字段。

在本地收到 JWT Token 后，我们就可以使用预先配置的 Public Key 来尝试解密 **`Signature`** ，成功解密旧表明认证成功。

{{< admonition note Note>}}
例如对于 Auth0，可以请求 `https://<your_domain>/.well-known/jwks.json` 来获取 Public Key。
{{< /admonition >}}

## 3 Session 对比 JWT

| Session                                        | JWT                                                |
| ---------------------------------------------- | -------------------------------------------------- |
| 用户信息存储在 Server 端，安全性高             | 用户信息由 Client 端保存，Server 端无状态          |
| 在 Server 端能保持大量信息，包括敏感信息       | 在 Client 端只能保存少量关键信息，不能包含敏感信息 |
| Server 端多节点时，节点间 Session 同步是个问题 | 信息存在 Client 端，不需要考虑分布式问题           |
| 未提供 Client 任何凭证，因此可以实现回收权限   | Token 签发后就是有效的，并且无法回收               |

Session 遇到多节点问题的常见方案
 - 负载均衡 - 特定用户只分配到特定 Server
 - 状态同步 - 每个 Server 将 Session 变动同步到其他 Server
 - 单点 - Session 集群存放在单点服务器，Server 无状态
