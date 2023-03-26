# AWS - VPC 基本概念


## 1  VPC 与 Subnet

云上的网络是由 [VPC]^(Virtual Private Cloud) 与 **`Subnet`** 组成的：

* <important>VPC 是针对 Region 级别</important>，其由一个 CIDR Block 来指定 IP 地址范围。
* <important>Subnet 是 VPC 网段的子网，AZ 级别资源</important>（无法跨 AZ 覆盖）。

EC2 Instance 启动会加入一个 Subnet，这样才能完成网络的连接。

{{< image src="img0.png" height=270 >}}

### 1.1 Default 与 Nondefault

对于每个 Region，都会自动为账户创建一个 **`Default VPC`**。Default VPC 默认包含一下资源：

* **Internet Gateway**
* **Default Network ACL**
* **每个 AZ 一个 `Default Subnet`**

当启动 Instance 没有指定 VPC 时，就会默认使用 Default VPC 与 Default Subnet。

相反，手动创建的 VPC 与 Subnet 都称为 **`Nondefault VPC`** 或者 **`Nondefault Subnet`**。

### 1.2 Public 与 Private

能够连接 Internet Gateway 的 Subnet 称为 **`Public Subnet`**。反之就称之为 **`Private Subnet`**。

同样，如果 VPC 流量可以转发到 Internet Gateway，表明可以连接到外网，称为 **`Public VPC`**。反之，流量无法通向外网，称为 **`Private VPC`**。

## 2 IP 类型

* **`Private IP`** - VPC 范围内的内网 IP。

  Instance 操作系统层面能够看到的都是 Private IP，Public IP 如要从 AWS 服务中读取。

* **`Public IP`** - “临时”的公网 IP，Instance 重启后重新分配。
  
  如果 Subnet 配置了 **"Auto-assign public IPv4 address"** 功能，或者创建 Instance 时指定，那么 Instance 会绑定 Public IP。

* **`Elastic IP`** - 固定生命周期的公网 IP

  Elastic IP 独立于 Instance，通过绑定与解绑与 Instance 进行交互。

如果 VPC 有着 IPv6 地址范围，那么 Instance 能够被分配 IPv6 地址，也是公网 IPv6 地址。

{{< admonition tip Tip>}}
下面所说的 IP 都是指 IPv4，IPv6 会显式说明。
{{< /admonition >}}


## 3 Route Table

VPC 内的路由规则都是由 **`Route Table`** 控制。

* **`Main Route Table`** - VPC 创建后默认创建的全局路由表，所有未绑定 Route Table 的 Subnet 默认会使用 Main Route Table
  
* **`Custom Route Table`** - 自定义的 Route Table

如果一个 Subnet 绑定了 Custom Route Table，那么进出的流量就由该路由表控制，否则就由 Main Route Table 控制了。

{{< image src="img1.png" height=270 >}}

按照 Route Table 关联的对象不同，又分为：

* **`Subnet Route Table`** ：关联到 Subnet
  
* **`Gateway Route Table`** ：关联到 Internet Gateway

* **`Local Gateway Route Table`** ：关联到 Local Gateway

### 3.1 Route Rule

Route Table 由多个 Route Rule 组成。

| Destination   | Target       | Status | Propagated |
| ------------- | ------------ | ------ | ---------- |
| 172.31.0.0/16 | local        | Active | No         |
| 0.0.0.0/0     | igw-3518715d | Active | No         |

* **Destination** ：要匹配流量的目的地址范围，包括
  
  * 0.0.0.0/0 ：CIDR 地址范围
  * pl-*id* ：Prefix Lists
  
* **Target** ：转发的目标，包括
  
  | 格式              | 目标                           |
  | ----------------- | ------------------------------ |
  | local             | Subnet 内部                    |
  | igw-*id*          | Internet Gateway               |
  | nat-gateway-*id*  | NAT Device                     |
  | vgw-*id*          | Virtual Private Gateway        |
  | lgw-*id*          | Outposts Local Gateway         |
  | cagw-*id*         | Carrier Gateway                |
  | pcx-*id*          | VPC Peering Connection         |
  | eigw-*id*         | Egress-only Internet Gateway   |
  | tgw-*id*          | Transit Gateway                |
  | eni-*id*          | Network Interface              |
  | vpce-*id*         | Gateway Endpoint               |
  | vpc-endpoint-*id* | VPC Endpoint（网关路由表使用） |

* **Propagated** ：是否允许 Virtual Private Gateway 将路由传播到 Route Table
  
{{< admonition note "Route Table？">}}
可以发现，并不支持指定地址或者指定 Subnet 的路由，因为 VPC 下默认所有 Subnet 之间都是互通的。

为此，可以理解为 **Route Table 是用于 Subnet 内将流量路由到 AWS 资源的。**
{{< /admonition >}}

## 4 Elastic Network Interface

[ENI]^(Elastic Network Interface) 是 VPC 网络中的虚拟网卡。无论 Private IP 还是 Public IP，其都是将其绑定到 ENI 上的。

ENI 包含以下属性：

* 一个或多个 Private IP
* 一个或多个 Public/Elastic IP，每个 Public IP 都要对应一个 Private IP
* 一个或多个 IPv6 地址
* MAC 地址
* 一个或多个 Security Group

