# AWS 学习 - 5 - VPC 与 Service 通信


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
{{< find_img "img2.png" >}}

## 2 Transit Gateway

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
