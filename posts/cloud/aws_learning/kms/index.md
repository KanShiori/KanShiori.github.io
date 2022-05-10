# AWS - KMS


## 1 概述

Key Management Service（简称 KMS）是用于管理密钥的服务：KMS 保管密钥，用户请求 KMS 以加密或解密数据。大多数的 AWS 服务都支持直接使用 KMS，以加密保存的数据。

大多数 AWS 服务使用 [信封加密]^(Envelope Encryption) 来加密数据：KMS 使用 KMS Key 来 **加密/解密** Data Key，AWS 服务使用原文的 Data Key 来 **加密/解密** 数据。

{{< image src="img1.png" height=180 >}}

* KMS Key - 加密 Data Key 的密钥，也称为 Master Key
* Data Key - 加密数据的密钥

{{< admonition note Note>}}
KMS 支持生成、加密和解密 Data Key，但是不会存储与管理 Data Key。因此，存储加密后的 Data Key 是 KMS 使用者的职责。
{{< /admonition >}}

使用 KMS 的优势在于：

* KMS 使用 HSM 保护着关键的 KMS Key，并且 Data Key 的加解密都是在 KMS 服务内部完成，KMS Key 永远不会被传输到 KMS 服务之外，降低泄漏的风险；
* 基于信封加密的方式，读取 Key 的权限与数据访问的权限是分离的。以 S3 为例，只有拥有 KMS 相关访问权限和 S3 访问权限的人，才能够得到最后的明文；

## 2 密钥管理

### 2.1 KMS Key

KMS 通过 KMS Key 来加解密。大多数情况下直接使用对称 KMS 密钥，也支持使用非对称 KMS 密钥。

KMS Key 除了包含密钥材料外，还包含一些元信息：密钥 ID、创建日志、描述和状态。

KMS Key 可以由 KMS 自动生成，也支持客户自己导入密钥材料。不过，客户无法导出和查看密钥材料。

#### 2.1.1 密钥分类

根据管理方式的不同，KMS 将密钥分为三类：

* Customer managed key
  
  用户创建的密钥，用户可以管理 KMS Key 的生命周期。可以选择是否 Rotate。

* AWS managed key
  
  AWS 为账号自动创建的 KMS 密钥，一些 AWS 服务不支持使用 Customer managed key，直接使用 AWS managed key 来进行加密。

  用户无法管理 AWS managed key，并且 AWS managed key 固定每三年自动轮换。

* AWS owned key
  
  AWS 全局的密钥，与用户无任何关系，一些 AWS 服务会使用 AWS owned key 来进行加密。

#### 2.1.2 对称加密与非对称加密

KMS 支持管理对称密钥和非对称密钥。

* 对称密钥
  
  对加密要求加解密使用相同的密钥，因此如果使用对称密钥，所有的加解密操作必须由 KMS 来处理。

  大多数 AWS 服务都是使用的 对称密钥 + 信封加密 的方式，KMS 负责加解密 Data Key。

* 非对称密钥
  
  如果使用非对称密钥，那么 KMS 负责管理私钥，用户可以使用 KMS 来进行公钥加密，也可以下载公钥并在 KMS 外部进行公钥加密。

### 2.2 Enable Disable

KMS Key 在创建完成后默认处于 Enabled。用户可禁用 KMS Key，禁用后 KMS Key 不能用于处于任何加密操作。

启用与禁用是可以来回操作的，因此在删除 KMS Key 之前，先考虑禁用该 Key。在一段时间观察不会有任何影响后，再删除该 Key。

### 2.3 Rotate

#### 2.3.1 自动轮换

KMS 支持对 KMS Key 开启自动轮换（仅支持对称密钥），KMS 会周期性地替换密钥材料，但是 KMS Key 相关的元属性不会有任何改变。

KMS 会保存 Key 所有旧的密钥材料，因此 KMS 使用新密钥材料加密新数据的同时，也支持解密旧数据。也就是说，自动轮换对于用户完全是透明的。

{{< image src="img3.png" height=150 >}}

{{< admonition note Note>}}
因为不会重新加密旧数据，因此自动轮换是无法处理 Data Key 泄漏的影响的，应该使用手动轮换。
{{< /admonition >}}

#### 2.3.2 手动轮换

手动轮换本质上是用户自己创建新的 KMS Key，替代旧的 KMS Key。新的 KMS Key 完全是一个独立的 Key，有着新的 Key ID 与 ARN。

因为 KMS Key 完全变化了，要求用户需要使用新的 KMS Key 来重新加密数据。也就是说，应用程序中需要判断密钥已经轮换，并且执行一次重新加密：使用旧密钥解密数据，然后使用新密钥加密数据。

