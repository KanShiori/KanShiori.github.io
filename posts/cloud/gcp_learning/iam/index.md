# GCP - IAM


## 1 基本模型

通过 IAM 控制 “谁” 对哪些资源有着哪种访问权限。

IAM 本质上是为了完成两个功能：
* **`Authentication`** - 身份认证，操作资源前**确认请求者是 “谁”**；
* **`Authorization`** - 授权，**操作者是否有权限操作资源**；

GCP IAM 的权限管理如下图：

{{< image src="img1.png" height=400 >}}

* **`Principal`** 表示资源操作者，也就是 “谁”，在 GCP 中包括：

* **`Role`** 表示一组权限的集合，决定了 Principal 对资源有着哪些访问全新；

* **`Role Binding`** 表示一个 Role 与多个 Principal 的绑定关系，表示真正为 Principal 赋予了 Role 对应的权限；

* **`Policy`** 是一组 Role binding 的集合，是某个资源所有设置的权限的集合；

## 2 Principal

Principal 表示资源操作者，也就是身份。Principal 支持的类型有：

* Google Account
* Service Account
* Google Group
* Google Workspace Account
* Cloud Identity Domain
* All Authenticated Users
* All Users

### 2.1 Google Account

**Google Account** 代表的就是一个终端用户，任何与 Google 账号关联的 Email 都可以作为 GCP 中的一个 Principal。

### 2.2 Google Group

Google Group 是 Google Account 和 Service Account 的集合，Group 有着唯一的 Email。

通过使用 Google Group，可以一次性管理 Group 整个的访问权限，而不需要一个个更改账户或者 Service Account 的权限。

不过，Google Group 仅仅用于批量配置权限，而不是真正的 Principal。也就是说，不能以 Google Group 的名义来访问资源。

### 2.3 Google Workspace Account

Google Workspace Account 是 Google Account 的集合，有着独立的 Domain。创建 Google Account 时，系统会自动将账户添加到 Google Workspace Account 中，例如 `username@example.com`。

与 Google Group 一样，Google Workspace 仅仅用于批量配置权限。

### 2.4 Cloud Identity Domain

Cloud Identity Domain 与 Google Workspace Account 一样，是 Google Account 的集合。但是 Cloud Identity Domain 无法访问 Google Workspace 的应用与功能。

### 2.5 All Authenticated Users

All Authenticated Users 指的是 Principal 值为特殊的 `allAuthenticatedUsers`，表示所有经过认证的 Principal。

### 2.6 All User

All User 指的是 Principal 值为特殊的 `allUsers`，表示所有经过认证和未经过认证的用户。

## 3 Service Account

