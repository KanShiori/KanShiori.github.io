# AWS - Private Link


## 1 概述

在 [**AWS - VPC 与 Service 通信**](../vpc-and-aws/) 中，描述了 VPC 与 Service 通信的几种方式：

* VPC Peering - 连接两个不同网段的 VPC

* Transit Gateway - 连接多个不同网段的 VPC

* Private Link - 通过 Endpoint 来作为 Service 的代理

每种方式的流量都是完全在 AWS 网络中的，不会经过公网。其中，VPC Peering 与 Transit Gateway 是基础网络架构上的构建，不适用于 C/S 的访问模式。

如下图，使用 Private Link，Client 通过访问 Endpoint 地址来访问后端的 Service，提供了一种相互隔离的 C/S 架构，更适用于 Saas 服务的场景。

{{< image src="img1.png" height=350 >}}

根据后端 Service 的不同，Private Link 分为：

* Endpoint - 用于访问 AWS Service，包括 Gateway Endpoint、Interface Endpoint

* Endpoint Service - 访问 Saas 厂商的 Service

* Gateway Load Balancers - 访问 [**Gateway Load Balancer**](../elb/#7-gateway-load-balancer)

## 2 Endpoint

### 2.1 Interface Endpoint

目前大多数的 AWS Service 都可以使用 Interface Endpoint 访问。

{{< image src="img2-1.png" height=300 >}}

创建 Interface Endpoint 后，AWS 会在指定的 Subnet 中创建一个 ENI，通过指定的 DNS 或 IP 就可以访问 Service。

以访问 S3 为例，使用 Interface Endpoint 大致流程如下：

1. 创建访问 Endpoint 使用的 Security Group

2. 创建 Interface Endpoint
   
   创建时，需要指定访问的 Service，以及 VPC 与 Subnet。

   ```bash
   aws ec2 create-vpc-endpoint \
       --vpc-id vpc-1a2b3c4d \
       --vpc-endpoint-type Interface \
       --service-name com.amazonaws.us-east-1.s3 \
       --subnet-ids subnet-7b16de0c \
       --security-group-id sg-1a2b3c4d \
       --tag-specifications ResourceType=vpc-endpoint,Tags=[{Key=service,Value=S3}]
   ```

   {{< admonition note Note>}}
   创建成功后，在各个 Subnet 会自动创建出访问 Service 使用的 ENI，用于提供网络的连通。
   {{< /admonition >}}

3. 查看 Endpoint 信息，知晓对应的 DNS name

   ```json
   // aws ec2 describe-vpc-endpoints --vpc-endpoint-id vpce-099deb00b40f00e22 --query VpcEndpoints[*].DnsEntries
   [
       [
           {
               "DnsName": "*.vpce-0e5d025c8b9244b00-frjqbhg9.s3.us-west-2.vpce.amazonaws.com",
               "HostedZoneId": "Z1YSA3EXCYUU9Z"
           },
           {
               "DnsName": "*.vpce-0e5d025c8b9244b00-frjqbhg9-us-west-2b.s3.us-west-2.vpce.amazonaws.com",
               "HostedZoneId": "Z1YSA3EXCYUU9Z"
           },
           {
               "DnsName": "*.vpce-0e5d025c8b9244b00-frjqbhg9-us-west-2a.s3.us-west-2.vpce.amazonaws.com",
               "HostedZoneId": "Z1YSA3EXCYUU9Z"
           }
       ]
   ]
   ```
   
   * Regional DNS name - 格式为：`${endpoint_id}-${service_id}.${region}.vpce.amazonaws.com`
   * Zonal DNS name - 格式为：`${endpoint_id}-${service_id}-${az}.${region}.vpce.amazonaws.com`

4. 通过特定 Domain 访问服务
   
   ```bash
   # S3 Domain: `bucket.${dns_name}`
   aws s3 ls s3://${bucket} --endpoint-url https://bucket.vpce-0e5d025c8b9244b00-frjqbhg9-us-west-2a.s3.us-west-2.vpce.amazonaws.com
   ```

   如果想要不通过设置 `--endpoint-url` 参数就通过 Endpoint 访问 Service，可以通过 Route53 将对应 Region 的 S3 Domain 赋予 Alias 以指向 Endpoint 地址。

来看一下 DNS 背后的细节：请求 AZ 级别的 Domain，返回的是对应 Subnet 下对应 ENI IP。请求 Region 级别的 Domain，返回的所有 ENI IP。

{{< image src="img3.png" height=100 >}}

### 2.2 Gateway Endpoint

Gateway Endpoint 可用于访问 S3 与 DynamoDB 两个服务，并且没有使用 Private Link。

{{< image src="img4.png" height=100 >}}

在使用方式上，Gateway Endpoint 与 Interface Endpoint 也不同：基于 Route Table 来将流量发送到 Gateway Endpoint，从而访问服务。

{{< admonition note Note>}}
因为基于 Route Table 实现，使得 Gateway Endpoint 只能在同 VPC 中访问（除非联通 VPC），带来了架构的负载性。
{{< /admonition >}}

以访问 S3 为例，使用 Gateway Endpoint 大致流程如下：

1. 创建 Gateway Endpoint

   创建时，需要指定位于的 VPC 和自动配置的 Route Table。

   ```bash
   aws ec2 create-vpc-endpoint \
    --vpc-id vpc-1a2b3c4d \
    --service-name com.amazonaws.us-east-1.s3 \
    --route-table-ids rtb-11aa22bb
   ```

2. 创建完成后，Route Table 会自动添加相应的 Route，指向 Endpoint ID
   
   ```json
   // aws ec2 describe-route-tables --route-table-ids=rtb-11aa22bb | jq ".RouteTables[0].Routes"
   [
     {
       "DestinationCidrBlock": "10.0.0.0/16",
       "GatewayId": "local",
       "Origin": "CreateRouteTable",
       "State": "active"
     },
     {
       "DestinationCidrBlock": "0.0.0.0/0",
       "GatewayId": "igw-062a0bf05cf1a616a",
       "Origin": "CreateRoute",
       "State": "active"
     },
     {
       "DestinationPrefixListId": "pl-68a54001",
       "GatewayId": "vpce-049adc53b12f5de85",
       "Origin": "CreateRoute",
       "State": "active"
     }
   ]
   ```

   该路由项为 Prefix List，包含了 S3 Endpoint URL 的地址段，也就是访问 S3 地址的数据包都会被路由到 Gateway Endpoint：

   ```json
   // aws ec2 describe-prefix-lists --prefix-list-ids=pl-68a54001
   {
     "PrefixLists": [
       {
         "Cidrs": [
           "3.5.76.0/22",
           "3.5.80.0/21",
           "18.34.48.0/20",
           "52.92.128.0/17",
           "52.218.128.0/17"
         ],
         "PrefixListId": "pl-68a54001",
         "PrefixListName": "com.amazonaws.us-west-2.s3"
       }
     ]
   }
   ```

## 3 Endpoint Service

[**Endpoint**](#2-endpoint) 用于访问 AWS Service，如果 AWS 客户像将自己的 Service 通过 Private Link 暴露出去，可以使用 Endpoint Service。

{{< image src="img5.png" height=300 >}}

### 3.1 Service Provider

Service Provider 作为服务的提供商，需要创建一个 Endpoint Service，其使用 NLB/GLB 作为服务的前端，客户流量基于 NLB/GLB 转发到自己的应用上。创建

创建 Endpoint Service 大致流程如下：

1. 创建 Endpoint Service
   
   创建时，需要指定 Service 入口的 NLB。
   
   ```bash
   aws ec2 create-vpc-endpoint-service-configuration \
    --network-load-balancer-arns arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/net/nlb-vpce/e94221227f1ba532 \
    --acceptance-required \
    --privateDnsName exampleservice.com
   ```

   通过 `--acceptance-required` 参数，指定客户连接 Service 需要 Provider 接受。

   通过 `--privateDnsName` 提供一个私有的 DNS name。这可以支持复用已有的 DNS name，使得客户无感知切换到 Endpoint Service。

2. 授予客户访问 Endpoint Service 的权限。
   
   添加需要访问 Service 的客户（AWS Account、IAM User 或 IAM Roles）ARN，使得有权限访问 Endpoint Service。

   ```bash
   aws ec2 modify-vpc-endpoint-service-permissions \
    --service-id vpce-svc-03d5ebb7d9579a2b3 \
    --add-allowed-principals '["arn:aws:iam::123456789012:user/admin"]'
   ```

   也可以通过 `*` 来表示允许所有客户访问。

创建 Endpoint Service 时，AWS 会自动生成如下格式的 DNS name：

* `<endpoint_service_id>.<region>.vpce.amazonaws.com` - Endpoint Service 对应的 DNS

当 Consumer 创建 Interface Endpoint 连接 Service 后，AWS 会创建如下 DNS name 用于访问：

* `<endpoint_id>.<endpoint_service_id>.<region>.vpce.amazonaws.com` - Region 级别
* `<endpoint_id>-<zone>.<endpoint_service_id>.<region>.vpce.amazonaws.com` - Zone 级别

### 3.2 Service Consumer

Service Consumer 作为服务的使用者，在自己的账户中需要创建 [**Interface Endpoint**](#21-interface-endpoint)，连接到 Provider 提供的 Endpoint Service。

大致流程与 [**Interface Endpoint**](#21-interface-endpoint) 一样，区别仅仅在于指定的 Service ID 为 Provider 提供的。

如果 Provider 使用私有的 DNS name，那么 Consumer 也需要开启私有 DNS 选项 `--private-dns-enabled`。

## 参考

* 官方文档：[**AWS PrivateLink**](https://docs.aws.amazon.com/zh_cn/vpc/latest/privatelink/endpoint-services-overview.html)

* Blog：[**使用 VPC Endpoint 从 VPC 或 IDC 内访问 S3**](https://aws.amazon.com/cn/blogs/china/use-vpc-endpoint-access-from-vpc-or-idc-s3/)





