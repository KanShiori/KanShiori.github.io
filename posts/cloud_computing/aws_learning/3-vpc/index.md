# AWS 学习 - 3 - VPC


## 1  VPC 与 Subnet

首先，云上的网络是由 [VPC]^(Virtual Private Cloud) 为基础的，**VPC 是针对 Region 而言的。**

VPC 需要通过 CIDR 块来指定其的地址范围，相当于给局域网指定的网段。

然而，VPC 仅仅是一个范围表示，需要分为多个地址范围更小的 **`Subnet`**。而 Instance 启动就是会加入一个 Subnet，这样才能完成网络的连接。

**Subnet 是针对 AZ 而言的**，也就是说 Subnet 无法跨 AZ 覆盖。

### 1.1 Default 与 Nondefault
对于每个 Region，都会自动为账户创建一个 **`Default VPC`**。

Default VPC 会默认包含以下组件：
* **Internet Gateway**
* **Default Network ACL**
* **每个 AZ 一个 `Default Subnet`**

当启动 instance 没有指定 VPC 时，就会默认使用 Default VPC 与 Default Subnet。

相反，你手动创建的 VPC 与 Subnet 都称为 **`Nondefault VPC`** 或者 **`Nondefault Subnet`**。

### 1.2 Public 与 Private

能够连接 Internet Gateway 的 Subnet 称为 **`Public Subnet`**。反之就称之为 **`Private Subnet`**。

同样，如果 VPC 流量可以转发到 Internet Gateway，表明可以连接到外网，称为 **`Public VPC`**。反之，流量无法通向外网，称为 **`Private VPC`**。


## 2 IP 地址类型
在 Instance 看到的 IP 都是 VPC 地址范围内的 **`Private IP`**。

如果 Subnet 配置了 *Auto-assign public IPv4 address*，或者创建 Instance 时指定分配，那么 Instance 会绑定 **`Public IP`**。

Instance 创建时或者创建后，可以给其分配 **`Elastic IP`**。

{{< admonition note "公网 IP 都是虚拟的">}}
Instance 能够看到的都是 Private IP，或者说 Network Interface 绑定的都是 Private IP。
{{< /admonition >}}

**Public IP 与 Elastic IP 都是公网 IP，能够通过 Internet Gateway 访问外网，外网也能通过该公网 IP 直接访问 Instance。**

Public IP 与 Elastic IP 的区别在于：
* Public IP 是随机的，无法复用，并且 Instance 重启后就会变化。
* Elastic IP 是固定的，可以解绑后再次分给某个 Instance 使用，也不和 Instance 生命周期绑定。

如果 VPC 有着 IPv6 地址范围，那么 Instance 能够被分配 IPv6 地址，也是公网 IPv6 地址。

{{< admonition tip Tip>}}
下面所说的 IP 都是指 IPv4，IPv6 会显式说明。
{{< /admonition >}}


## 3 路由控制
### 3.1 Route Table
VPC 内的路由规则都是由一个个 **`Route Table`** 控制。**Subnet 可以绑定一个 Route Table，那么 Subnet 的进出流量都会由该 Route Table 控制。**

VPC 创建后，会自动创建 **`Main Route Table`**，**所有未绑定 Route Table 的 Subnet 默认会使用 Main Route Table**。自己创建的 Route Table 就称为 **`Custom Route Table`**。

按照 Route Table 关联的对象不同，又分为：
* **`Subnet Route Table`** ：关联到 Subnet
* **`Gateway Route Table`** ：关联到 Internet Gateway
* **`Local Gateway Route Table`** ：关联到 Local Gateway

### 3.2 Route Rule
Route Table 的 Rule 类似于：
Destination | Target | Status | Propagated
-|-|-|-
172.31.0.0/16 | local        | Active | No
0.0.0.0/0     | igw-3518715d | Active | No

路由表中每一项称为 Route，包含以下属性：
* **Destination** ：要匹配流量的目的地址范围，包括
  * 0.0.0.0/0 ：CIDR 地址范围
  * pl-*id* ：Prefix Lists
