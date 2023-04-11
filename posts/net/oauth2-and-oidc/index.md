# OAuth 2.0 与 OIDC


## 1 OAuth 2.0

OAuth 2.0 是一个标准的 Authz 协议（没有 Authn 功能）。如果期望将 Resource 授权给第三方应用，那么就可以使用 OAuth 2.0。

OAuth 2.0 协议中主要有四个角色：

- Resource Owner - 能够授权 Client 访问资源的角色。最典型的就是 User。
- Resource Server - 存放 Resource 的服务器，能够验证 Access Token 来决定是否返回 Resource
- Authz Sever - 对 Resource Owner 进行认证，并负责颁发 Access Token。
- Client - 第三方应用，得到 Resource Owner 授权后，请求访问 Resource

其他一些重要概念：

- Grant - Resource Owner 授予 Client 凭证，用它来访问 Authz Server
- Access Token - 由 Authz Server 颁发的 Token，通过 Token 来访问 Resource Server
- Refresh Token（可选）- 用于 Access Token 过期后获取一个新的 Access Token。

可以看到，Authz Server 集中了鉴权的功能。通过给 Client 颁发 Token，使得 Client 在不知道账户密码的情况下，能够访问 Resource Server 获取资源。

## 2 OIDC

OIDC 是基于 OAuth 2.0 的 Authn 协议，增加了 ID Token 的概念。OIDC 也指定了 OAuth 2.0 中未定义的部分，包括：scope、服务发现、用户信息字段等。

OIDC 中的角色如下：

- End User - 终端用户，对应于 OAuth 2.0 中的 Resource Owner，ID Token 中会包含 End User 相关的信息
- Relying Party - 第三方应用，对应于 OAuth 2.0 中的 Client
- OpenID Provider - Auth Server，负责签发 ID Token 与 Access Token
- Resource Server

其他一些重要概念：

- ID Token - 由 OpenID Provider 颁发的 Token，包含了 EU 的身份信息
- Claim - ID Token 中 EU 信息相关的字段
- UserInfo Endpoint - OpenID Provider 提供的获取 EU 更多信息的接口

OIDC 提供的 Authz 流程与 OAuth 2.0 一样，主要区别在于 OIDC 在 Authz 过程中会额外返回 ID Token。

{{< admonition note Note>}}
可以这样理解，OIDC 与 OAuth 2.0 的流程基本相同，只是 OIDC 基于 ID Token 提供了更多额外功能。
{{< /admonition >}}

## 3 Authz 流程

### 3.1 基本流程

最基本的访问流程如下：

{{< image src="img1.png" height=300 >}}

1. App 请求 Resource Owner，获取 Grant。
2. App 携带 Grant 访问 Authz Server，获取 Access Token 与 ID Token。
3. App 携带 Token 访问 Resource Server，获取 Resource。

基于三个步骤扩展出来，OAuth 2.0 提出了四种授权方式。

### 3.2 Authz Code 授权码模式

Authz Code 是最严格但最常见的方式，考虑了几乎所有的敏感信息泄漏的预防和后果。对于基本流程的第 1 步以及第 2 步都有了扩展。

- App 往往是一个后端服务，必须提供回调 URL（预先提供给 Authz Server）。通过访问 URL 能够将 Authz Code 传递给 App，然后 App 自己存储 Authz Code。
- Resource Owner 需要手动登陆 Authz Server 的登录页面。

{{< image src="img2.png" height=400 >}}

1. 获取 Grant（Authz Code）
   
    1. App 发起授权请求
    2. Authz Server 将请求重定向到授权页面
    3. Resource Owner 手动登陆 Authz Server 的授权页面，表示允许 Grant。
    4. Authz Server 返回的 URL 和 Authz Code，浏览器通过重定向访问 URL，也就是访问后端服务，将 Authz Code 传输给 App。
   
2. 获取 Access Token
   
    1. Client 拿到 Authz Code 后，就能够使用 Authz Code 访问 Authz Server，这样来申请 Access Token 与 ID Token。
        
        如果需要，还会返回 Refresh Token。
        
3. 通过 Token 访问 Resource Server

### 3.3 Implicit 隐式授权模式

Implicit 省略了 Authz Server 颁发 Authz Code 的步骤，并且明确禁止发放 Refresh Token。该模式适用于 App 无法存储 Authz Code 的情况。

{{< image src="img3.png" height=300 >}}

App 发起请求时，会指定接收 Token 的 URL。完成 Grant 后，Authz Server 会回调 URL 来提供 Token。

{{< admonition note Note>}}
与 Authz Code 最大的区别是，没有 Authz Code，而是直接返回 Token。
{{< /admonition >}}

### 3.4 Resource Owner Password Credentials 密码模式

Resource Owner Password Credentials 整合了 Authn 与 Authz 的过程。Resource Owner 直接提供账号密码给 App，App 使用账号密码去 Authz Server 获取 Token。

{{< image src="img4.png" height=250 >}}

因为 Client 直接拿着密码，所以这种模式适用于 Client 高度可信的情况下。

### 3.5 Client Credentials 客户端模式

Client Credentials 指的是：App 直接使用 Resource Owner 提供的 Credentials，向 Authz Server 申请 Token。