{{< image src="img4.png" height=150 >}}

手动轮换的示例见：[**Rotate manually**](https://aws.amazon.com/cn/premiumsupport/knowledge-center/rotate-keys-customer-managed-kms/)。

### 2.4 Delete Key

KMS Key 被完全删除后，意味着加密的数据将无法恢复。因为 KMS Key 的重要性，AWS 不会直接删除，会在一定时间的等待期后才真正删除。

> 因为我们不知道 KMS Key 加密了多少数据，所以正确做法应该是先 Disable Key。

默认的等待期为 30 天，可以设置为 7-30 天。在等待期内，KMS Key 状态为 Pending deletion。

待删除的 KMS Key 不能用于任何加密操作，也不会进行 Rotate。

处于 Pending deletion 状态的 KMS Key 可以取消删除，只要有着 `kms:CancelKeyDeletion` API 的权限。

### 2.5 Multi-Region Key

默认下，所有创建的 KMS Key 都是 Regional 的，使用时需要指定 Region。KMS 也支持 Multi-Region KMS Key。Multi-Region Key 存在一个 primary key，以及多个 replica key。

创建 KMS Key 指定 Multi-Region 支持后，该 Region 下的 KMS Key 就是 primary key。后续可以将主密钥复制到其他 Region 中，成为 replica key。

{{< image src="img5.png" height=150 >}}

虽然 primary key 与 replica key 有着相同的元属性与密钥材料，但是本质上它们完全是独立的，有着独立的 Key Policy、Grant、Alias 和 Tag。同样，禁用某个 Region 的 Key 也不会影响到其他 Region 的 Key。

## 3 加密操作

KMS Key 是由 KMS 保管的，因此任何使用 KMS Key 的操作都需要调用 KMS API 来执行。

例如，调用 Encrypt 接口，指定 KMS Key 来加密原文：

```json
$ aws kms encrypt --key-id <key-id> --plaintext <plaintext>
{
    "CiphertextBlob": "<ciphertext>",
    "KeyId": "<key-arn>",
    "EncryptionAlgorithm": "SYMMETRIC_DEFAULT"
}
```

对应的，使用 Decrypt 来解密：

```json
$  aws kms decrypt --key-id <key-id> --ciphertext-blob <ciphertext>
{
    "KeyId": "<key-arn>",
    "Plaintext": "aGVsbG8K",
    "EncryptionAlgorithm": "SYMMETRIC_DEFAULT"
}
```

## 4 Alias

KMS Key 使用 Key ID 来索引，不利于记忆与使用。可以使用 Alias 来为 Key 定义别名。

Alias 是独立的资源（不是 KMS Key 的属性），有着独立的 Alias ARN。需要先创建 Alias 然后将其绑定到 KMS Key 上。这样，使用该 Alias 就可以管理对应的 KMS Key 了。

Alias 与 KMS Key 的关联关系：

* Alias 只能映射到一个 KMS Key，不过 KMS Key 可以拥有多个 Alias；

* 如果删除 Alias，不会对关联的 KMS Key 有任何影响。但是，如果删除 KMS Key，那么会同时删除所有关联的 Alias。

Alias 的特性：

* KMS API 中，Alias 名称始终以 `alias/` 为前缀。

* Alias 是 Regional Resource，不同 Region 可以有着相同的 Alias。
  
  因此，使用 Alias 可以使得不同 Region 复用程序代码。

* AWS managed key 始终有着 Alias，名称格式为 `alias/aws/<service-name>`。

* 使用 Alias 可以控制 KMS Key 的访问权限。
  
  可以使用 `kms:RequestAlias` 条件来匹配 KMS API 请求中的 Alias 参数（无法匹配指定 Key ID 的请求）。

  或者使用 `kms:ResourceAliases` 条件匹配 KMS API 请求操作的 KMS Key 的 Alias（可以匹配指定 Key ID 的请求）。

## 5 访问控制

KMS Key 的访问控制支持多种方式，可以结合使用：

* [**IAM Policy**](../iam/#3-policy) - 针对于请求者来进行访问控制。

* Key Policy（必须）- 针对于 KMS Key 来进行访问控制。

* Grant - 赋予 Entity 使用 KMS 的权限。

### 5.1 Key Policy

Key Policy 是 KMS Key 的基于资源的策略，每个 KMS Key 必须有一个 Key Policy。这也意味着，所有针对于 KMS Key 的请求必须由 Key Policy 允许。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Describe the policy statement",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:user/Alice"
      },
      "Action": "kms:DescribeKey",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:KeySpec": "SYMMETRIC_DEFAULT"
        }
      }
    }
  ]
}
```

### 5.2 Grant

Grant 是 KMS Key 特有的授权方式，创建 Grant 可以直接将 KMS Key 的访问权限赋予某个 Entity。

通常会使用 Grant 来授予 AWS 服务使用 KMS Key 的权限，以加密静态数据。

Grant 的一些特性：

* Grant 只允许授予允许访问的权限，不能定义拒绝访问。

* 每个 KMS Key 的 Grant 数量是有限制的。

* Grant 只能授予部分 KMS Key 的权限。具体见：[**授权操作**](https://docs.aws.amazon.com/zh_cn/zh_cn/kms/latest/developerguide/grants.html#grant-concepts)。

* 创建 Grant 之后不是立即生效的，有五分钟左右的延迟。如果需要立即生效，需要使用 Grant Token。

#### 5.2.1 使用 Grant

通过调用 KMS `CreateGrant` API 来创建 Grant，请求的一些参数：

* Name - Grant 的名字
* Key ID - 针对的 KMS Key ID
* Operations - 授予权限的操作
* Grantee principal - 授予权限的 Entity（Role User 等）
* Retiring principal（可选）- 允许取消授权的 Entity
* Constraints（可选）- 授权约束

```bash
 aws kms create-grant \
    --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
    --grantee-principal arn:aws:iam::111122223333:user/exampleUser \
    --operations Decrypt \
    --retiring-principal arn:aws:iam::111122223333:role/adminRole \
    --constraints EncryptionContextSubset={Department=IT}