每个 Instance 的第一个 ENI 称为 **`Primary Network Interface`**，可以来自于：

* 自动创建 - Instance 创建时自动创建的 ENI
* 指定 - Instance 创建时指定已经存在的 ENI 作为 Primary ENI

每个 Instance 可以被附加上多个 ENI，其数量上限取决与 Instance 类型，具体见 [**文档**](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)。

{{< admonition note Note>}}
大多数情况在描述 IP 与 Security Group 时都是直接说 Instance 时，而会忽略 ENI。
{{< /admonition >}}

## 5 Security Group

**`Security Group`** 用于定义进出 ENI 数据包的白名单。分为 Inbound rules 与 Outbound rules。

* Inbound rules - 入 ENI 数据包的白名单。

  <important>默认规则为拒绝所有主动访问流量</important>。需要通过配置 Inbound rules 来一个个打开配置。

* Outbound rules - 出 ENI 数据包的白名单。

   <important>默认规则为允许所有出的流量</important>。一旦配置了规则了，默认会为拒绝所有出的流量。

{{< image src="img2.png" height=270 >}}

一个 Rule 类似于：

| Type | Protocol | Port Range | Destination/Source | Description |
| ---- | -------- | ---------- | ------------------ | ----------- |
| SSH  | TCP      | 22         | 0.0.0.0/0          | Allow SSH   |

* **Type** ：常用的端口，或者自己配置
* **Protocol** ：协议，TCP/UDP/ICMP
* **Port Range** ：允许的端口范围，ALL 表示全部端口
* **Destination/Source** ：匹配流量的 目的/源 地址范围

  对于 Inbound Rule 就是 Source，对于 Outbound Rule 就是 Destination

特别的注意，<important>Security Group 是有状态的</important>：

* 对于 Outbound 流量，自动允许对应的 Inbound 流量
* 对于 Inbound 流量，自动允许对应的 Outbound 流量

{{< admonition note "\"对应的\"意思">}}
所有 “对应” 就是指 Source IP/Port 与 Destination IP/Port 相反。类似于端口限制型的 NAT。
{{< /admonition >}}

## 6 Network ACL

**`Network ACL`** 用于控制出入 Subnet 的流量，也分为 Inbound rules 与 Outbound rules。

* Inbound rules - 入 Subnet 数据包的过滤规则，支持 Allow 与 Deny 规则。

* Outbound rules - 出 Subnet 数据包的过滤规则，支持 Allow 与 Deny 规则。

当创建 VPC 时，会自动创建一个默认 ACL，而用户可以自己创建 ACL 并绑定到 Subnet 上。

* 默认 ACL - VPC 创建时自动创建，默认允许所有出入流量。
  
  未绑定 ACL 的 Subnet 自动关联到默认 ACL。

* 自定义 ACL - 用户手动创建的 ACL，默认拒绝所有出入流量。

{{< image src="img3.png" height=240 >}}

与 Security Group 不同，Network ACL 是无状态的，出入数据包都会完全经过 Inbound/Outbound rules 的过滤。

Network ACL 的 Rule 类似于：

| Rule Number | Type        | Protocol | Port Range | Destination/Source | Allow/Deny |
| ----------- | ----------- | -------- | ---------- | ------------------ | ---------- |
| 100         | Custome TCP | TCP      | 123        | 0.0.0.0/0          | Allow      |
| \*          | All traffic | All      | All        | 0.0.0.0/0          | Deny       |

* **Rule Number** ：规则的编号
* **Type** ：常用的端口或者自定义
* **Protocol** ：传输层协议
* **Port Range** ：匹配的端口范围
* **Destination/Source** ：匹配的 目的/源 地址范围，对于 Inbound Rule 就是 Source，对于 Outbound 就是 Destination
* **Allow/Deny** ：允许还是禁止

匹配流量时，会按照 Rule Number 从小到大的顺序进行匹配，如果匹配上某一条 Rule （无论 Allow 或 Deny）就不再匹配其他的 Rule 了。

{{< admonition tip "预留 Rule Number">}}
建议以 100 间隔来添加 Rule，为中间预留可插入的编号。
{{< /admonition >}}

## 5 VPC 中的 DNS

对每个 VPC 会有一个 Route 53 Resolver 作为 DNS 服务器，会每个 Instance 的每个 IP 地址（不包括 IPv6）提供 DNS 主机名，包括内网 IP 与公网 IP。

Private DNS 主机名支持同 VPC 解析到对应的 IP，格式为：
* 对于 Region us-east-1，格式为 `ip-<private_ip>.ec2.internal`
* 其他 Region 形式为 `ip<private_ip>.<region>.compute.internal`

Public DNS 主机名支持外网访问，格式为：
* 对于 Region us-east-1，格式为 `ec2-<public-ip>.compute-1.amazonaws.com`
* 其他 Region 格式为 `ec2-<public-ip>.region.compute.amazonaws.com`

{{< admonition tip "关闭 DNS 解析">}}
可以通过设置 VPC 的 `enableDnsSupport` 属性来关闭 DNS 解析功能，通过设置 VPC 的 `enableDnsHostnames` 属性来不为 Public IP 提供 DNS 主机名。
{{< /admonition >}}

## 参考
* [**VPC 官方文档**](https://docs.aws.amazon.com/zh_cn/vpc/latest/userguide/what-is-amazon-vpc.html)
