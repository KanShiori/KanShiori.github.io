# OAuth2 与 OIDC


## 1 Why

OAuth 2 与 OIDC 都是解决一个问题：解决面向 [第三方应用]^(Third-Party Application) 的认证授权问题。

以一个实际的例子来说：CI 需要访问我的 GitHub 去进行打包，而问题就在于 CI 如何访问到我的 GitHub。最简单的方法就是：我将账号密码告知 CI，但是这样显然导致了密码泄露，并且还无法很好的回收权限。

对于这个问题的解决核心思想是，**使用 Token 代替账号密码作为访问凭证**。

使用 Token 的好处在于：

* 哪怕令牌泄露，也不会导致密码泄露；
* 令牌可以设定访问资源的范围，以及时效性；
* 每个应用都有独立的令牌，其中一个泄露不会导致其他应用收到影响；

<important>对于 OAuth 2，Resource Server 通过颁发 Access Token 来授予 Third-Party 访问资源的权限。</important>

<important>对于 OIDC，OpenID Provider 通过颁发 ID Token + Access Token 来授予 Third-Party 获取用户信息的权限。</important>

在围绕着这个基本模式上，针对如何验证 User 身份，如何颁发 Token 引入了多种的授权方式。

## 2 OAuth 2

OAuth 2 是一个开放授权标准，定义了第三方用户如何访问系统。

### 2.1 基本概念

一些基本概念如下：

* [Resource Owner]^(资源拥有者) - 拥有着资源访问凭证（例如账户密码），例子中的 “我”
* [Resource Server]^(资源服务器) - 保证这资源的服务器，例子中的 GitHub
* [Client]^(第三方应用) - 需要得到授权，访问资源的引用，例子中的 CI
* [Authorization Server]^(授权服务器) - 根据 Resource Owner 意愿颁发 Access Token 的服务器
* [Authorization Grant]^(授权许可) - Resource Owner 授予 Client 访问 Authz Server 的凭证
* [Access Token]^(访问令牌) - Authz Server 授予 Client 访问 Resource Server 的凭证

可以看到，Authz Server 是中间层，通过颁发 Token 给 Client，使得 Client 在不知道账户密码情况下就可以访问 Resource Server 获取资源。

{{< admonition note Note>}}
Authz Server 和 Resource Server 可以是同一个。例如例子中往往我们会使用 GitHub 提供授权的功能。
{{< /admonition >}}

基本的访问流程如下图：

{{< image src="img1.png" height=300 >}}

1. Client 访问 Resource Owner，发起授权请求。
2. Resource Owner 返回 [Grant]^(授权许可) 给 Client，表示允许。
3. Client 使用 Grant 访问 Authz Server，申请 Token。
4. Authz Server 验证 Grant，允许后返回 Token。
5. Client 使用 Token 访问 Resource Server；
6. Resource Server 验证 Token 以及权限检查，返回 Resource；

上述只是基本的流程，其中“如何获取 Grant”、“如何获取 Access Token” 等都是可以扩展的地方。为此 OAuth 2 提出了四种方式：

* Authorization Code
* Implicit
* Resource Owner Password Credentials
* Client Credentials

### 2.2 四种授权模式

#### 2.2.1 Authorization Code

[Authorization Code]^(授权码模式) 是最严格但是最常用的的方式，考虑了几乎所有的敏感信息泄漏的预防和后果。

在 Client 获取 Token 时，做了严格的控制，要求必须由 Resource Owner 手动允许。并且要求 Client 有着自己的服务器，使得能够将 Authz Code 从 Resource Owner 传递到 Client。

{{< image src="img2.png" height=400 >}}

开始授权前，Client 先要到 Authz Server 注册，从 Authz Server 中获得 Client 专属的 ClientID 和 ClientSecret。

1. Client 将 Resource Owner 导向到 Resource Server 的授权页面，由 Resource Owner 手动允许授权。同时，也会带有 ClientID 以及 Client Secret。
2. Resource Owner 同意后，Authz Server 就收到了允许授权的请求，并验证 Client 的相关权限。
3. Resource Server 返回给 User Agent 对应的 Client URL 以及 Authz Code。
4. User Agent 访问 Client URL，传输 Authz Code 给 Client。
5. Client 拿到 Authz Code 后，使用 Authz Code 访问 Resource Server，以申请 Access Token。
6. Resource Server 返回 Token，后续由 Client 使用 Token 获取资源。

后续还有一些关于 Token 的过期，刷新的机制。

#### 2.2.2 Implicit

[Implicit]^(隐式授权模式) 省略了 Authz 颁发 Authz Code 的步骤，并且明确禁止发放刷新令牌。适用于 Client 没有服务器来处理 Authz Code 的情况。

{{< image src="img3.png" height=300 >}}

可以看到，授权服务器不会验证第三方应用的身份，在得到用户授权后，直接返回访问令牌给第三方引用。显然，这样减低了安全性，但是无需注册提升了灵活性。

另外，RFC 6749 中明确说明令牌必须是通过 URI 的 Fragment 带回的。

#### 2.2.3 Resource Owner Password Credentials

[Resource Owner Password Credentials]^(密码模式) 整合了认证和授权的过程。Client 直接拿着 Resource Owner 密码向 Authz Server 换取 Token。

因为 Client 直接使用了 Resource Owner 凭证，因此这种情况一般用于 Client 被高度信任的情况下。

{{< image src="img4.png" height=250 >}}

#### 2.2.4 Client Credentials

[客户端模式]^(Client Credentials) 指 Client 直接以 Resource Owner 名义，向 Authz Server 申请 Token。该模式通常用于管理操作或者自动处理的场景中，即常用于服务间认证授权。

