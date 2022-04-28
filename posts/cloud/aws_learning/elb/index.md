# AWS - ELB


## 1 LB 的基本模型

不同的 LB 在使用上是基本相同的，都依据以下的模型：
{{< find_img "img1.png" >}}

* Domain - LB 拥有着一个唯一的域名，Client 通过该域名来访问 LB。

    LB 的域名是单点的，但是域名对应多个 IP（每个 AZ 一个 IP）。也就是说，LB 所在 Region 的每个 AZ 都有着 LB 的处理程序，客户端的请求会被某个 AZ 的 LB 实例进行处理。

* Listener - LB 域名下扩展多个 Listener，用于根据请求的 Port 来进行转发。

    通过 Listener，Client 请求一个 LB 时可以根据请求的 Port 由不同的 Target 进行处理。
    
* Rule - 如果是 ALB，还可以给 Listener 添加多个 Rule，进一步根据请求信息（HTTP Path、HTTP Method 等）来进行转发。

* Target Group - 每个 Listener 对应于一个 Target Group，包含多个用于处理请求的 Target。

* Target - 处理请求的实例，最平常的就是一个 Instance。

## 2 Domain

LB 分为两种 Scheme：
* Internet-facing - 面向公网的 LB，具有 Public IP，需要部署在 Publich Subnet
* Internal - 内网 LB，仅仅具有 Private IP

无论哪种 LB，创建后都会分配一个唯一的 Domain，用作访问 LB 的入口。该 Domain 会被解析为其服务的多个 Subnet 的 Public/Private IP。

LB 的 Domain 格式为：
```
<LB-id>-<hash>.elb.<region>.amazonaws.com
```

## 3 Listener

Listener 将 Domain 访问根据 Protocol + Port 分为多个路由项。也就是说，访问 Domain 的请求根据 Protocol + Port 由不同的 Listener 来处理。

每个 Listener 需要指定：
* Protocol - 匹配的协议
  
  对于 ALB，Protocol 支持 HTTP 与 HTTPs。对于 NLB，Protocol 支持 TCP、TCP_UDP、TLS 与 UDP。

* Port - 匹配的 Port
  
* Default Action - 默认路由的 Target Group

### 3.1 Rule

对于 ALB，在 Listener 范围下还可以添加 Rule，进一步的根据信息添加路由项。例如，你可以创建一个 HTTP 80 端口的 Listener，然后创建不同的 HTTP Path 的路由项。
{{< find_img "img1-1.png" >}}

每条 Rule 支持如下的方式进行组合：
* Host Header
* HTTP Header - 请求的 Header
* HTTP Request Method - 请求的 Method
* Path - 请求的 Path
* Query string
* Source IP - 源 IP 地址

Rule 按照编号从小到大进行匹配，当匹配到了之后就执行路由，不再匹配。Listener 创建时指定的规则为 Default Action，是最后一个 Rule。

## 4 Target Group

每个 Listener(Rule) 都需要绑定对应的 Target Group。Target Group 是一个抽象的概念，用于表明有哪些后端服务。

Target Group 有四种类型：

* Instance - EC2 Instance
  
  选择与 LB 同 Subnet 的 EC2 Instance

* IP 地址 - 指定多个后端服务的 IPv4 地址
  
* Lambda 函数 - 指定触发的 Lambda 函数（仅 ALB 支持）
  
* ALB - 后端由 ALB 处理

指定 Target Group 时，也需要指定请求后端服务的 Port。

{{< admonition note Note>}}
Listener 的 Port 用于判断是否处理请求，Target Group 的 Port 是访问 Target 指定的端口。 
{{< /admonition >}}

### 4.1 Health Check

创建 Target Group 时，需要指定 Health Check 机制。LB 会周期性对所有 Target 进行健康检查。

Health Check 的状态变化是平滑的：
* Healthy -> Unhealthy: 连续 Health Check 失败次数达到阈值 "Healthy threshold"
* Unhealthy -> Healthy: 连续 Health Check 成功次数达到阈值 "Healthy threshold"

当某个 Target 变为 Unhealthy 时，LB 就不会将流量路由到该 Target，直到重新变为 Healthy。
{{< find_img "img1-2.png" >}}

Health Check 支持以下协议：
* TCP - 对于 TCP 端口发起连接请求，连接成功就表明健康检查成功
* HTTP - 发起一个 HTTP Path 的请求，检查其回复的 HTTP Code
* HTTP - 发起一个 HTTPs Path 的请求，检查其回复的 HTTP Code

{{< admonition note Note>}}
对于 UDP NLB，也仅仅只能通过 TCP 来进行健康检查。
{{< /admonition >}}

