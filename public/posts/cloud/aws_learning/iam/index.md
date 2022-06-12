# AWS - IAM


使用 AWS 时，不同的员工或者应用程序有着不同的 AWS 资源需求，通过 IAM 对其访问权限进行控制。

IAM 本质上是为了完成两个功能：
* **`Authentication`** - 身份认证，操作资源前**确认请求者是 “谁”**；
* **`Authorization`** - 授权，**操作者是否有权限操作资源**；

这个 “谁” 被称为 **`Entity`**，后面看到的 User、User Group、Role 都属于 Entity。

## 1 与 AWS 交互

有四种访问 AWS 服务的方式，**所有的方式本质上都是通过请求 AWS API 来访问，不同方式发送时带有的凭证方式不同**。

* **Console** - Username + Password 登陆网页
* **CLI** - Access Key + Secret Key（后续简称为 AK/SK）
* **SDK** - AK/SK
* **AWS Service** - Assume Role

所有的 API 请求会发送到各个 Region 的 AWS **`API 终端节点`**，所有请求都会经过 IAM 的身份认证与授权检查，以及 CloudTrail 的记录。
{{< find_img "img1.png" >}}

所有访问方式发送 API 时都会带有一个凭证，其内容包括：
* **请求主体**；
* **签名** - <important>由 SK 作为私钥，对 请求主体 + 元信息 + 时间戳 等信息进行加密，得到签名</important>；

该凭证实现的作用：
* **身份验证** - SK 签名实现 - AWS 服务端会查找对应的 AK 进行解密，从而验证是否是对应的 SK 加密的；
* **防止篡改** - 摘要信息实现 - AK 解密后得到摘要信息，通过对比摘要信息来检查请求主体是否被篡改；
* **防止重放攻击** - 时间戳实现 - 请求中会带有 时间戳 信息，通过时间戳信息来判断请求有效期（默认 5min），防止重放攻击；

{{< admonition tip "AK/SK 的作用">}}
可以看到，**AK 作用类似于公钥，SK 作用类似于私钥**。
{{< /admonition >}}

当然，所有的凭证生成对于用户都是透明的，用户只需要配置好 AK/SK 即可访问 AWS 服务。

## 2 Root User / IAM User / User Group

### 2.1 Root User

创建 AWS Account 时，会自动创建一个 **`Root User`**，其权限不受控制。

对 **`Root User`**，因为其权限太大，所以日常使用不要使用，同时推荐对其开启 [**MFA**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html)，登陆时要求两步验证。

