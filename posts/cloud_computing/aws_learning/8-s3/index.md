# AWS 学习 - 8 - S3


## 1 Storage Type

为了不同的存储场景，S3 提供了多种类型的存储类型。

| 存储类型               | 使用场景                                     |
| ---------------------- | -------------------------------------------- |
| S3 Standard            | 经常访问的数据                               |
| S3 Intelligent-Tiering | 动态自动优化访问模式                         |
| S3 Standard-IA         | 不经常访问数据，会跨多个 AZ 存储冗余保存数据 |
| S3 One Zone-IA         | 不经常访问数据，但没有多 AZ 数据备份         |
| S3 Glacier             | 归档数据，解冻数据时间为 1-5 分钟            |
| S3 Deep Archive        | 深度归档数据，解冻数据时间为 12 小时         |

各个存储类型的定价见 [**Amazon S3 定价**](https://aws.amazon.com/cn/s3/pricing/)。

## 2 存储管理

### 2.1 S3 Lifecycle

S3 Lifecycle 定义一组规则，用于配置 Bucket 自动对特定对象的操作。操作包括：

* Transition - 定义将对象转换为另一个 Storage Type 的时间。

    例如，可以选择在对象创建 30 天后将其转换为 S3 Standard-IA，创建 1 年后使用 S3 Glacier 存储。

* Expiration - 定义对象的过期时间，S3 将自动删除过期的对象。

S3 Lifecycle 可以通过多种方式配置。例如使用 CLI 配置 JSON 文件：
```json
{
    "Rules": [
        {
            "Filter": {
                "Prefix": "documents/"
            },
            "Status": "Enabled",
            "Transitions": [
                {
                    "Days": 365,
                    "StorageClass": "GLACIER"
                }
            ],
            "Expiration": {
                "Days": 3650
            },
            "ID": "ExampleRule"
        }
    ]
}
```
* Filter - 设置应用 Bucket 文件中的文件范围。支持通过 prefix 或者 tag 进行筛选。
* Status - 开启或关闭
* Transitions - Enabled/Disabled，定义 Transition 规则
* Expiration - 定义 Expiration 规则

#### 2.1.1 Transition

Transition 支持自动在不同 Storage Type 转换，以节省存储成本。
{{< find_img "img1.png" >}}

#### 2.1.2 Expiration

当文件满足 Expiration 定义的时间后，S3 会将其自动删除。

如果需要知晓一个文件的过期时间，你可以通过 HEAD/GET HTTP 访问文件，这样可以得到文件计划过期时间的 HEAD。

### 2.2 Object Lock

通过 Object Lock 功能，可以防止对象在第一次写入后不再允许修改或者删除。

Object Lock 支持两种方式锁定方式：
* Retention period - 指定对象在一段时间内锁定
* Legal hold - 对象永久保留（直到明确移除 Legal hold）

Object Lock 只能在创建 Bucket 时候开启，创建后无法修改。并且，Object Lock 必须开启 Object 的版本控制功能。

#### 2.2.1 使用方式

1. 创建 S3 Bucket 时开启 Object Lock 功能。
2. 上传文件到 Bucket（如果使用 AWS CLI/SDK，可以在上传文件时指定 Object Lock 参数）。
3. 在 Console 修改 object properties，应用 Retention Period 或者 Legal hold 保留模式。

#### 2.2.2 Retention Period

Retention Period 支持两种模式：
* Governance mode - 具有权限的 IAM User 可以修改锁定设置，并且可以调整保留期限
* Compliance mode - 任何 User（包括 root user）都无法覆盖或删除受保护的 object，期限也无法修改

Retention Period 允许在一定时间内锁定对象。开启 Retention Period 后，S3 会在对象版本的元信息中存储 `Retain Until Date`，以表明 object 的到期时间戳。当对象保留时期到期后，便可以覆盖或删除 object。

Retention Period 可以适用对 object 的各个 version，每个 object 的 version 可以有着不同的保留模式和周期。例如，可以将 object 保留为 15 天，之后将同名 object 存入 S3 并保留 60 天，这样 S3 将创建一个保留 60 天的 object，旧版本 object 保持原来对的保留期限。

通过对对象提交新的锁定请求，就可以更新 object 的 `Retan Until Date`。

#### 2.2.3 Legal hold

Legal hold 也可以防止 object version 被覆盖或者删除，不过 Legal hold 模式没有关联保存期限。

### 2.3 Replication

使用 Replication 的场景：

* 复制 Object，并保留元信息
  
  可以使用 Replication 来对 Object 创建副本，并且会保留 Object 的所有元信息（Object 创建时间，版本等）。

* 将 Object 复制到不同的 Storage Type 的 Bucket
  
  在 [**Storage Type**](#storage-type) 中介绍了各种的存储类型，可以使用 Replication 将 Object 复制到其他存储类型的 Bucket。

* 维护 Object 多个副本，每个副本提供给用户不同的访问权限

  无论谁创建了 Object，你可以将其 Object 复制到目标 Bucket，并修改 Object 的所属账户，这被称为 `owner override`。之后，你可以限制对 Object 副本的访问权限。

* 将 Object 保存在多个 Region
  
  在多个 Region 设置多个 Bucket，然后通过 Replication 保存相同的 Object。这可能有助于满足合规性要求。

* 15 分钟内复制 Object
  
  可以使用 S3 Replication Time Control（简称 RTC）在一定时间内完成 Replication。S3 RTC 在 15 分钟内复制 S3 中存储的 99.99% 的新对象。

#### 2.3.1 前提条件

使用 Replication 必须满足如下要求：
* 源 Bucket 拥有者的账户必须开启源 Region 与目标 Region，目标 Bucket 拥有者的账户必须开启目标 Region。
* 源 Bucket 和目标 Bucket 必须都开启 Version Control。
* S3 必须有足够的权限将源 Bucket 复制到目标 Bucket。
* 源 Bucket 的拥有者必须有所有 Object 的 `READ` 和 `READ_ACP` 权限（通过 Object ACL 可以开启）。
* 如果源 Bucket 开启了 Object Lock，那么目标 Bucket 也必须开启 Object Lock。

当跨账户复制 Bucket 时，有一些额外的要求：
* 目标 Bucket 的拥有者必须提供给源 Bucket 的拥有者复制 Object 的权限。参考：[**当源存储桶和目标存储桶由不同的 AWS 账户拥有时授予权限**](https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/userguide/setting-repl-config-perm-overview.html#setting-repl-config-crossacct)。
* 目标 Bucket 不能配置为 Requester Pays Bucket。

#### 2.3.2 复制的内容

默认下，S3 会复制 Bucket 中的以下内容：
* 配置 Replication 后，新添加的 Object
* 未加密的 Object
* 使用 SSE-S3 或者 SSE-KMS 静态加密的 Object（如果要复制这些加密 Object，必须显式启用该选项）
* Object 元数据
* 仅限源 Bucket 拥有者具体读取权限的 Object
* Object ACL 的更新
* Object Tag
* Object Lock 的信息（如果有）

默认下，S3 不会复制 Bucket 中的以下内容：
* 配置 Replication 之前存在的 Object，即 S3 不会复制已经存在的 Object
* 复制的 Object 不会传递。例如，Bucket A 复制到 Bucket B，Bucket B 复制到 Bucket C，这种情况下复制到 Bucket B 的 Object 不会复制到 Bucket C
* 已经被复制的 Object。例如，如果修改了 Replication 的目标 Bucket，那么 S3 不会复制已经旧的 Object
* 使用 SSE-C 加密的 Object
* 从不同的 AWS 账户删除标记复制时，添加到存储桶的删除标记不会被复制
* S3 Glacier 或 S3 Glacier Deep Archive 中的 Object 不允许复制
* 源 Bucket 拥有者没有权限访问的 Object
* S3 Lifecycle 标记。例如，启用 Lifecycle 会对 Object 创建删除标记，但是 Replication 不会复制这些标记。

如果你需要复制已经保存的 Object，需要联系 AWS Support。

### 2.4 Batch Operation

S3 Batch Operation 可以对 S3 执行大规模的批量操作，S3 可以提供完全托管的无服务器体验，并提供通知与详细的完成报告。

Batch Operation 的术语如下：
* Job - S3 Batch Operation 的基本工作单元，Job 中包含运行操作的全部信息。在 Job 开始后，Job 会为清单中的每个 Object 执行操作。
* Operation - 对 Object 执行的操作，例如复制 Object。对于清单中指定的所有 Object，每个 Job 只能执行一种 Operation。
* Task - Task 是 Job 的执行单元，Task 表示对 S3 或者 Lambda 的一个 API 调用。在 Job 整个的生命周期中，Batch Operation 会为每个 Object 创建一个 Task。
* Manifest - 包含 Job 需要操作的 Object 的清单。

#### 2.4.1 使用方式

创建 Batch Operation Job 时，需要提供以下信息：
* Operation - 指定对 Object 的操作。
* Manifest - 需要操作的 Object 的列表，Manifest 文件需要存储在 S3 Bucket 中。
* Priority - Job 在账户中对比与其他 Job 的优先级，数字越大，优先级越高。但是，优先级不代表 Job 之间的执行顺序。
* RoleArn - 运行 Job 使用的 IAM Role，必须要有足够的权限执行操作。
* Report - 指示是否需要生成完成报告。你可以配置报告的格式、信息等。
* Tags
* Description

使用 Batch Operation 的基本流程为：
1. 创建 Manifest。
2. 创建并运行 Job。创建成功后，S3 将返回一个 Job ID。
3. 查看报告

#### 2.4.2 支持的操作

Batch Operation 支持的操作包括：

* Copy
  
  复制 Manifest 中的每个 Object，可以复制到相同或不同 Region 的 Bucket。复制的同时，还支持额外的操作选项，包括设置 Object 元数据、设置权限或者更改存储类型。

* Invoke AWS Lambda function
  
  调用指定的 AWS Lambda 函数，来对 Manifest 中的 Object 执行自定义操作。Batch Operation 会对每个 Object 运行 Lambda 函数（LambdaInvoke API）。

* Replace all object tags
  
  替换 Manifest 中的每个 Object 的 Tag。

* Delete all object tags
  
  删除 Manifest 中每个 Object 的所有 Tag，不支持保留 Tag。

* Replace access control list
  
  替换 Manifest 中每个 Object 的 ACL。

* Restore

  还原 Manifest 中每个 Object，该操作主要用于还原 S3 Glacier 这类需要 “解冻” 的存储类中的 Object。

* Object Lock retention
  
  对 Manifest 中的每个 Object 执行 Lock retention。

* Object Lock legal hold
  
  对 Manifest 中的每个 Object 执行 Lock legal hold。

## 3 访问控制

### 3.1 访问权限控制

默认下，S3 资源（Bucket、Object 和其他子资源）都是私有的，只有资源拥有者和创建资源的 AWS 账户可以访问 S3 资源。

他人要访问私有的 S3 资源，可以通过授予 `Identity-based Policy` 或者 `Resource-based Policy`。

Resource-base Policy 就是基于基本的 IAM 来授予用户调用 S3 相关接口的权限。

Identity-based Policy 是以 S3 资源为对象，提供给被授予者相关 S3 资源的权限。包含两种方式：ACL 与 Bucket Policy。

#### 3.1.1 S3 中的资源

S3 中，Bucket 与 Object 是最基本的资源，并且它们有着相关的子资源。

Bucket 的子资源：
* lifecycle - [**生命周期配置信息**](#s3-lifecycle)
* website - 使用 Bucket 存储静态网站时的配置信息
* versioning - Bucket Versioning 的配置
* policy 和 acl - 保存存储访问权限信息
* cors - 支持配置 Bucket 以允许跨源请求
* object ownership - 允许 Bucket 取得 Bucket 中新对象的所有权
* logging - 请求 S3 保存 Bucket 的访问日志

Object 的子资源：
* acl - Object 访问权限信息
* restore - 支持临时还原 Archive Object

默认情况下，只有资源的拥有者可以访问这些资源。资源拥有者指的是创建资源的 AWS 账户，即：
* 创建 Bucket 和上传 Object 的 AWS 账户拥有这些资源。
* 如果使用 IAM User 或者 Role 上传 Object，那么 User 与 Role 所属的 AWS 账户拥有该 Object。
* Bucket 拥有者可以授予其他账户上传 Object 的权限。这种情况下，上传 Object 的账户拥有这些对象。而 Bucket 拥有者对这些 Object 没有权限。除了一些特殊情况：
  * Bucket 完全由 Bucket 拥有者付费，那么拥有者有着完全的访问权限。
  * Bucket 拥有权可以存档或还原 Object。

#### 3.1.2 ACL

ACL 表明了被授权者以及被授权的权限，每个 Bucket 与 Object 都有关联的 ACL。通过 ACL，你可以向其他 AWS 账户授予基本的 R/W 权限。

下例 ACL 表明了 Bucket 拥有者有着完全的控制权限：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<AccessControlPolicy xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <Owner>
    <ID>*** Owner-Canonical-User-ID ***</ID>
    <DisplayName>owner-display-name</DisplayName>
  </Owner>
  <AccessControlList>
    <Grant>
      <Grantee xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
               xsi:type="Canonical User">
        <ID>*** Owner-Canonical-User-ID ***</ID>
        <DisplayName>display-name</DisplayName>
      </Grantee>
      <Permission>FULL_CONTROL</Permission>
    </Grant>
  </AccessControlList>
</AccessControlPolicy> 	
```

#### 3.1.3 Bucket Policy

Bucket Policy 是针对于 Bucket 的 IAM Policy，将其绑定到其他 AWS 账户或者 IAM User 就可以使其获得相应的权限。

下面 Policy 实例，表明授予对一个 Bucket 所有对象的匿名读取权限：
```json
{
    "Version":"2012-10-17",
    "Statement": [
        {
            "Sid":"GrantAnonymousReadPermissions",
            "Effect":"Allow",
            "Principal": "*",
            "Action":["s3:GetObject"],
            "Resource":["arn:aws:s3:::awsexamplebucket1/*"]
        }
    ]
}
```

### 3.2 Block Public Access

默认情况下，新 Bucket、Endpoint 和 Object 不允许公有访问。但是，用户可以通过修改 Bucket Policy、Endpoint Policy 或者 Object Permission 以允许公有访问。

用户也可以设置 Block Public Access 覆盖上述开启公有访问的策略和权限。当 S3 收到访问 Bucket 或 Object 的请求时，它将确认 Bucket 或者 Bucket 拥有者是否开启了 Block Public Access。如果请求时 Endpoint 发出的，还会检查 Endpoint 的 Block Public Access 设置。

如果当前开启了 Block Public Access，那么 S3 将拒绝请求。

Block Public Access 有四种设置，可以使用任意组合将其应用到 Endpoint、Bucket 或者整个 AWS Account。如果设置到账户时，将会应用于账户下所有的 Bucket 与 Endpoint，如果设置到 Bucket，将会应用到 Bucket 关联的所有 Endpoint。

* BlockPublicAcls
* IgnorePublicAcls
* BlockPublicPolicy
* RestrictPublicBuckets

### 3.3 访问分析器

## 数据处理

## 监控

## 工作原理