## 5 Application Load Balancer

ALB 是基于七层的代理，创建时需要指定其所在的 VPC，以及指定至少两个 AZ 的 Subnet。LB 会在每个 Subnet 创建一个 ENI，用于作为 LB 域名解析后的 IP 之一。
{{< find_img "img2.png" >}}

{{< admonition note Note>}}
使用 `nslookup <lb-addr>` 解析 LB 域名时，得到的就是各个 Subnet 的 ENI 的 IP 地址。
{{< /admonition >}}

因为 LB 使用 ENI 来实现，因此也可以为 LB 设置 Security Group，来控制出入 LB 的流量。

### 5.1 LB 的状态

LB 可能处于以下状态之一：
* provisioning - 创建 LB 中
* active - LB 已经就绪，并正常路由流量中
* active_impaired - LB 正在路由流量，但是没有扩展所需的资源
* failed - LB 无法创建

### 5.2 Access Log

ELB 可以提供 Access Log，用于查看发送到 LB 的请求的详细信息。

Access Log 功能默认是关闭的，启用后 AWS 会将访问日志压缩，并存储到用户指定的 S3 bucket 路径。使用访问日志不需要额外的付费，只需要为 S3 的存储付费。

ELB 默认每 5min 保存一次 LB 的 Access Log，存储的文件名格式如下：
```
<bucket path>/AWSLogs/<aws-account-id>/elasticloadbalancing/<region>/<year>/<month>/<day>/<account_id>_elasticloadbalancing_<region>_<load-balancer-id>_<end-time>_<ip-address>_<random-string>.log.gz
```