该模式常用于 CICD 与服务间的认证授权（M2M 授权）。

{{< image src="img5.png" height=250 >}}

{{< admonition note Note>}}
Client Credentials 因为没有获取 Grant 的步骤，因此也不支持 Refresh Token。
{{< /admonition >}}

### 3.6 Hybrid 混合模式

完成 Grant 后，Auth Server 同时返回 Token 与 Authz Code。App 可以直接使用 Token，后续可以使用 Authz Code 重新获取 Token。

## 4 Token

### 4.1 Access Token

通常 Access Token 是 JWT Base64 编码。其中包含了常见的鉴权使用的字段：

```json
{
  "jti": "3YByyfvL0xoMP5wpMulgL",
  "sub": "60194296801dc7bc2a1b2735",
  "iat": 1612444871,
  "exp": 1613654471,
  "iss": "https://steam-talk.authing.cn/oidc",
  "aud": "60193c610f9117e7cb049159",
  "scope": "openid email message"  // 细节控制权限范围
}
```

### 4.2 ID Token

OIDC 对 OAuth 2.0 最主要的扩展就是 ID Token。ID Token 相当于用户的身份凭证。应用可以通过校验 ID Token 以确定用户身份。

ID Token 本质上是 `JWT Token` ，包含用户信息的 K/V 键值对。

```json
{
   "iss": "https://server.example.com",  // identity provider 标识，往往是用于验证身份的 URL 
   "sub": "24400320",                    // 用户的唯一标识信息
   "aud": "s6BhdRkqt3",                  // 表示 ID Token 受众，也就是哪些服务能够处理 ID Token
   "nonce": "n-0S6_WzA2Mj",              // 随机值，用于防止重放攻击
   "exp": 1311281970,                    // Token 过期时间
   "iat": 1311280970,                    // Token 颁发时间
   "auth_time": 1311280969,              // 用户进行身份认证的时间
   "acr": "urn:mace:incommon:iap:silver" // 用户认证的上下文引用
}
```

可以看到，ID Token 通过 JWT Token 直接承载了用户信息。但是因为 JWT Token 无法承载太多信息，因此协议定义了 UserInfo Endpoint 接口来用于获取更多的用户信息。

当 RP 收到 OP 提供的 ID Token 与 Access Token 后，可以调用 UserInfo Endpoint 获取更多用户信息。

### 4.3 Refresh Token

Access Token 以及 ID Token 的有效期比较短。失效后，Client 需要重新获取 Grant，然后重新去 Authz Server 获取一个新的 Access Token。

为了让 Client 跳过获取 Grant 的流程，Authz Server 提供 Access Token 同时也会提供 Refresh Token。

Client 携带 Refresh Token 能够重新从 Authz Server 获取一个 Access Token，而不需要再次获取 Grant。

### 4.4 如何验证 Token

当 Resource Server 收到 Access/ID Token，通过 JWT 的 `Signature` 信息来验证 Token 是否合法。

- 例如使用 RS256 算法（非对称加密），Auth Server 会使用 Private Key 对 `Header` 与 `Payload` 进行加密。应用收到 Token 后就可以使用 Public Key 来解密验证 `Signature` 。
- 例如使用 HS256 算法（对称加密），Auth Server 会使用 Secret 对 `Header` 与 `Payload` 进行加密。应用需要先从 Auth Server 获取 Secret，然后收到 Token 后解密来验证（或者调用 Auth Server 接口验证）。

### 4.5 传递 Token

RFC 6750 定义了 Client 如何将 Access Token 传递给 Resource Server。

- **URI Query Parameter** - 基于 Param 传递
    
    ```http
    GET /resource?access_token=mF_9.B5f-4.1JqM HTTP/1.1
    Host: server.example.com
    ```
    
    > 注意添加 `Cache-Control:no-store` 防止中间件缓存。
    > 

- **HTTP Header** - 基于 HTTP Header `Authorization`
    
    ```http
    GET /resource HTTP/1.1
    Host: server.example.com
    Authorization: Bearer mF_9.B5f-4.1JqM
    ```
    
    最常见的方式，其中 `Bearer` 是固定的前缀。
    
- HTTP Body - 基于 HTTP Body
    
    ```http
    POST /resource HTTP/1.1
    Host: server.example.com
    Content-Type: application/x-www-form-urlencoded
    
    access_token=mF_9.B5f-4.1JqM
    ```
    
    不允许使用 `Get` 方式，并且必须设置 `Content-Type: application/x-www-form-urlencoded` 。

## 5 Federation Auth

Auth 流程中描述的都是 Authz Server 负责了 Authn 功能，来颁发 ID Token。一些场景下，可能 App 想复用互联网产品的账号来进行身份认证，例如网站支持使用 QQ 登陆。这种情况下，可以使用 Federation Auth。

{{< image src="img7.png" height=350 >}}

通常，Identity Provider 分为三类：

- Social Identity Provider - Identity Provider 为第三方社交平台，例如微信、QQ、Google 等；
- Enterprise Identity Provider - Identity Provider 为办公应用（例如飞书、企业微信）或标准协议应用（例如 OIDC、SAML、CAS 等标准协议）；
- Database Identity Provider - 提供给 Authz Server 一个数据库，用于存放身份信息；