{{< admonition note "Root User 权限收到控制的情况">}}
有个例外，当使用 AWS Organization 限制时，某个 Account 的 **`Root User`** 会受到 [**Service Control Policy**](#24-service-control-policy) 限制。
{{< /admonition >}}

最佳实现是：最开始创建 Account 时，先用 **`Root User`** 创建 Admin IAM User（作为顶级管理员），然后使用 Admin IAM User 去创建各个 IAM User。

### 2.2 IAM User

**`IAM User`** 访问 AWS 服务收到 IAM 的限制，**通过 IAM Policy 来限制其访问权限**，以满足最小权限原则（只有用户至少需要的权限）。

每个 IAM User 有着**独立的 Username/Password 与 AK/SK**。每个 IAM User 最多有两套 AK/SK，以方便轮换密钥：先停用当前使用的密钥，切换使用新的密钥，后删除旧的密钥。

> 后面将 IAM User 直接简称为了 User。

### 2.3 User Group

为了更加轻松的管理 User 与他们拥有的权限，可以将 User 添加到 **`User Group`** 中。

**当将 `IAM Policy` 附加到某个 Group 中，该 Group 内的所有 User 都具有了相应的权限**。

### 2.4 Service Control Policy

使用 AWS Organization 管理多个 Account 时，Organization 可以通过 **`Service Control Policy`** 控制一个或一组 Account 的访问权限，Account 下所有 Entity 操作都会受到该 Policy（包括 Root User）控制

**`Service Control Policy`** 就是 Policy，会作用于 IAM 进行 Policy 评估时（后续会看到）。

## 3 Policy

Policy 定义了权限这个概念，<important>通过将 Policy 附加到 User/Role，就可以限制其对 AWS 资源的访问权限</important>。

{{< admonition tip Note>}}
Policy 仅仅是静态的权限定义，能够限制 “谁” 是靠附加 Policy 实现的（除了 Resource-based Policy）。
{{< /admonition >}}

其格式类似于：
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "ec2:*",
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:CreateServiceLinkedRole",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:AWSServiceName": [
                        "autoscaling.amazonaws.com",
                        "ec2scheduled.amazonaws.com",
                        "elasticloadbalancing.amazonaws.com",
                        "spot.amazonaws.com",
                        "spotfleet.amazonaws.com",
                        "transitgateway.amazonaws.com"
                    ]
                }
            }
        }
    ]
}
```
* Statement - 各个语句数组，多个规则之间为 OR 逻辑；
  * `Effect` - 描述该语句是 Allow/Deny；
  * `Action`/`NotAction` - 语句针对哪些 API 进行控制，支持通配符（例如 ec2:start*）；
  * `Resource`/`NotResource` - 语句针对哪些资源进行控制，使用 ARN 地址；
  * `Condition` - 语句生效的额外匹配条件，多个条件之前为 AND 逻辑；
  * `Principal` - （仅用于 **`Resource-based Policy`**）语句作用于 User/Service；

{{< admonition note Note>}}
注意 Resource 与 Principal，Resource 指操作的目标，Principal 指谁进行操作。理论来说，Principal 可以由 Condition 来实现。
{{< /admonition >}}

### 3.1 Policy 的类型

根据 Policy 的管理方式，分为：
* AWS 托管 - 由 AWS 进行维护与更新，包含大多数常用的权限语句；
* 客户托管 - 由客户自定义与维护，支持版本控制；
* 内联 - （很少使用）针对于 User/Role 直接绑定的 Policy，不可复用；

根据 Policy 应用主体，分为：
* **`Identity-based Policy`** **基于身份的策略** - 定义 Policy，并附加到 User/Role 来实现对其权限的控制；
* **`Resource-based Policy`** **基于资源的策略** - 与特定资源绑定的 Policy，配置后限制 “谁” 能访问该资源;
  
  例如 S3 Bucket Policy 配置后限制哪些 Entity 可以访问这个存储桶，Lambda Policy 限制哪些 Entity 可以调用这个 Lambda。


## 4 Role

User 有着长期使用的 Username/Password 与 AK/SK，并不适合临时提供给他人使用。而 **`Role`** 就是用于<important>解决临时的授权操作</important>，主要用于：
* **AWS 服务 Assume Role 后，能够拥有访问 AWS 资源的权限**；
* **User（当前账户 User 或者其他账户 User）Assume Role 后，能够拥有访问 AWS 资源的权限**；

使用 Role 的基本过程为：
1. 创建 Role；
2. 将 Policy 附加到 Role 上；
3. User/Service 进行 Assume Role 操作；

Assume Role 后，会在一定时间后过期，也支持**手动撤销（Revoke）所有已经分配**（不影响后续使用）。

创建一个 Role 并分配了 Policy 后，可以将 Role 授予某个 User 或者 Resource。这样 User/Resource 会丢弃已有的权限，仅拥有 Role 定义的资源访问权限了。

### 4.1 Trust Relationship

可以看到，User/Service 需要执行 Assume Role 操作来作为一个 Role，而一个 Role 的 **`Trust Relationship`** 就是控制 “谁” 能够 Assume 这个 Role。

**`Trust Relationship`** 本质就是一个 Policy，其控制的是 `sts:AssumeRole`/`sts:AssumeRoleWithWebIdentity` API，从而实现了对 Assume 的控制。
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### 4.2 Session Policy

执行 Assume Role 行为时，可以传递一个 **`Session Policy`**，作为**本次 Assume Role 期间额外添加的 Policy**。通过 Session Policy 实现一次性的约束。

### 4.3 Assume Role

Assume Role 本质上是调用 `sts:AssumeRole`/`sts:AssumeRoleWithWebIdentity` 相关 API，调用时会检查 **`Trust Relationship`**（实际上就是 IAM 检查 Policy）。

Assume 成功后，User/Service 会得到一个 AK/SK + Token 等信息，同时带有着超时时间。这些信息作为一个**临时凭证**，提供访问 AWS 资源的权限。而该临时凭证由 AWS STS 来颁发。


## 5 Permission Boundary

默认下，当 Admin User 修改 Entity 权限（Policy）时，是可能提供过大的权限。这样操作，仅仅拥有管理 IAM 的 User 通过创建权限很大的 User 可以做到任何操作。

**`Permission Boundary`** 本质上就是**为某个 Entity 指定一个全局的 Policy**。当创建 User/Role 时，可以选择是否要带上 **`Permission Boundary`**（可选项）。

强制要求使用 **`Permission Boundary`**：即使 **`Permission Boundary`** 是可选项，我们可以通过对拥有创建 User/Role 权限的 User 的 Policy 进行条件约束，**要求其调用 `iam:CreateUser` 等 API 时，必须要求带上 `Permission Boundary`**。
```json
{
  "Effect": "Allow",
  "Action": [   // -> 针对创建 User 等相关的 API
    "iam:CreateUser"
    // ...
  ]
  "Condition": {
    "StringEqua1s": {
      "iam:permissionsBoundary": "arn:aws:iam::ACCOUNT_ID:policy/PolicyX"，// -> 要求 Permission Boundary 必须指定了某个 Policy
    }
  }
}
```

## 6 Policy 评估

总结一下前面提到的所有 IAM 会评估的 Policy：
* [**Identity-based Policy**](#31-policy-的类型)
* [**Resource-based Policy**](#31-policy-的类型)
* [**Session Policy**](#42-session-policy)
* [**Permission Boundary**](#5-permission-boundary)
* [**Service Control Policy**](#24-service-control-policy)

每个 IAM 检查 API 请求时，所有的设置了的 Policy 都会被评估，整体可以见下图：
{{< find_img "img2.png" >}}

看着流程很复杂，但是可见总结一下其判断逻辑：
1. **显式拒绝** - 任何存在的 Policy，只要判断出 Deny，但是结果就是 Deny；
2. **基于资源策略显式允许** - 不存在显示拒绝情况下，如果 Resource-based Policy 是 Allow，那么结果是允许（不参考其他的 Policy）；
3. **其他策略交集取允许** - 其他策略要所有策略都需要显式的 Allow；
4. **隐式拒绝** - 所有策略没有显式 Allow，那么结果为 Deny；

{{< admonition note Note>}}
所有策略评估都是针对于使用了策略情况下，如果没有应用策略那么不会对结果有影响。
{{< /admonition >}}

以伪代码方式看：
```go
if anyDeny {
  return Deny  // 显式拒绝
}