{{< image src="img5.png" height=250 >}}

### 2.3 Refresh Token

上述得到 Access Token 同时，一般也会提供给 Client 一个 Timeout 和 Refresh Token。当 Access Token 过期时，Client 可以使用 Refresh Token 再次获取新的 Access Token，而不用让用户再次登陆授权。

### 2.4 传递 Token

在 RFC6750 定义了，Client 拿到 Access Token 后，如何传递 Resource Server。

包括三种方式：

* URI Query Parameter - 基于 Parameter 参数传递

  ```http
  GET /resource?access_token=mF_9.B5f-4.1JqM HTTP/1.1
  Host: server.example.com
  ```

  注意添加 `Cache-Control:no-store` 防止中间件缓存。

* Authorization Request Header Field - 基于专用的 Header Authorization
  
  ```http
  GET /resource HTTP/1.1
  Host: server.example.com
  Authorization: Bearer mF_9.B5f-4.1JqM
  ```

  其中，Bearer 是固定的前缀信息。

* Form-Encoded Body Parameter - 基于 Body
  
  ```http
  POST /resource HTTP/1.1
  Host: server.example.com
  Content-Type: application/x-www-form-urlencoded
  
  access_token=mF_9.B5f-4.1JqM
  ```

  同时，`Content-Type: application/x-www-form-urlencoded` 是必须的，并且不可以使用 GET 访问。

## 3 OIDC

[OIDC]^(OpenID Connect) 是基于 OAuth 2 的身份认证协议，是 OAuth 2 的超集。也就是说搭建了一个 OIDC 的服务后，完全可以当做一个 OAuth2 来使用。

OIDC 在 OAuth 2 上构建了适用任何客户端的身份层，用于身份认证。后续流程使用 OAuth 2 负责授权。

### 3.1 基本概念

OIDC 在 Access Token 基础上引入了 ID Token，提供用户信息。

基本概念如下：

* End User - 个人用户
* Relying Party - 第三方应用，即 Client
* OpenID Provider - 有能力提供 EU 认证的服务，为 RP 颁发 ID Token 与 Access Token
* ID Token - JWT 格式的 Token，包含 EU 的身份标识
* UserInfo Endpoint - 用户信息接口，由于 RP 请求获取用户信息

{{< admonition note Note>}}
OpenID Provider 也称为 Identity Provider。
{{< /admonition >}}

基本的流程如下：

{{< image src="img6.png" height=400 >}}

1. PR 发送身份认证请求给 OP。
2. OP 需要对 EU 进行身份认证，并请求授权。通常会导向到 EU 的登录页面。
3. OP 将 ID Token 与 Access Token 返回给 RP。
4. RP 使用 Access Token 请求 UserInfo Endpoint，根据 ID Token 获取 User 相关信息；
5. OP 返回 User 信息；

根据如何获取 ID Token，OIDC 扩展出了三种方式：

* Authorization Code Flow
* Implicit Flow
* Hybrid Flow

### 3.2 ID Token

ID Token 是 JWT 格式的数据结构，基于 JWT 的防篡改机制，使得 ID Token 可以安全的传递给 RP，用于后续 RP 请求 EU 信息时进行验证。

ID Token 的主要构成如下：

```json
{
  "iss": "https://oidc.eks.us-west-2.amazonaws.com/id/12312312",
  "sub": "system:serviceaccount:default:controller"
  "aud": [
    "sts.amazonaws.com"
  ],
  "nonce": "xxxx",
  "exp": 1666838679,
  "iat": 1666752279,
  "auth_time": 1311280969,
  "acr": "xxxx"
}
```

* iss (Issuer Identifier) - ID 颁发者（Identity Provider）的唯一标识，一般是个 HTTPS URL
* sub（Subject Identifier）- iss 提供的 EU 标识信息，RP 可以同它来标识唯一的 User
* aud（Audience）- 标识 ID Token 的目前受众，也就是哪些服务能够处理 ID Token
* exp（Expiration time）- Token 的过期时间
* iat（Issued At Time）- Token 的构建时间
* auth_time（Authentication Time）-（可选）EU 完成认证的时间
* acr（Authentication Context Class Reference）-（可选）标识一个上下文

### 3.3 授权方式

#### 3.3.1 Authorization Code Flow

[Authorization Code Flow]^(授权码) 使用 OAuth 2 的授权码方式，来换取 ID Token 与 Access Token。

#### 3.3.2 Implicit Flow

[Implicit Flow]^(隐式授权) 在 OAtuh 2 的 Implicit 颁发 Access Token 时，同时加上 ID Token。

#### 3.3.3 Hybrid Flow

[Hybrid Flow]^(混合授权) 同时使用 Authorization Code + Implicit。

### 3.4 UserInfo Endpoint

UserInfo Endpoint 是 OP 提供的一个接口，也可以看做受到 OAuth 2 保护的资源。因为 RP 需要使用 Access Token 才可以请求该资源。

为了防止 ID Token 承载太多的信息，所以抽出 UserInfo Endpoint 接口用以获得 User 的具体信息。并且规定，该接口必须使用 TLS。

当 RP 收到 OP 提供的 ID Token 与 Access Token 后，就可以调用 UserInfo Endpoint 去获取更多的信息或者资源。

## 参考

* Blog：[**OAuth2 授权**](https://www.cnblogs.com/linianhui/p/oauth2-authorization.html)
* Blog：[**OIDC 身份认证**](https://www.cnblogs.com/linianhui/p/openid-connect-core.html#auto-id-9)
* [**OpenIDConnect**](https://openidconnect.net/)



