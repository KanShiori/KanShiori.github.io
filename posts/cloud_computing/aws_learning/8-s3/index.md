# AWS 学习 - 8 - S3


## Storage Type

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

## 存储管理

### S3 Lifecycle

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

#### Transition

Transition 支持自动在不同 Storage Type 转换，以节省存储成本。
{{< find_img "img1.png" >}}

#### Expiration

当文件满足 Expiration 定义的时间后，S3 会将其自动删除。

如果需要知晓一个文件的过期时间，你可以通过 HEAD/GET HTTP 访问文件，这样可以得到文件计划过期时间的 HEAD。

### Object Lock

通过 Object Lock 功能，可以防止对象在第一次写入后不再允许修改或者删除。

Object Lock 支持两种方式锁定方式：
* Retention period - 指定对象在一段时间内锁定
* Legal hold - 对象永久保留（直到明确移除 Legal hold）

Object Lock 只能在创建 Bucket 时候开启

### Replication

### Batch Operation

## 访问控制

### IAM

### Bucket Policy

### ACL

### 访问分析器

## 数据处理

## 监控

## 工作原理


