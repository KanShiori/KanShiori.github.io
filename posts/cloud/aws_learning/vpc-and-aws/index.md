# AWS - VPC 与 Service 通信


## 1 VPC Peering

**`VPC Peering`** 用于<important>连接两个不同地址范围的 VPC，VPC 可以处于不同的 Region，甚至不同的 AWS 账户</important>。

当 VPC Peering 连接两个 VPC 后，VPC 内部的资源（Instance、RDS 等）可以直接访问到对端 VPC 的资源。
{{< find_img "img1.png" >}}

但是，VPC Peering 是**不具有传递性的**。以上图来说，VPC1 是无法通过 VPC2 与 VPC3 通信的。

### 1.1 使用 VPC Peering

使用 VPC Peering 连接 VPC 需要的工作：
1. 由源端 VPC **发起 VPC Peering**，连接到对端 VPC。
   
   发起的称为 **`Requester VPC`**。

2. 对端 VPC **接受 VPC Peering**。
   
   对端的称为 **`Accepter VPC`**。

3. **配置 Route Table**，将对方 VPC 的数据包发往 VPC Peering。
   
    | Destination   | Target       | Status | Propagated |
    | ------------- | ------------ | ------ | ---------- |
    | 172.31.0.0/16 | pcx-1353251  | Active | No         |

4. **配置链路上的 Network ACL 与 Instance 的 Security Group**，不影响流量。
  
{{< admonition note "VPC 地址范围不能重叠">}}
因为要通过 IP 地址寻址表明了，VPC 之间的地址范围不能重叠。
{{< /admonition >}}

### 1.2 VPC Peering 的状态流转

从发起请求开始，VPC Peering 会经过各个阶段：

{{< image src="img2.png" >}}

### 1.3 VPC Peering 构建步骤

使用 VPC Peering 联通两个 VPC 的大致流程如下：

1. 构建 VPC Peering
   
   VPC A 发起 VPC Peering 连接，而 VPC B 需要 Accept 该 VPC Peering。

2. 配置 Route Table

   两个 VPC 或 Subnet 的 Route Table 都需要添加一项指向对方 VPC 子网的路由项，Destination 为 Peering ID `pcx-xxxx`。

3. 配置 Network ACL
   
   确保互相访问的 Subnet 的 Network ACL 不会拦截对方网段的数据包。

4. 配置 Security Group
   
   确保互相访问的 Instance 的 Security Group 能够允许对方网段的数据包。

## 2 Transit Gateway

Transit Gateway（简称为 TGW）类似于一个路由器，可以连接多个 VPC 或者其他网络设备，基于网络层（IP）来路由数据包。

因此 TGW 更适用于多个 VPC 之间的互联，简化整体的网络架构。

{{< image src="img6.png" height=250 >}}

Transit Gateway 的基本概念如下：

* Attachment
  
  Attachment 意味着将 VPC 或者网络设备连接到 TGW。

* Transit Gateway Route Table
  
  TGW Route Table 决定了 TGW 如何转发数据报。

* Associations
  
  Associations 代表 Attachment 与 Route Table 的关联关系。一个 Attachment 与一个 Route Table 关联，Route Table 可以有多个 Attachment 复用。

* Route propagation
  
  对于部分 Attachment 那么支持动态将路由传播给 Route Table。

### 2.2 Attachment

Attachment 表示将 VPC 或者其他网络设备连接到 Transit Gateway 中，构建了一个双向联通的通道。

目前支持连接的网络设备包括：

  * VPC（本质是 Subnet）
  * Connect SD-WAN / 第三方网络设备
  * Direct Connect 网关
  * Transit Gateway Peering - 连接到另一个 Transit Gateway
  * VPN 连接

#### 2.2.1 VPC

Attach VPC 本质上是物理上联通 AZ 下的某个 Subnet 与 TGW，并通过配置 Subnet 的 Route Table 来实现网络层的联通。

{{< image src="img6.5.png" height=250 >}}

Attach VPC 的大致流程为：

1. 创建 Attachment，选择类型为 `VPC`，接着指定需要连接的 TGW 与 Subnet。

2. 修改 Subnet 的 Route Table，添加指向 TGW ID 的路由项。
   
    | Destination   | Target       | Status | Propagated |
    | ------------- | ------------ | ------ | ---------- |
    | 10.99.99.0/24 | tgw-0b20f516255224ace | Active | No|

{{< admonition note Note>}}
创建一个 Attachment 实际上会在对应的 Subnet 创建一个 ENI，而所有数据包会路由给 ENI 从而传输到 TGW。
{{< /admonition >}}

#### 2.2.2 Transit Gateway Peering

Transit Gateway Peering 支持在两个 TGW 之间路由流量，并且支持跨账户 TGW 之间的联通。

与 VPC Peering 类似，创建 TGW Peering 后，需要对端接受 Peering 连接，才能完成构建。TGW Peering 的构建仅仅表示物理网络上的联通，网络层的联通还是需要通过配置 TGW Route Table 实现。

{{< image src="img7.png" height=200 >}}

使用 TGW Peering 大致流程为：

1. 创建 Attachment，选择类型为 `Peering Connection`，指定对端的 TGW。

2. 修改两个 TGW 的 Route Table，添加指向 Attachment ID 的路由项。

    | Destination   | Target       | Status | Propagated |
    | ------------- | ------------ | ------ | ---------- |
    | 17.31.0.0/24 | tgw-attach-069fe98e37eb88937 | Active | No|