* **Target** ：转发的目标，包括
  格式 | 目标
  -|-
  local             | Subnet 内部
  igw-*id*          | Internet Gateway
  nat-gateway-*id*  | NAT Device
  vgw-*id*          | Virtual Private Gateway
  lgw-*id*          | Outposts Local Gateway
  cagw-*id*         | Carrier Gateway
  pcx-*id*          | VPC Peering Connection
  eigw-*id*         | Egress-only Internet Gateway
  tgw-*id*          | Transit Gateway
  eni-*id*          | Network Interface
  vpce-*id*         | Gateway Endpoint
  vpc-endpoint-*id* | VPC Endpoint（网关路由表使用）
* **Propagated** ：是否允许 Virtual Private Gateway 将路由传播到 Route Table
  
{{< admonition note "Route Table？">}}
可以发现，并不支持指定地址或者指定 Subnet 的路由，因为 VPC 下默认所有 Subnet 之间都是互通的。

为此，可以理解为 **Route Table 是用于 Subnet 内将流量路由到 AWS 资源的。**
{{< /admonition >}}


## 4 流量控制
目前，包含两种方式进行流量的控制：
* Security Group 安全组
* Network Access Control List，又称 Netowrk ACLs

### 4.1 Security Group
**`Security Group`** 是**针对 Network Interface 而言的**，即网卡进出流量的白名单。
{{< admonition note "Instance Security Group">}}
常说的 Instance 配置 Security Group，其实就是给 Instance 主网卡 eth0 配置。
{{< /admonition >}}

#### 4.1.1 Rule
对于 Inbound 流量，默认会拒绝所有主动访问流量，通过配置 Security Group 来打开限制。

对于 Outbound 流量，如果没有配置任何规则，那么默认允许所有。但是一旦配置了规则，那么默认就是拒绝其他所有。

一个 Rule 类似于：
Type | Protocol | Port Range | Destination/Source | Description
-|-|-|-|-
SSH | TCP | 22 | 0.0.0.0/0 | Allow SSH 
* **Type** ：常用的端口，或者自己配置
* **Protocol** ：协议，TCP/UDP/ICMP
* **Port Range** ：允许的端口范围，ALL 表示全部端口
* **Destination/Source** ：匹配流量的 目的/源 地址范围

  对于 Inbound Rule 就是 Source，对于 Outbound Rule 就是 Destination

特别的注意，Security Group 是有状态的，因此对于 Outbound 流量，自动会允许其回包。
{{< admonition note "类似 conntrack">}}
这个行为其实就是类似于 conntrack，端口限制型。
{{< /admonition >}}


### 4.2 Network ACL
**`Network ACL`** 是 VPC 的额外完全层，可**以用作防火墙来控制一个或多个 Subnet 的流量**。

与 Security Group 不同，Network ACL 是属于 VPC 下的，**当创建 VPC 时，会自动创建一个默认的 Network ACL，未指定 Network ACL 的 Subnet 就会关联该 ACL。**

一个 Subnet 可以关联一个同 VPC 下的 Network ACL，其 Subnet 的出入流量都会受到该 Network ACL 的控制。
{{< find_img "img4.png" >}}

{{< admonition note "Security Group 与 Network ACL 的区别">}}
要明确，它们之间最大区别在于面向对象不同，因此其保护的范围是不同的。
* Security Group 面向 Network Interface
* Network ACL 面向 Subnet
{{< /admonition >}}

#### 4.2.1 Rule
默认的 Network ACL 允许所有的出入流量，新创建的 Network ACL 默认拒绝所有出入流量。

Network ACL 的 Rule 类似于：
Rule Number | Type | Protocol | Port Range | Destination/Source | Allow/Deny
-|-|-|-|-|-
100 | Custome TCP | TCP | 123 | 0.0.0.0/0 | Allow
\*  | All traffic | All | All | 0.0.0.0/0 | Deny
* **Rule Number** ：规则的编号
* **Type** ：常用的端口或者自定义
* **Protocol** ：传输层协议
* **Port Range** ：匹配的端口范围
* **Destination/Source** ：匹配的 目的/源 地址范围
  
  对于 Inbound Rule 就是 Source，对于 Outbound 就是 Destination
* **Allow/Deny** ：允许还是禁止

匹配流量时，会按照 Rule Number 从小到大的顺序进行匹配，如果匹配上某一条 Rule 就不再匹配其他的 Rule 了。
{{< admonition tip "预留 Rule Number">}}
建议以 100 间隔来添加 Rule，为中间预留可插入的编号。
{{< /admonition >}}