与 [**AWS Role**](../../aws_learning/iam/#4-role) 的设计理念类似，Service Account 用于 GCP 中的应用程序使用。

与 Google Account 相对，Service Account 没有固定的密码，通过 Token 作为身份凭证。并且，也支持其他 Principal 或者 Service Account 来模拟 Service Account。

### 3.1 Service Account key pair

每个 Service Account 有着自己的 public/private 密钥对。此密钥对称为 **`Google-managed key pair`**。

通过调用 Service Account Credentials API，GCP 会使用 Google-managed key pair 创建出多个 **`User-managed key pair`**，使用 User-managed key pair 的私钥就可以作为 Service Account 来访问 GCP。

#### 3.1.1 Google-managed key pair

Google-managed key pair 是完全由 Google 管理的密钥对，用于生成短期的身份凭证：User-managed key pair。

GCP 会自动轮转 Google-managed key pair，使用时间最多两周。

Google-managed key pair 中的私钥始终中第三方托管，用户永远无法访问。

Google-managed key pair 中的公钥是可公开访问的，因此任何人都可以读取公钥来验证私钥创建的签名。通过以下 API 就可以访问公钥：

* X.509 证书格式：`https://www.googleapis.com/service_accounts/v1/metadata/x509/${SERVICE_ACCOUNT_EMAIL}`
* JWK 格式：`https://www.googleapis.com/service_accounts/v1/jwk/${SERVICE_ACCOUNT_EMAIL}`
* Raw 格式：`https://www.googleapis.com/service_accounts/v1/metadata/raw/${SERVICE_ACCOUNT_EMAIL}`

#### 3.1.2 User-managed key pair

通过调用 GCP API，GCP 会使用 Google-managed key pair 创建出 User-managed key pair。

使用 User-managed key pair 的私钥就可以作为 Service Account 访问 GCP 了。该私钥也称为 **`Service Account Key`**。

每个 Service Account 最多有 10 个 Service Account Key。GCP 只会存储公钥，Service Account Key 是完全由用户管理的。

目前支持两种方式创建 User-managed key pair：

* 使用 IAM API 创建 User-managed key pair，GCP 自动存储公钥，私钥将返回给用户；

* 用户自行创建 User-managed key pair，然后将公钥上传到 GCP；

### 3.2 Service Account 短期凭证

除了 Service Account Key 之外，GCP 也提供 API 用于创建 Service Account 的短期凭证。

与 Service Account Key 相比，使用短期凭证也可以访问 GCP，但是短期凭证持有短期的生命周期，例如几小时或者更短。

因此，如果需要为第三方提供 Service Account 的使用权限，那么 Service Account 短期凭证很有用。

目前支持的凭证类型包括：

* OAuth 2.0 Access Token
* OIDC ID Token

参考 [**Creating short-lived service account credentials**](https://cloud.google.com/iam/docs/creating-short-lived-service-account-credentials) 文档来创建短期凭证。

### 3.3 Service Account 的类型

根据管理 Service Account 的方式不同，分为：

* **用户管理的 Service Account**
  
  用户使用 GCP 创建的 Service Account。这种 Service Account 使用以下地址标识：`${sa-name}@${project}.iam.gserviceaccount.com`。

* **Default Service Account**
  
  启动某些 GCP 服务后，这些服务会创建自己的 Default Service Account。当使用这些 GCP 服务时，不指定 Service Account 那么就会使用 Default Service Account。

  目前的 Default Service Account 包括：

  {{< image src="img1-1.png" height=150 >}}

* **Google 管理的 Service Account**
  
  某些 GCP 服务需要访问用户的资源时，只能通过使用用户的 Service Account 才能操作用户的资源。为此，GCP 会为这些服务创建 Service Account，被称为 Google 管理的 Service Account。

  例如，Cloud Run 需要运行容器，那么需要使用对应的 Service Account 来创建容器。

  这些 Service Account 在 GCP Console 是不会展示出来，在 Policy 或者 Audit Log 中会看到它们。

### 3.4 Service Account 使用权限

前面说了，Service Account 是属于 Principal，可以用于访问其他 Resource。当然，Service Account 本身也是 Resource，可以配置 Policy 来控制其他 Principal 如何使用 Service Account。

GCP 提供了 Predefined Role 来管理 Service Account 使用权限：

* `roles/iam.serviceAccountUser`
  
  授予该 Role 后，表示允许使用该 Service Account 来访问 GCP。

* `roles/iam.serviceAccountTokenCreator`
  
  授予该 Role 后，表示允许创建该 Service Account 的短期凭证。

## 4 Role 与 Permission

与 [**AWS IAM Policy**](../../aws_learning/iam/#3-policy) 的设计理念类型，Role 表示的一组 Permission 的集合。

为 Principal 授予一个 Role 就是授予该 Role 包含的所有权限。

{{< admonition tip Note>}}
Role 仅仅是静态的权限定义，能够限制 “谁” 是靠 Role Binding 实现的。
{{< /admonition >}}

### 4.1 Permission

一个 Permission 对应于对某种资源的操作，以 `service.resource.verb` 的格式表示。

Permission 通常与资源管理的 REST API 对应。也就是说，Principal 必须被授予 API 对应的 Permission 后，才能调用对应的 API。例如，如果要创建某个 Bucket，那么必须要包含 `storage.buckets.create` Permission。

### 4.2 Role

一个 Role 的结构如下：

```yaml
# gcloud iam roles describe roles/storage.admin --format yaml
name: roles/storage.admin
stage: GA
title: Storage Admin
description: Full control of GCS resources.
etag: AA==
includedPermissions:
- storage.buckets.create
# ...
```

其中 `includedPermissions` 就是 Role 包含的权限的集合，每个权限代表着调用对应的 API 的权限。

### 4.3 Role 的类型

根据 Role 的管理方式，范围：

* **Basic Roles**
  
  权限范围很大的 Role，一般提供给 Admin User 管理 GCP。包括：Owner、Editor 和 Viewer。

* **Predefined Roles**
  
  GCP 预先创建好的 Role，由 GCP 维护与更新，包含某个资源的大多数权限。

  例如 `roles/storage.admin` 提供了 GCS 的管理权限。

* **Custom Roles**
  
  由用户自定义的 Role，用户可以根据权限需要自行组合 Permission。

## 5 Access Control

当 Principal 发出一个 Resource 操作请求时，GCP IAM 权限检查基于以下流程：

{{< image src="img2.png" height=400 >}}

1. 检查 Resource 所属的 **Organization 的 Policy**，判断 Principal 是否有权限操作资源。
   
2. 检查 Resource 所属的 **Folder 的 Policy**，判断 Principal 是否有权限操作资源。
   
3. 检查 Resource 所属的 **Project 的 Policy**，判断 Principal 是否有权限操作资源。
   
4. 检查 **Resource 的 Policy**，判断 Principal 是否有权限操作资源。

也就是说，GCP 会按照 Resource 的层次结构依次判断是否有操作权限。任一一个层次判断通过后，该 Resource 操作请求就会被通过。

### 5.1 Policy

GCP 的访问控制都是基于 Policy 管理的。Policy 包含了一组 Role Binding，描述了授予哪些 Principal 有着哪些 Role 的权限。

Policy 的结构如下：

```json
{
  "bindings": [
    {
      "condition": {
        "title": "condition-1",
        "expression": "request.time < timestamp(\"2020-01-01T00:00:00Z\")",
        "description": ""
      },
      "role": "roles/storage.objectAdmin",
       "members": [
         "user:ali@example.com",
         "serviceAccount:my-other-app@appspot.gserviceaccount.com",
         "group:admins@example.com",
         "domain:google.com"
       ]
    },
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "user:maria@example.com"
      ]
    }
  ],
  "etag": "BwXpzEB96jg=",
  "version": 1
}
```

* `bindings`
  * `role` - 表示授予 Principal 的 Role。
  * `members` - 表示授予的 Principal，不同类型的 Principal 有着不同的前缀。
  * `condition` - 权限评估时的条件检查
* `etag` - 用于并发冲突检查

GCP 中的每个 Resource 都有着对应的一个 Policy：

* **Resource-based Policy**
  
  GCP 中每个 Resource 实例都有着对应的一个 Policy，配置该 Policy 决定了哪些 Principal 可以访问该 Resource 实例。

  例如，要配置部分 Principal 可以访问某个 GCS Bucket，那么配置该 Bucket 的 Policy：

  ```json
  // gsutil iam get gs://my-bucket
  {
    "bindings": [
      {
        "members": [
          "serviceAccount:access-sa@my-project.iam.gserviceaccount.com",
          "user:shiori@example.com"
        ],
        "role": "roles/storage.objectViewer"
      }
    ],
    "etag": "CJkW"
  }
  ```

* **Identity-based Policy**
  
  与大多数 Resource Policy 不同的是，Project、Folder、Organization 的 Policy 是用于直接配置该层次范围内的 Principal 的权限的。

  例如，给 Project 下的某个 Service Account 配置能够访问所有的 GCS Bucket，那么我们需要修改 Project 的 Policy。

  ```json
  // gcloud projects get-iam-policy my-project --format json
  {
    "bindings": [
      {
        "members": [
          "serviceAccount:my-sa@my-project.iam.gserviceaccount.com"
        ],
        "role": "roles/storage.objectViewer"
      }
    ],
    "etag": "CJkW"
  }
  ```

### 5.2 修改 Policy

前面看到，所有的 IAM 访问控制都是配置 Policy 实现的。为了能够方便的修改 Policy，GCP 提供了两种改动方式：

* 增量更新 - 添加或删除一个 RoleBinding，不修改其他的 RoleBinding
* 全量更新 - 完全替换 Policy

#### 5.2.1 增量更新

增量更新用于单独管理每一条 RoleBinding。添加或删除一条 RoleBinding 时，不会修改其他 RoleBinding。

以 CLI 为例，CLI 的每个 Resource 的子命令都会有：

* `add-iam-policy-binding` - 添加一条 RoleBinding
* `remove-iam-policy-binding` - 移除一条 RoleBinding

以授予某个 Principal 使用 Service Account 权限为例：

```bash
gcloud iam service-accounts add-iam-policy-binding ${sa} --member ${member} --role=roles/iam.serviceAccountUser

gcloud iam service-accounts remove-iam-policy-binding ${sa} --member ${member} --role=roles/iam.serviceAccountUser
```

管理单条 RoleBinding 时，是以 Principal(member) 为 Key 的。每次只能给一个 Principal 配置一个 Role。

#### 5.2.2 全量更新

全量更新用于直接替换 Policy，也就能够支持同时管理多个 RoleBinding。

以 CLI 为例，CLI 的每个 Resource 的子命令都会有：

* `get-iam-policy` - 查看 Policy
* `set-iam-policy` - 替换 Policy

## 参考

* 官方文档：[**IAM**](https://cloud.google.com/iam/docs/overview)