### 2.3 Route Table

与路由器一样，TGW 处理数据包都是基于 TGW Route Table。因此，当使用 TGW 互联各个网段时，关键就是需要配置 Route Table。

#### 2.3.1 Association

TGW Route Table 是一个静态的路由表，需要创建 Association 将 Route Table 与 Attachment 关联，从而使得 Route Table 生效。

因为 Route Table 是与 Attachment 关联的，所以 TGW 可以包含多个 Route Table，以针对不同的 Attachment 实现隔离。

当 TGW 创建时，会创建一个默认的 Route Table，没有构建 Association 的 Attachment 都会默认关联该 Route Table。

{{< admonition note Note>}}
本质上，路由是基于 Attachment 来判断的。TGW 收到某个 Attachment 的数据包，根据该 Attachment 关联的 Route Table 来决定从哪个 Attachment 出。
{{< /admonition >}}

#### 2.3.2 Route propagation

Route propagation 指的是：创建 Attachment 时，是否会将相关的路由信息自动同步到 Route Table，其路由项的 `Route type` 为 "Propagated"。

* 对于 VPC Attachment，会自动在 Route Table 创建指向 VPC 的 CIDR 的路由项。
  
    | CIDR   | Attachment ID       | Resource ID | Resource type | Route type|
    | ------------- | ------------ | ------ | ---------- | -- |
    | 10.0.0.0/24 | tgw-attach-053ad9590c5ceba7c | vpc-04547264eb685fb15 | VPC|Propagated|

* 对于 Direct Connect 网关，会通过 BGP 将本地路由器的路由表同步到 Route Table。
  
* 对于 Connect Attachment，与 Connect Attachment 关联的路由表会同步给 VPC 中的第三方虚拟设备。

#### 2.3.3 Prefix list references

Prefix list 是一个网段下的子网的集合，使用 Prefix list 可以将 CIDR 分组，从而简化路由表。

### 2.4 Multicast

TGW 也支持 Multicast，可以数据包发送给多个 Attachment。

## 3 Endpoint 与 Private Link

前面在 Route Table 看到，Route Table 控制将 Subnet 内的流量转发到 AWS 资源。但是 AWS 有着许多的服务，服务类型也不断出现着，特定的 Rule.Target 无法满足增长。

因此，你可以看到 Rule.Target 有一类 vpc-endpoint-*id*，这就是表明路由到一个 Endpoint。

**`Endpoint` 就是各式各样的 Service 的连接点的抽象**。通过 Endpoint，Instance 就可以直接使用 Private IP 访问各种 Service 了。

{{< admonition note "Service Network Interface">}}
可以将其理解为 Endpoint 就是 Service 的 "Network Interface"，有了 Endpoint 就可以将 Subnet 的流量发送给 Service。
{{< /admonition >}}

目前，Endpoint 分为三类：
* Gateway Endpoints
* Interface Endpoints
* Gateway Load Balancer Endpoints

其中，第一种流量会通过公网，而后两种不会通过公网，其实现技术称之为 **`Private Link`**。

### 3.1 Gateway Endpoint

使用 **`Gateway Endpoint`** **只能连接特定的 AWS 服务：S3 与 DynamoDB**。或者说，当你创建 Endpoint 是连接的是 S3 或者 DynamoDB 服务，可以选择使用 Gateway Endpoint。
{{< find_img "img3.png" >}}

使用 Gateway Endpoint 的步骤：

1. **创建 Endpoint**，创建时选择目标服务为 Gateway 类型的 DynamoDB 或 S3。

2. 配置 Route Table，是流量能够指向对应 Endpoint 的 vpce-*id*。

    | Destination   | Target       | Status | Propagated |
    | ------------- | ------------ | ------ | ---------- |
    | pl-6ea54007   | vpc-a2984bc5 | Active | No         |

3. Instance 通过 Service 域名来访问 Endpoint。

### 3.2 Interface Endpoint

大多数 AWS Service 可以通过 **`Interface Endpoint`** 进行连接。创建 Interface Endpoint 相当于创建一个 Network Interface，有着 Private IP 与 Private DNS。
{{< find_img "img4.png" >}}

使用 Interface Endpoint 的步骤：
1. **创建 Endpoint**，创建时选择需要的目标 AWS 服务。
2. Instance **通过 Private IP 或者 Private DNS 来访问 Service**。

{{< admonition note "支持 S3 与 DynamoDB">}}
Interface Endpoint 是支持连接 S3 与 DynamoDB Service 的。
{{< /admonition >}}

### 3.3 Gateway Load Balancer Endpoint

Gateway Load Balancer Endpoint 能够将流量路由到 Gateway Load Balancer，然后由 Load Balancer 负载均衡到下游的 Service。

创建 Endpoint 的称为 Service Consumer，下游的 Service 称为 Service Provider。

## 4 Endpoint Service

你也可以将自己的程序作为一个 **`Endpoint Service`**，**使得别人可以通过 Interface Endpoint 来访问你的服务。**

Endpoint Service 的入口是一个 LB：**Network Load Balancer** 或 **Gateway Load Balancer**。
{{< find_img "img5.png" >}}

## 参考

* [**AWS PrivateLink and VPC endpoints**](https://docs.aws.amazon.com/vpc/latest/privatelink/endpoint-services-overview.html)