每个访问请求的信息包括 请求时间、客户端 IP、延迟、服务器回复等等，完整的列表见：[**Access Log Entry**](https://docs.aws.amazon.com/zh_cn/elasticloadbalancing/latest/application/load-balancer-access-logs.html#access-log-entry-format)。

看一个例子，每个信息之间通过空格隔开：
```
http 2018-07-02T22:23:00.186641Z app/my-loadbalancer/50dc6c495c0c9188 
192.168.131.39:2817 10.0.0.1:80 0.000 0.001 0.000 200 200 34 366 
"GET http://www.example.com:80/ HTTP/1.1" "curl/7.46.0" - - 
arn:aws:elasticloadbalancing:us-east-2:123456789012:targetgroup/my-targets/73e2d6bc24d8a067
"Root=1-58337262-36d228ad5d99923122bbe354" "-" "-" 
0 2018-07-02T22:22:48.364000Z "forward" "-" "-" 10.0.0.1:80 200 "-" "-"
```

### 5.3 Deletion Protection

为了防止 LB 意外被删除，可以开启 Deletion Protection 功能。开启后，需要先关闭 Deletion Protection 功能，然后才能删除 LB。

### 5.4 Connection idle timeout

对于 Client 的访问，LB 会维护两个连接：
* Client 与 LB 之间的连接
* LB 与 Target 之间的连接

对于这两个连接，一旦连接空闲（未发送或接收任何数据）时间超过 idle timeout 后，LB 就会关闭连接。默认的 idle timeout 为 60s。

对于 LB 与 Target 之间的连接，为了能够复用后端连接，推荐开启 HTTP keep alive。

### 5.5 Desync mitigation mode

在 LB 这种模式下，HTTP 的访问是由 LB 异步发送给 Target 进行处理的。Desync mitigation mode 就是用于保护 Target 不受到 HTTP 异步的影响。

LB 会使用 [**http_desync_guardian**](https://github.com/aws/http-desync-guardian) 库，将每个请求进行威胁分级，根据 desync mitigation mode 来选择是否转发某个请求。

| Classifications | Monitor mode | Defensive mode（默认） | Strictest mode |
| --------------- | ------------ | ---------------------- | -------------- |
| Compliant       | Allowed      | Allowed                | Allowed        |
| Acceptable      | Allowed      | Allowed                | Blocked        |
| Ambiguous       | Allowed      | Allowed                | Blocked        |
| Severe          | Allowed      | Blocked                | Blocked        |

### 5.6 HEAD 处理

ALB 支持转发请求时添加 `X-Forwarded` 相关的 HTTP HEAD，用于提供给 Target 一些无法看到的信息。

* `X-Forwarded-For` - 包含 Client 的 IP 地址
  
  因为 LB 负责与 Target 进行通信，因此 Target 无法从协议上看到客户端的源 IP 地址。ALB 会在发送给 Target 的请求的 HTTP HEAD 加上 `X-Forwarded-For` 以提供给 Target 客户端的 IP 地址。

  进一步的，可以开启 “客户端端口保留功能”，`X-Forwarded-For` 还会加上 Client 的源端口。

* `X-Forwarded-Proto` - 包含 Client 与 ALB 通信的协议

* `X-Forwarded-Port` - 包含 Client 访问 ALB 的目标端口

当请求包含无效格式（符合正则表达式 [-A-Za-z0-9]+） 的 HTTP HEAD 时，默认 ALB 转发请求时会将其删除，用户可以选择保留无效的 HTTP HEAD。

### 5.7 Sticky Sessions

默认下，ALB 会根据选择的负载均衡算法将请求路由到 Target 上。用户也可以选择使用 Sticky Sessions 功能，使得将 Session 绑定路由到某个 Target 上。也就是说，在 Session 期间，用户的所有请求都会发送到同一个 Target 来处理。

{{< admonition note Note>}}
要使用 Sticky Sessions，客户端必须开启 Cookie。
{{< /admonition >}}

ALB 支持两种类型：

* 基于持续时间的 Cookie
  
  当 ALB 第一次收到 Client 请求时，会根据选定算法路由给 Target，并根据相关信息生成名为 `AWSALB` 的加密的 Cookie。Cookie 有着 7 天的过期时间。

  后续 ALB 收到 Client 请求时，如果 Client 包含 `AWSALB` Cookie，那么会将其解密，根据信息将其路由到同一个 Target。如果该 Target 已经不存在或者 unhealthy，那么会选择一个新的 Target，同时更新 Cookie 的值。

* 基于应用程序的 Cookie

    基于应用程序的 Cookie 是在 基于持续时间 的基础上，让应用程序也知晓 Cookie 的存在。

    当 ALB 第一次收到 Client 请求时，会根据选定算法路由给 Target，而 Target 返回回复需要包含一个 Application Cookie。ALB 收到 Application Cookie 后，会根据信息与该 Cookie 生成 `AWSALB` Cookie。同样，该 Cookie 有着 7 天的过期时间。Client 会收到 Application Cookie 与 `AWSALB` Cookie。

    后续 ALB 收到 Client 请求时，LB 会解密 `AWSALB` Cookie，根据信息将其路由到同一个 Target，并且会传递 Application Cookie。Target 可以根据 Application Cookie 从而知晓这是一个存在 Session。

## 6 Network Load Balancer

NLB 是基于四层的负载均衡，支持 TCP 与 UDP 协议。

对于 TCP 流量，LB 基于协议、源 IP 地址、源端口、目标 IP 地址、目标端口和 TCP 序列号，使用哈希算法选择 Target。每个 TCP 连接在有效期内都会路由到同一个 Target。

对于 UDP 流量，LB 基于协议、源 IP 地址、源端口、目标 IP 地址、目标端口，使用哈希算法选择 Target。如果 UDP 数据包有着相同的源地址和目标地址，那么就会路由到同一个 Target。

创建 NLB 时需要指定其所在的 VPC，以及至少两个 AZ 的 Subnet。LB 会在每个 Subnet 创建一个 ENI，用于作为 LB 域名解析后的 IP 之一。
{{< find_img "img2.png" >}}

{{< admonition note Note>}}
使用 `nslookup <lb-addr>` 解析 LB 域名时，得到的就是各个 Subnet 的 ENI 的 IP 地址。
{{< /admonition >}}

因为 LB 使用 ENI 来实现，因此也可以为 LB 设置 Security Group，来控制出入 LB 的流量。

### 6.1 LB 的状态

LB 可能处于以下状态之一：
* provisioning - 创建 LB 中
* active - LB 已经就绪，并正常路由流量中
* failed - LB 无法创建

### 6.2 Access Log

ELB 可以提供 TLS 请求的 Access Log，用于查看发送到 LB 的请求的详细信息。

{{< admonition note Note>}}
仅仅当 LB 具有 TLS Listener，且它们仅仅包含 TLS 请求的信息时，才会创建访问日志。
{{< /admonition >}}

Access Log 功能默认是关闭的，启用后 AWS 会将访问日志压缩，并存储到用户指定的 S3 bucket 路径。使用访问日志不需要额外的付费，只需要为 S3 的存储付费。

ELB 默认每 5min 保存一次 LB 的 Access Log，存储的文件名格式如下：
```
<bucket path>/AWSLogs/<aws-account-id>/elasticloadbalancing/<region>/<year>/<month>/<day>/<account_id>_elasticloadbalancing_<region>_<load-balancer-id>_<end-time>_<ip-address>_<random-string>.log.gz
```

每个访问请求的信息包括 请求时间、客户端 IP、延迟、服务器回复等等，完整的列表见：[**Access Log Entry**](https://docs.aws.amazon.com/zh_cn/elasticloadbalancing/latest/application/load-balancer-access-logs.html#access-log-entry-format)。

看一个例子，每个信息之间通过空格隔开：
```
tls 2.0 2018-12-20T02:59:40 net/my-network-loadbalancer/c6e77e28c25b2234 g3d4b5e8bb8464cd 
72.21.218.154:51341 172.100.100.185:443 5 2 98 246 - 
arn:aws:acm:us-east-2:671290407336:certificate/2a108f19-aded-46b0-8493-c63eb1ef4a99 - 
ECDHE-RSA-AES128-SHA tlsv12 - 
my-network-loadbalancer-c6e77e28c25b2234.elb.us-east-2.amazonaws.com
- - - 
```

### 6.3 Deletion Protection

为了防止 LB 意外被删除，可以开启 Deletion Protection 功能。开启后，需要先关闭 Deletion Protection 功能，然后才能删除 LB。

### 6.4 跨 AZ 负载均衡

默认情况下，每个 Subnet 的 LB 节点仅仅为当前 AZ 的 Target 路由流量。开启 跨 AZ 负载均衡 功能后，每个 LB 节点会为所有启用的 AZ 的 Target 之间路由流量。

## 7 Gateway Load Balancer

Gateway Load Balancer 作为一个透明网络的网关设备（网络中的所有流量的单一入口与出口点），所有的流量会经过注册的第三方虚拟设备的处理（例如安全设备对流量的检查）。GWLB 基于第三层（网络层）工作，监听所有端口的 IP 数据包，将流量路由到注册的 Target。

对于 TCP/UDP 流量，GWLB 使用 5 元组来路由流量。对于非 TCP/UDP 流量，使用 3 元组来路由流量。

GWLB 使用 GWLB Endpoint 来安全地跨 VPC 边界交换流量。进出 GWLB Endpoint 的流量使用 Route Table 进行配置。

举一个实际的例子：
{{< find_img "img3.png" >}}

从 Internet 到 Application 的流量（蓝色箭头）：
1. 流量进入 Internet Gateway 进入 Public VPC；
2. 根据 Route Table，流量发送给了 GWLB Endpoint；
3. GWLB Endpoint 将流量传输到了 GWLB；
4. 经过 Security Application 检查后，流量会发送回 GWLB Endpoint；
5. 流量被发送到 Application；

从 Application 到 Internet 的流量（橙色箭头）：
1. 根据 Route Table 的默认路由， Application 发出的流量发送到了 GWLB Endpoint；
2. 流量将发送到 GWLB，并经过了 Security Application 的检查；
3. 检查后，流量发送回 GWLB Endpoint；
4. 根据 Route Table，流量发送到 Internet Gateway；
5. 流量发送到 Internet；

使用 GWLB 时，通过对 Route Table 的配置，让 Application 所在的网段的流量都经过另一个 Subnet 的 GWLB Endpoint。例如：
| Destination | Target              |
| ----------- | ------------------- |
| 10.0.0.0/16 | 本地                |
| 10.0.1.0/24 | vpc-endpoint-id     |
| 0.0.0.0/0   | internet-gateway-id |

对于 Application 的路由表，将默认路由指向另一个 Subnet 的 GWLB Endpoint，从而让所有流量经过 GWLB。例如：
| Destination | Target          |
| ----------- | --------------- |
| 10.0.0.0/16 | 本地            |
| 0.0.0.0/0   | vpc-endpoint-id |

### 7.1 LB 的状态

LB 可能处于以下状态之一：
* provisioning - 创建 LB 中
* active - LB 已经就绪，并正常路由流量中
* failed - LB 无法创建

### 7.2 Deletion Protection

为了防止 LB 意外被删除，可以开启 Deletion Protection 功能。开启后，需要先关闭 Deletion Protection 功能，然后才能删除 LB。

### 7.3 跨 AZ 负载均衡

默认情况下，每个 Subnet 的 LB 节点仅仅为当前 AZ 的 Target 路由流量。开启 跨 AZ 负载均衡 功能后，每个 LB 节点会为所有启用的 AZ 的 Target 之间路由流量。

## 8 限制

如果请求与回复都是在同一个 LB 代理的 Node，LB 是不会处理该请求的，这是由于虚拟网卡的 hairpin 模式引起。

## 参考

* [**Application Load Balancer**](https://docs.aws.amazon.com/zh_cn/elasticloadbalancing/latest/application/introduction.html)
* [**Network Load Balancer**](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)
* [**Gateway Load Balancer**](https://docs.aws.amazon.com/zh_cn/elasticloadbalancing/latest/gateway/introduction.html)