if ResourceBasedAllow {
  return Allow  // 基于资源策略显式允许
}

if AllAllow {
  return allow  // 交集取允许
}

return Deny  // 隐式拒绝
```

## 7 Security Token Service

使用 AssumeRole 提供的临时权限本质上，是为 Role 申请了一个 Token，用以访问 AWS。这些临时的 Token 都是由 Security Token Service 来颁发的，简称为 STS。

使用 Token 与 IAM User 的差异主要在于：

* Token 是临时的，有着过期时间，过期后不再能够通过身份认证；
* Token 不是跟随用户的，需要通过 AWS API 动态申请的；

### 7.1 申请 Token 的接口

AWS API 中有多个接口可用于请求 STS 以获取 Token：

* AssumeRole
  
  AssumeRole 为 IAM User 获取 Role 对应的 Token。

  Token 有效时间默认为 1h，支持设置 15min 到 Role Session 持续时间。

* AssumeRoleWithWebIdentity

  AssumeRoleWithWebIdentity 为经过第三方身份认证的用户获取 Role 对应的 TOken。这样配合着第三方的认证也称为：Web [联合认证]^(identity federation)。

  Token 有效时间默认为 1h，支持设置 15min 到 Role Session 持续时间。

  第三方的认证包括：Login with Amazon、Facebook、Google 和 OIDC。

  {{< admonition note Note>}}
  Kubernetes 中就是使用 OIDC 的访问来获取 STS，使得 Pod 内能够访问 AWS 资源。甚至能够跨云 Kubernetes 访问 AWS。

  AWS 提供了示例 [**Web Identity Federation Playground **](https://web-identity-federation-playground.s3.amazonaws.com/index.html)。
  {{< /admonition >}}

* AssumeRoleWithSAML
  
  AssumeRoleWithSAML 为基于 SAML 实现的身份系统的用户获取 Role 对应的 TOken。

  Token 有效时间默认为 1h，支持设置 15min 到 Role Session 持续时间。

* GetFederationToken
  
  GetFederationToken 为联合身份用户获取 Role 对应的 TOken。
  
  不同于 `AssumeRole`，该接口的默认有效期为 12h 而不是 1h，并且支持设置 15min 到 36h 的有效时间。较长的有效事件以减少获取新 Token 的次数。

* GetSessionToken
  
  GetSessionToken 为 IAM User 获取 Role 对应的 Token，通过支持 MFA 提供更高的安全性。

## 参考

* 官方文档：[**IAM User Guide**](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)
