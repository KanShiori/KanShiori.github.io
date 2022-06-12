# AWS EBS


## 1 概述

AWS Elastic Block Store（简称 EBS） 是 AWS 中的主要存储服务，EBS Volume 可以附加到某个 EC2 Instance 上，在 Instance 上看就是一个未经过格式化的块设备，与其他物理硬盘一样。

EBS 最大的特性是其生命周期不受到 Instance 的影响。可以将 EBS Volume 从某个 Instance 分离后，再次附加到另一个 Instance，其数据依旧是保留的。更进一步，多个 Instance 也可以同时使用一个 EBS，见 [**Multi-Attach**]()。

EBS Volume 是 AZ 级别的存储，也就是说，EBS 只能由同 AZ 的 Instance 使用。如果需要跨 AZ 使用 EBS，需要先创建 Snapshot，然后从 Snapchat 创建新的 EBS 使用。

## 2 EBS Volume Type

EBS 提供多种类型的 Volume，已提供不同的性能与定价。总体上分为：

* SSD - 主要性能属性为 iops
* HHD - 主要的性能属性为 throughput
* Previous generation - 上一代的 Volume，性能比较低

### 2.1 SSD

SSD 类型的 EBS 又可以分为几个类别：

* General Purpose SSD - 性价比高的类型，大多数 Instance 使用这种类型。具体包括
  
  * **gp3** - 无论 Volume 的大小，gp3 提供了标准的性能：3000 IOPS 与 125 MiB/s。当然，EBS–optimized Instance 可以提高更高的性能。

  * **gp2** - gp2 的性能与使用情况有关，基于令牌桶的实现：

    {{< image src="img1.png" height=200 >}}
  
    * 其令牌被称为 IO 积分。IO 需要使用 IO 积分。
    * 当 IO 积分用完（IO 突增情况下），最大的 IOPS 会被限制在基准 IOPS 性能。
  
    而 gp2 的基准 IOPS 性能与 Volume 大小有关：Volume 越大，基准 IOPS 性能越高，IO 积分的累计也越快。也就是说，Volume 越大，应对高性能 IO 的能力也越好。

* Provisioned IOPS SSD - 高性能存储，适用于关键性的负载。具体包括：

  * **io2 Block Express** - 提供最高性能，但是只允许特定的 Instance 使用。

  * **io2** - 满足 I/O 密集型工作负载的需要。

  * **io1** - 与 io2 类似，区别在于提供了更高的耐用性。