Network ACL 是静态的（无状态的），仅仅就来过滤出入流量。

### 4.3 [WIP] Network Firewall


## 5 DNS
对每个 VPC 会有一个 Route 53 Resolver 作为 DNS 服务器，会每个 Instance 的每个 IP 地址（不包括 IPv6）提供 DNS 主机名，包括内网 IP 与公网 IP。
{{< admonition tip "关闭 DNS 解析">}}
可以通过设置 VPC 的 `enableDnsSupport` 属性来关闭 DNS 解析功能。
{{< /admonition >}}

Private DNS 主机名支持同 VPC 解析到对应的 IP，格式为：
* 对于 Region us-east-1，格式为 `ip-<private_ip>.ec2.internal`
* 其他 Region 形式为 `ip<private_ip>.<region>.compute.internal`

Public DNS 主机名支持外网访问，格式为：
* 对于 Region us-east-1，格式为 `ec2-<public-ip>.compute-1.amazonaws.com`
* 其他 Region 格式为 `ec2-<public-ip>.region.compute.amazonaws.com`

{{< admonition tip "不提供 Public DNS">}}
可以通过设置 VPC 的 `enableDnsHostnames` 属性来不为 Public IP 提供 DNS 主机名。
{{< /admonition >}}


## 6 VPC 与外网通信

### 6.1 Internet Gateway
**`Internet Gateway`** 用于将 VPC 的流量转发给外网，并将外网的主动访问转发给指定的 Instance。

要让 Instance 连接外网，需要以下操作：
1. Internet Gateway 关联到 VPC 上。
1. Route Table 能够将 Instance 的流量转发到 Internet Gateway。
1. Instance 有着公网 IP 地址，即 Public IP、Elastic IP 或 IPv6。
1. Security Group 与 Network ACL 不会拦截 Instance 的流量。

{{< find_img "img6.png" >}}

Instance 使用 Public IP 与 Elastic IP 连接外网时，Internet Gateway 的操作：
* 对于 Outbound 流量，Internet Gateway 会将数据包源地址从 Private IP 变为对应的 Public IP/Elastic IP 转发。
* 对于 Inbound 流量，Internet Gateway 会根据数据包源地址，匹配 Public IP/Elastic IP，然后将目的地址变为 Private IP 转发给 Instance。

{{< admonition note "Instance 只能看到 Private IP">}}
要明确，所谓的 Public IP 与 Elastic IP 都是针对于 Internet Gateway 设置的，因为 Instance 根本就无法知晓自己有着公网 IP 地址。

感觉就好像在路由器上设置了内网 IP 到外网 IP 的一对一映射。
{{< /admonition >}}

### 6.2 Egress-only Internet Gateway
当 Instance 有着 IPv6 地址时，能够通过 Internet Gateway 访问外网。同时，IPv6 也是暴露在公网的，外网可以主动通过 IPv6 访问 Instance。

使用 **`Egress-only Internet Gateway`** 可以**阻止外网主动访问 IPv6**。这表明你的 Instance 仅仅只能作为一个 Client，而不能作为 Server。

Egress-only Internet Gateway 当然是有状态的，能将 response 转发给对应的 Instance。

### 6.3 NAT Device
与实际的 NAT 设备一样，**`NAT Device`** 为 Instance 提供的 NAT 的功能，这样即使 Instance 只有 Private IP，也能通过 NAT Device 访问外网。

这表明你的 Instance 作为一个 Client，而不能作为 Server。

NAT Device 当然是有状态的，能将 response 转发给对应的额Instance。
{{< admonition note "直接看做路由器">}}
感觉可以将其直接看做路由器。
{{< /admonition >}}


## 7 VPC 与资源之间的的通信

### 7.1 VPC Peering

**`VPC Peering`** 可以**连接两个不同地址范围的 VPC，VPC 可以处于不同的 Region，甚至不同的 AWS 账户**。

当 VPC Peering 连接两个 VPC 后，VPC 内部的资源（Instance、RDS 等）可以直接访问到对端 VPC 的资源。
{{< find_img "img8.png" >}}