```

#### 5.2.2 Grant Constraint

Grant Constraint 指的是在授予权限的同时，指定操作的约束条件。例如，加密操作的原文必须包含某些字符串。

#### 5.2.3 Grant Token

创建 Grant 后，需要一定的延迟才能生效（通常不到五分钟）。如果授权立即生效，需要使用 Grant Token。

Grant Token 其实是创建的 Grant 的字符串表示。在 KMS API 中传入 Grant Token 参数，可以让 API 立即生效而不用等待 Grant 的延迟。

Grant Token 只有在 `CreateGrant` API 调用时返回：

```bash
$ token=$(aws kms create-grant \
    --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
    --grantee-principal arn:aws:iam::111122223333:user/appUser \
    --retiring-principal arn:aws:iam::111122223333:user/acctAdmin \
    --operations GenerateDataKey Decrypt \
    --query GrantToken \
    --output text)
```

在 KMS API 中，通过传入 Grant Token 来使用：

```bash
$ aws kms generate-data-key \
    --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
    –-key-spec AES_256 \
    --grant-tokens $token
```

## 6 使用 KMS 密钥

### 6.1 AWS 服务

#### 6.1.1 EBS

EBS 可以 KMS Key 来加密存储的数据。加密和解密对于用户都是透明的：数据写入时会加密并保存到磁盘，读取数据时会先解密后读取。当然，前提是 EC2 有着 KMS Key 相关的加解密访问权限。

EC2 使用信封加密的方式来加密数据：使用 KMS Key 来加解密 Data Key，加密后的 Data Key 会与加密后的数据一起保存到磁盘上。

EBS 使用 KMS 的大致原理如下：

{{< image src="img6.png" height=270 >}}

#### 6.1.2 S3

S3 提供多种加密方案：

* CSE 
  
  客户端本地完成加解密，S3 直接保存加密后的数据。
  
* SSE-C
  
  用户上传数据的同时上传 Data Key，S3 使用 Data Key 加密后保存。

  用户下载数据的同时也需要上传 Data Key，S3 解密后返回数据。S3 不负责保存 Data Key。

* SSE-S3
  
  Data Key 由 S3 生成并保存，用户 上传/下载 数据后由 S3 负责 加密/解密。数据加密对于用户是无感知的。

* SSE-KMS
  
  与 SSE-S3 原理一样，区别在于 S3 会使用[**信封加密**](#1-概述)的方式，使用 KMS 来加密 Data Key。

SSE-KMS 方式与 KMS 集成，数据加解密过程对于用户是完全透明的。优势在于，使用 KMS 将密钥权限与 S3 访问权限分离，只有拥有 KMS Key 读取权限和 S3 访问权限的人才可以得到最后的密文。

{{< image src="img7.png" height=270 >}}

{{< admonition note Note>}}
S3 中每个文件的 Data Key 都是独一无二的。
{{< /admonition >}}

## 参考

* 官方文档：[**AWS KMS**](https://docs.aws.amazon.com/zh_cn/kms/latest/developerguide/overview.html)