具体各个类型的的性能比较见文档：[**固态硬盘 (SSD)**](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/ebs-volume-types.html#solid-state-drives)。

### 2.2 HDD

HDD 提供低成本，大容量的块存储。具体包括：

* Throughput Optimized HDD (st1) - 低成本 HDD。

  st1 的性能也是基于令牌桶模型。Volume 大小决定了基准性能，以及 IO 积分累计速率。
  
* Cold HDD (sc1) - 最低成本 HDD，性能也最低。

  sc1 的性能也是基于令牌桶模型。Volume 大小决定了基准性能，以及 IO 积分累计速率。

{{< admonition note Note>}}
CloudWatch 中提供的 EBS `BurstBalance` 指标来监控 gp2、st1 和 sc1 卷的突增存储桶水平。
{{< /admonition >}}

## 3 EBS 使用

### 3.1 Volume State

Volume State 描述了 Volume 的可用性，包括：

* Creating - Volume 正在创建中
* Available - Volume 还未被 Attach
* In-use - Volume 已被 Attach
* Deleting - Volume 正在删除中
* Deleted - Volume 已经被删除
* Error - Volume 底层硬件出现错误，并且数据不可恢复

### 3.2 Attach Volume

Attach Volume 到一个 Instance 代表了 Instance 上插入了一个块设备。

当然，只有处于 Available 状态的 Volume 才能够被使用。并且，Instance 需要与 Volume 处于同一个 AZ。

### 3.3 Multi Attach

Multi Attach 允许将单个 Volume（目前仅仅支持 io1 与 io2 类型）挂载当同一个 AZ 下的多个 Instance。

Multi Attach 下的每个 Instance 都对 Volume 有着完全的读写权限。因此，可以实现让不同 Instance 的多个应用并发写同一个 Volume。

不过，Multi Attach 有着许多的 [**注意事项和限制**](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/ebs-volumes-multi.html#considerations)。

### 3.4 Detach Volume

与 Attach Volume 相反，Detach Volume 从 Instance 上分离存储。当然，Instance 操作系统层面需要先进行 umount 块设备。

如果 Detach 了处于挂载状态的 Volume，那么 Volume Attachment 可能处于 `busy` 状态。直到 Volume 被 umount。

```json
"Volumes": [
    {
        "AvailabilityZone": "us-west-2b",
        "Attachments": [
            {
                "AttachTime": "2016-07-21T23:44:52.000Z",
                "InstanceId": "i-fedc9876",
                "VolumeId": "vol-1234abcd",
                "State": "busy",
                "DeleteOnTermination": false,
                "Device": "/dev/sdf"
            }
        ...
    }
]
```

如果 Volume Attachment 长时间处于 `detaching` 状态，可以选择使用 `Force Detach` 强制执行分离操作。这种情况下，Instance 没有机会来回写文件系统 cache 与 metadata，可能会导致出现数据丢失与坏块。因此，下一次使用时必须进行文件系统检查与修复流程。

### 3.5 Delete Volume

Delete Volume 表示完全删除某个 Volume，已经 Volume 的所有数据。该 Volume 再也不能被 Attach。

### 3.6 Restore Root Volume

Restore Root Volume 指的是：将 Instance 使用的 Root Volume 还原到初始状态或者特定 Snapshot 状态。

通过创建一个 Volume Replacement Task 来发起替换。通过 Task 可以监控替换的进度与结果。

## 4 EBS 监控

### 4.1 状态检查

EBS 提供周期性的 Volume 状态检查，以探测存储是否损坏。Volume 状态检查是自动执行的，每 5min 运行一次。

{{< image src="img2.png" height=250 >}}

* **IO 状态** - 每 5min 对 Volume 进行硬件检查。如果发现具有潜在不一致性时，会变为 `impaired` 状态。
  
  检查失败后，默认会禁用 Volume 的 IO，并发送一个 IO 禁用事件。当然，用户可以手动立即重新打开 IO。

  如果 Volume 启用了 Auto-Enable IO 功能，那么检查失败也不会禁用 IO，仅仅会发送事件。

* **IO 性能** - 每 1min 还会进行一次 IO 性能检查（CloudWatch 每 5min 收集数据）。如果 Volume 的性能低于预期，那么会提醒用户。
  
  > IO 性能检查仅仅对于 `io`、`io2` 和 `gp3` 类型有效。

状态检查的结果如下：

| 状态              | IO 状态           | IO 性能  |
| ----------------- | ----------------- | -------- |
| ok                | 正常              | 正常     |
| warning           | 正常              | 低于预期 |
| impaired          | 正常，但是禁用 IO | 停滞     |
| insufficient-data | 数据不足          | 数据不足 |

### 4.2 Volume 事件

上面说到，Volume 出现异常时，就会创建一个 Volume 事件来指明故障的原因。

一个 Volume 的事件类型包括：

* Awaiting Action: Enable IO - 数据具有潜在一致性问题，IO 被禁用。
* IO Enabled - 用户明确启用 IO
* IO Auto-Enabled - 数据具有潜在一致性问题，因为设置了 Auto-Enable IO 功能，IO 自动启用
* Normal - 性能正常
* Degraded - 性能低于预期
* Severely Degraded - 性能达达低于预期
* Stalled - 性能收到严重影响

## 5 EBS Snapshot

### 5.1 原理

同一个 Volume 的多个 Snapshot 之间，使用增量备份的方式保存数据。这意味着，除了第一个 Snapshot 需要复制完整的数据，其他的 Snapshot 都仅仅保存增量数据。

例如下图，Snap A 保存全量数据，而 Snap B 仅仅保存相对于 Snap A 修改的数据，不变的数据直接引用的 Snap A。相比于全量备份需要 20GiB 数据，Snapshot 仅仅使用了 14GiB 的数据。

{{< image src="img3.png" height=400 >}}

同样，删除 Snapshot 时，仅仅删除只有该 Snapshot 专门使用的数据，被其他 Snapshot 引用的数据不会被删除。

例如下图，Snap A 被删除时，仅仅只会删除 Snap A 中未被其他 Snap 引用的 4GiB 数据。被引用的 6GiB 数据会被分配给 Snap B。

{{< image src="img4.png" height=400 >}}

所有 Snapshot 的数据都是保存在 S3 上。可以想象到，都是元数据来索引各个不同的 Snapshot 以及不同的数据块。

### 5.2 管理

#### 5.2.1 使用

与普通的创建 Volume 流程一样，当创建 Volume 时指定 Snapshot ID 就可以基于 Snapshot 创建一个 Volume。

如果需要跨 AZ 使用 Snapshot，需要先将 Snapshot 复制到另一个 AZ，然后在另一个 AZ 基于 Snapshot 创建 Volume。

{{< admonition note "初始化 Volume">}}
还原后的 Volume 会在使用时才从 S3 下载真正的存储块并写入。因此，还原后首次访问每个数据库的 IO 会有很大的延迟。

为了避免性能下降，用户可以在还原后，手动进行一次全盘的读操作。也称为 Volume 初始化。
{{< /admonition >}}

### 5.2.2 复制

复制 Snapshot 指定是将 Snapshot 数据复制到另一个 AZ。复制的 Snapshot 是完整的数据副本，而不是增量数据。

如果是 Encrypted Snapshot，那么新的 S3 上的数据会使用新的密钥进行加密。S3  在复制期间保护传输中的数据。

### 5.2.3 归档

为了满足不同的需求，Snapshot 的存储分为了两种：

* Standard tier - 默认情况下，创建的 Snapshot 数据都存储在 Standard tier 中，并使用增量备份的方式。
* Archive tier - 选择归档 Snapshot 后，会将完成的数据副本移动到 Archive tier。

Archive tier 设计用于保存不经常访问，但是需要长时间保存的 Snapshot，其存储成本降低 75%。需要注意，归档不会使用增量备份，所有的 Snapshot 都是完整的数据副本。

当要使用归档后的 Snapshot 时，需要先将其还原到 Standard tier（最长需要 72h）。在还原时，可以选择是临时还原或者永久还原，区别在于还原后是否删除归档的 Snapshot。

## 6 EBS 功能

### 6.1 Modify Volume

EBS Volume 支持动态修改 Volume Type、Size、IOPS 以及 Throughput，不需要 Umount 文件系统以及 Detach Volume。

关于修改 Volume Type 的限制条件，参考[**官方文档**](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/modify-volume-requirements.html)。

修改 Volume Type 的大致流程如下：

1. 发起 Modify Volume 请求，指定需要修改的 Volume type、Size 等参数。

2. 发起后，Volume 将会依次进入 `modifying` -> `optimizing` -> `completed ` 状态。
   
   `optimizing` 状态表示 Volume 正在修改参数的过程中。Size 的修改往往只需要几秒钟，IOPS 的修改可能需要几分钟或者几小时。

   通过 API 或 CloudWatch，都可以观测到修改的进度。以 AWS CLI 为例，通过 `aws ec2 describe-volumes-modifications` 命令可以观察到修改的进度：

   ```json
   // aws ec2 describe-volumes-modifications --volume-ids vol-11111111111111111 vol-22222222222222222
   {
      "VolumesModifications": [
          {
              "TargetSize": 200,
              "TargetVolumeType": "io1",
              "ModificationState": "modifying",
              "VolumeId": "vol-11111111111111111",
              "TargetIops": 10000,
              "StartTime": "2017-01-19T22:21:02.959Z",
              "Progress": 0,
              "OriginalVolumeType": "gp2",
              "OriginalIops": 300,
              "OriginalSize": 100
          },
          {
              "TargetSize": 2000,
              "TargetVolumeType": "sc1",
              "ModificationState": "modifying",
              "VolumeId": "vol-22222222222222222",
              "StartTime": "2017-01-19T22:23:22.158Z",
              "Progress": 0,
              "OriginalVolumeType": "gp2",
              "OriginalIops": 300,
              "OriginalSize": 1000
          }
      ]
    }
   ```

   其中，`Progress` 以百分比的形式展示出修改的进度。

3. AWS 仅仅负责修改 Volume 底层的参数。当修改 Volume Size 后（变为 `optimizing` 状态），用户必须手动扩容文件系统的大小。
   
   不同的文件系统有着不同的扩容命令，例如 ext 文件系统使用 `resize2fs` 命令来进行扩容。

### 6.2 EBS 加密

EBS 加密使用 [**KMS**](../kms/) 服务来对保存的数据进行加密，所有数据都会在加密后才保存到硬件上。并且，相关的 Snapshot 以及数据复制都会在加密后才会传输或保存。

{{< admonition note Note>}}
为了能够调用 KMS 服务，相关 EC2 Instance 必须授予 KMS 相关的权限。
{{< /admonition >}}

加密和解密对于用户都是透明的：数据写入时会加密并保存到磁盘，读取数据时会先解密后读取。

#### 6.2.1 原理

EC2 使用信封加密的方式来加密数据：使用 KMS Key 来加解密 Data Key，加密后的 Data Key 会与加密后的数据一起保存到磁盘上。

EBS 使用 KMS 的大致原理如下：

{{< image src="img6.png" height=270 >}}

## 参考

* 官方文档：[**Amazon Elastic Block Store**](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/AmazonEBS.html)
