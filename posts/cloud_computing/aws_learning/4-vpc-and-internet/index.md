# AWS 学习 - 4 - VPC 与 Internet 通信


## 1 Internet Gateway

**`Internet Gateway`** 用于将 VPC 的流量转发给外网，并将外网的主动访问转发给指定的 Instance。

使用 Internet Gateway 的**必要操作**：
1. 创建 Internet Gateway 并关联到 VPC 上。
2. Instance 拥有公网 IP 地址：Public IP、Elastic IP 或者 IPv6。
3. Route Table 能够将 Instance 的流量转发到 Internet Gateway。
4. Security Group 与 Network ACL 不会拦截 Instance 的流量。

{{< find_img "img1.png" 1 >}}

Internet Gateway 内部会维护一份 Public IP 与 Private IP 的映射关系，其原理可以简单理解为：
* 对于 Outbound 流量，Internet Gateway 会将数据包源地址从 Private IP 变为对应的 Public IP/Elastic IP。
* 对于 Inbound 流量，Internet Gateway 会根据数据包源地址，匹配 Public IP/Elastic IP，然后将目的地址变为 Private IP 转发给 Instance。

{{< admonition note "Instance 只能看到 Private IP">}}
要明确，所谓的 Public IP 与 Elastic IP 都是针对于 Internet Gateway 设置的，因为 Instance 根本就无法知晓自己有着公网 IP 地址。

感觉就好像在路由器上设置了内网 IP 到外网 IP 的一对一映射。
{{< /admonition >}}

## 2 Egress-only Internet Gateway

当 Instance 有着 IPv6 地址时，能够通过 Internet Gateway 访问外网。同时，IPv6 也是暴露在公网的，外网可以主动通过 IPv6 访问 Instance。

使用 **`Egress-only Internet Gateway`** 可以**阻止外网主动访问 IPv6**。这表明你的 Instance 仅仅只能作为一个 Client，而不能作为 Server。

Egress-only Internet Gateway 当然是有状态的，能将 response 转发给对应的 Instance。

## 3 NAT Device

**`NAT Device`** 允许 Private Subnet 中的 Instance 访问 Internet，NAT Device 会将数据包的源地址 IP 替换为 NAT Device 的 Elastic IP。

NAT Device 是有状态的，当向实例发送回复时，会自动进行目标地址转换，并转发给指定的 Instance。

{{< find_img "img3.png" >}}