我们以连通网络的角度来看，使用 VPC Peering 连接 VPC 需要的工作：
* 物理连通性：由 VPC Peering 保证两个 VPC 之间 “物理” 上是连通的
* 网络连通性：通过设置 Route Table 来实现流量是能够成功被转发的，并配置 Security Group 与 Network ACL 不影响流量
* 寻址：通过各自 VPC 下资源的 IP 地址或者 DNS 主机名来寻址
  {{< admonition note "VPC 地址范围不能重叠">}}
  要通过 IP 地址寻址表明了，VPC 之间的地址范围不能重叠。
  {{< /admonition >}}

VPC Peering 相当于一个连接，因此为分为：
* Requester VPC: 发起 VPC Peering 连接请求的 VPC
* Accepter VPC：接受 VPC Peering 连接请求的 VPC

{{< admonition note "看做 tunnel">}}
把 **VPC Peering 看做一个 tunnel，将两个 VPC 连通**。但是不负责流量到达 VPC 内后的路由，因此需要配置 Route Table 让流量可以发到指定的 Instance。
{{< /admonition >}}

#### 7.1.1 VPC Peering 的状态流转
从发起请求开始，VPC Peering 会经过各个阶段：
{{< find_img "img9.png" >}}

### 7.2 Endpoint 与 Private Link
前面在 Route Table 看到，Route Table 控制将 Subnet 内的流量转发到 AWS 资源。但是 AWS 有着许多的服务，服务类型也不断出现着，特定的 Rule.Target 无法满足增长。

因此，你可以看到 Rule.Target 有一类 vpc-endpoint-*id*，这就是表明路由到一个 Endpoint。

**Endpoint 就是各式各样的 Service 的连接点的抽象**。通过 Endpoint，Instance 就可以直接使用 Private IP 访问各种 Service 了。
{{< admonition note "Service Network Interface">}}
可以将其理解为 Endpoint 就是 Service 的 "Network Interface"，有了 Endpoint 就可以将 Subnet 的流量发送给 Service。
{{< /admonition >}}

目前，Endpoint 分为三类：
* Gateway Endpoints
* Interface Endpoints
* Gateway Load Balancer Endpoints

其中，第一种流量会通过公网，而后两种不会通过公网，其实现技术称之为 Private Link。
{{< admonition note "分类依据的猜想">}}
这会分为三类我觉得是技术发展过程中遗留的，因为理论上可以统一。
{{< /admonition >}}

#### 7.2.1 Gateway Endpoint
使用 **`Gateway Endpoint`** **只能连接特定的 AWS 服务：S3 与 DynamoDB**。或者说，当你创建 Endpoint 是连接的是 S3 或者 DynamoDB 服务，可以选择使用 Gateway Endpoint。

通过以下步骤：
1. 创建 Endpoint，连接搭配 Prefix List 表示的 Service。
2. 配置 Route Table，是流量能够指向对应 Endpoint 的 vpce-*id*。

{{< find_img "img11.png" >}}

#### 7.2.2 Interface Endpoint
大多数 AWS Service 可以通过 **`Interface Endpoint`** 进行连接。创建 Interface Endpoint 相当于创建一个 Network Interface，有着 Private IP 与 Private DNS。

**然后就可以通过 Private IP 或者 Private DNS 来访问 Service。**

{{< admonition note "支持 S3 与 DynamoDB">}}
Interface Endpoint 是支持连接 S3 与 DynamoDB Service 的。
{{< /admonition >}}

{{< find_img "img12.png" >}}

#### 7.2.3 Gateway Load Balancer Endpoint
Gateway Load Balancer Endpoint 能够将流量路由到 Gateway Load Balancer，然后由 Load Balancer 负载均衡到下游的 Service。

创建 Endpoint 的称为 Service Consumer，下游的 Service 称为 Service Provider。

### 7.3 Endpoint Service
你也可以将自己的程序作为一个 **`Endpoint Service`**，**使得别人可以通过 Interface Endpoint 来访问你的服务。**

Endpoint Service 的入口是一个 LB：**Network Load Balancer** 或 **Gateway Load Balancer**。
{{< find_img "img13.png" >}}



## [WIP] 8 VPC 与个人网络的通信

### 8.1 VPN

### 8.2 Direct Connect


## [WIP] VPC 中的限制


## 参考
* [**VPC 官方文档**](https://docs.aws.amazon.com/zh_cn/vpc/latest/userguide/what-is-amazon-vpc.html)
