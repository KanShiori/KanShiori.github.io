# AWS - EKS


## 1 集群管理

EKS 集群包含两个层面：

{{< image src="img2.png" height=350 >}}

* EKS Control Plane
  
  控制面包含 Kubernetes 的核心组件，例如 ETCD 和 APIServer。控制面完全是由 AWS 托管的，也就是说用户不需要维护控制面。

  AWS 会为控制面配置 ELB，从而可以通过 ELB 来访问 APIServer。

* EKS Node
  
  EKS Node 是账户下运行的一组 EC2 Instance，通过 APIServer Endpoint 连接控制面。

### 1.1 创建集群

有三种方式可以创建 EKS 集群：

* `eksctl` - 开源的 EKS 集群管理命令行。
* AWS Console - 通过 Web 控制台一步步创建 EKS 集群。
* AWS CLI - 通过调用 AWS API 创建 EKS 集群。

### 1.2 升级集群

随着 Kubernetes 的版本迭代，AWS 只会维护一定版本的 EKS 集群，并要求用户定期升级。

升级集群时，AWS 会对 Kubernetes Node 进行滚动升级。当升级后的 Node 就绪后，才会继续升级流程。

{{< admonition note Note>}}
AWS 还会定期备份托管的集群，并且必须要支持恢复集群。
{{< /admonition >}}

主要的升级流程见文档：[**Update EKS Kubernetes Version**](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/update-cluster.html#update-kubernetes-version)。

### 1.3 删除集群

在删除 EKS 集群前，需要先手动删除已经创建的 External Service。如果是 AWS CLI 需要自行删除 NodeGroup。

### 1.4 Addon

Addon 是 AWS 为 EKS 提供的一个特殊概念，用于方便的管理 EKS 上部署的某些辅助软件。基于 Addon，可以通过 AWS CLI 中的 addon 相关命令来进行附加组件的管理。

```bash
aws eks describe-addon-versions
```

目前支持使用 Addon 管理的有：

* VPC CNI
* CoreDNS
* kube-proxy
* EBS CSI

## 2 Node

EKS 的 Worker Node 可以分为三种类型：

* Managed Node Group
  
  针对集群创建的 Node Group，对应的 Instance 创建出来后自动加入到 Kubernetes 集群中。

* Self-Managed Node 
  
  用户自行启动的 Instance，启动后再加入到 Kubernetes 集群。

*  AWS Fargate

> 大多数情况下我们都是直接使用 Managed Node Group，后面操作默认表明都是 Managed Node Group 相关操作。

Node Group 表示一组相同机型的 Node，通过修改 Node Group 可以简单的来管理 Node。

一个 Node Group 对应一个 EC2 ASG，用户只需要调整 Node Group，对应的 ASG 会随之更新。

### 2.1 Auto Scale

EKS 支持两款 AutoScaling 产品：

* Kubernetes 社区的 Cluster AutoScaler - 使用 [**AWS Scale Group**](https://docs.aws.amazon.com/zh_cn/autoscaling/ec2/userguide/AutoScalingGroup.html) 来实现 Instance 自动缩扩容

* Karpenter - 使用 [**EC2 fleet**](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/ec2-fleet.html) 实现 Instance 自动缩扩容

#### 2.1.1 Cluster AutoScaler

Cluster AutoScaler 会根据当前集群的负载情况，通过调整 ASG 的 Instance 数量来实现自动缩扩容。

部署步骤参考官方文档：[**Cluster Autoscaler**](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/autoscaling.html)。大致的部署流程如下：

1. 创建 IAM Policy，包含 Cluster AutoScaler 需要的相关操作权限。
   
2. 创建 Role，关联上述创建的 Policy，并在 [**Trust Relationship**](../iam/#41-trust-relationship) 中填入 EKS 中 OIDC Provider ID，表明允许 AssumeRole。
   
3. 部署 Cluster AutoScaler 所需的 RBAC 一套，并且 ServiceAccount 通过 annotation 表明使用上述创建的 Role。
   
   ```bash
   kubectl annotate serviceaccount cluster-autoscaler \
     eks.amazonaws.com/role-arn=arn:aws:iam::$ACCOUNT_ID:role/$AmazonEKSClusterAutoscalerRole
   ```

   > 或者使用 `eksctl create iamserviceaccount` 来创建 Role 与 ServiceAccount。

4. 部署 Cluster AutoScaler。

## 3 Storage

AWS 相关的 EBS 已经集成到了 Kubernetes 源码中，因此支持直接创建对应类型的 StorageClass，当创建 PV 时就会创建对应的 AWS EBS 实例。

目前原生支持的 EBS 类型：io1, gp2, gp3, sc1, st1。

除了 Kubernetes 原生支持外，也可以安装 AWS EBS CSI 来支持 EBS 类型的 StorageClass。Kubernetes 原生支持与 EBS CSI 有着相同的作用，只是一个是跟着 Kubernetes 发型周期，EBS CSI 是有 AWS 自行维护，往往支持最新的 EBS 存储类型。

{{< admonition note Note>}}
Kubernetes 原生与 EBS CSI 创建 StorageClass 时指定的 [**Provisioner**](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner) 不同。 因此创建 PV 时的逻辑完全是两个不同的程序负责的。

不过两者实现的功能都是一样的。可能 EBS CSI 可以支持更新的存储类型。
{{< /admonition >}}

### 3.1 EBS CSI

EBS CSI 支持两种部署方式：

* EKS Addon 进行部署

  将 EBS CSI 以 EKS 的一种功能进行管理，通过 AWS CLI 来部署与升级。

* 自行部署
  
  通过 Helm 或者 YAML 定义来部署 EBS CSI。

部署 Driver 后，在创建 StorageClass 时将 `provisioner` 指定为 `ebs.csi.aws.com`。

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  fsType: ext4
  iops: "4000"
  throughput: "400"
```

### 3.2 EFS

除了 EBS 块存储外，也支持提供 EFS 文件系统给 Pod 使用。EFS 在 Pod 中所见的也是一个目录，区别在于该目录会对应 EFS 中的一个目录。

要使用 EFS，首先需要安装 EFS CSI。通过 Helm 或者 YAML 定义自行安装 EFS CSI。

一个 StorageClass 的示例如下：

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-92107410
  directoryPerms: "700"
  gidRangeStart: "1000" # optional
  gidRangeEnd: "2000" # optional
  basePath: "/dynamic_provisioning" # optional
```

## 4 Network

EKS 的网络结构图大致如下：

{{< image src="img2.png" height=400 >}}

### 4.1 Control Plane 网络

Control Plane 处于一个独立的 VPC。

当创建 EKS 集群时，用户需要选择 EKS Node 所在的 VPC 与 Subnet。集群创建后，AWS 选择在部分 Subnet 中创建访问 Control Plane 使用的 ENI。可以通过 Description `Amazon EKS` 来搜索这些 ENI。

{{< image src="img3.png" >}}

### 4.2 Node 网络

所有的 Node 都处于一个独立的 VPC，即创建 EKS 集群时指定的 VPC。创建 Node Group 或者加入 Node 都处于该 VPC 下的某个 Subnet。

### 4.3 Pod 网络

默认下，在 Node 上运行的每个 Pod 的会被分配一个 Subnet 网段下的 IP，也就是说 Pod 和 Node 都是属于同一个网段的。

每个 Pod 的 IP 会作为 Secondary IP 添加到 Node 对应的 ENI 上。因此，Pod 与 Node 使用相同的 SecurityGroup 来控制网络访问。

{{< image src="img4.png" height=300 >}}

当然，这样弊端也很明显：Pod 地址是有限的。因此，EKS 也支持自定义 Pod 网络，支持将 Pod 使用同 VPC 下的其他 Subnet。详情参考文档：[**CNI 自定义网络**](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/cni-custom-network.html)。

#### 4.3.1 Service

Kubernetes LoadBalancer 与 Ingress 可以对应到 AWS 的 ELB 与 ALB 服务。当创建 Service/Ingress 资源时，都是由 AWS Load Balancer Controller 来创建对应的 ELB/ALB 服务的。

AWS Load Balancer Controller 也可以作为 EKS Addon 进行快速的安装。

## 5 Security

### 5.1 APIServer 访问控制

创建 EKS 集群时，可以选择三种 APIServer 的访问方式：

* Public（默认）
  
  APIServer 会具有一个域名 + 公网地址，可以限制访问 APIServer 的 IP 白名单。

  当 Kubernetes 集群同 VPC 内请求 APIServer 时流量也会离开 VPC（但是不会离开 AWS 网络）。

* Public and Private
  
  APIServer 会具有一个域名 + 公网地址，可以限制访问 APIServer 的 IP 白名单。

  与 Public 方式的区别在于，同 VPC 内请求 APIServer 时流量不会离开 VPC。

* Private
  
  APIServer 具有域名 + 私网地址，必须只有同 VPC 的流量才可以访问 APIServer。

  > 域名可以被公网解析，但是解析后为 VPC 内的私网地址。

当然，无论哪种方式访问 APIServer，请求都会受到 AWS IAM 与 Kubernetes RBAC 的权限控制。

### 5.2 身份认证

[**APIServer 访问控制**](#apiserver-访问控制) 中控制的是如何能够访问 APIServer，身份认证控制的请求发送到 APIServer，经过 APIServer 身份认证。

EKS APIServer 身份认证使用的是 Kubernetes 本身的身份认证方式，而身份是基于 [**IAM Entity**](../iam/) 的。简单点说，Kubernetes 会认证 AWS User 或者 Role 是否有权限访问 APIServer。

在 [**Kubernetes - 认证与鉴权机制**](../../cloud_native/kubernetes/k8s_learning/authentication-and-authorization/#2-身份认证) 中提到，Kubernetes 支持多种身份认证的方式。EKS APIServer 使用的身份认证方式类似为 [**Webhook Token**](../../cloud_native/kubernetes/k8s_learning/authentication-and-authorization/#26-webhook-token)。其 Webhook Service 为 [**AWS STS**](../iam/#7-security-token-service)，有 STS 进行 IAM 的权限控制。

{{< image src="img5.png" height=250 >}}

大致的认证流程如下：
1. kubectl 在本地执行 `aws eks get-token` 从 STS 获取 Token。
2. kubectl 发送请求给 APIServer，HTTP Head 中包含 Token。
3. APIServer 会请求 STS 验证 Token，STS 会基于 IAM 认证用户是否有权限访问 EKS 集群。

### 5.3 鉴权

身份认证后，Kubernetes 会基于 [**鉴权模块**](../../cloud_native/kubernetes/k8s_learning/authentication-and-authorization/#3-鉴权模块)（实际只使用 RBAC）进行权限判断。

因为 EKS 的用户是 IAM Entity，而 Kubernetes 鉴权使用的是集群内部的用户机制。因此，需要构建一个 IAM Entity -> Kubernetes User 的映射关系。

创建 EKS 时使用的 IAM Entity 会自动加入到 Kubernetes 集群的 `system:masters` 组中，有着完全的访问权限。

其他 IAM Entity 需要通过编辑 ConfigMap `aws-auth` 来映射到某个 Kubernetes 集群的用户或者用户组。`aws-auth` 在集群创建时自动被创建。

`aws-auth` 的内容如下：

```yaml
apiVersion: v1
data:
  mapRoles: |
    - rolearn: arn:aws:iam::111122223333:role/eksctl-my-cluster-nodegroup-standard-wo-NodeInstanceRole-1WP3NUE3O6UCF
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::111122223333:user/admin
      username: admin
      groups:
        - system:masters
    - userarn: arn:aws:iam::444455556666:user/ops-user
      username: ops-user
      groups:
        - eks-console-dashboard-full-access-group
    - userarn: arn:aws:iam::444455556666:user/ops-user2
      username: ops-user2
      groups:
        - eks-console-dashboard-restricted-access-group
```

* mapRoles - 将 IAM Role 映射到 Kubernetes 用户或用户组
* mapUsers - 将 IAM User 映射到 Kubernetes 用户或用户组

### 5.4 Pod 访问 AWS

上面说的都是外部访问 Kubernetes 资源，如果 Pod 内部要访问 AWS 资源，一般有两种方式：

* AK/SK - Pod 内程序直接使用 AK/SK 访问 AWS；
* Role - 将 Service Account 与 Role 构建映射关系，然后 Pod 使用 Service Account 来访问 AWS；

因此，我们需要在集群启动时部署 IAM OIDC Provider，它负责将 Service Account 转换为对应的 IAM Role。

大致步骤如下：

1. 为集群创建 IAM OIDC Provider。
2. 创建 IAM Role，关联相应的 IAM Policy。
3. 将 IAM Role 与 Kubernetes ServiceAccount 关联。

#### 5.4.1 IAM OIDC Provider

在 [**Kubernetes - 认证与鉴权机制**](../../cloud_native/kubernetes/k8s_learning/authentication-and-authorization/#24-serviceaccount-token) 中提到过，ServiceAccount 本质上是基于 JWT 方式来进行鉴权的，Pod 使用时挂载到容器中也是 JWT 格式的。

IAM 支持 OIDC 方式的背后，是 OIDC Provider 会请求 AWS STS 获取一个 JWT（AssumeRoleWithWebIdentity 接口），此 JWT 会被认为是某个 Role 的临时凭证，以此来访问 AWS 服务。

显而易见，IAM OIDC Provider 就是会从 STS 获取 Role 对应的 JWT，然后将其作为 ServiceAccount 对应的 Token，提供给 Pod。

{{< image src="img6.png" height=350 >}}

为 EKS 提供 OIDC Provider 的方式见官方文档 [**Create an IAM OIDC provider**](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)。

{{< admonition note Note>}}
除了 IAM OIDC Provider 外，EKS 也支持使用相同的方式使用任意的 OIDC Provider，以支持使用其他 OIDC 进行身份认证。
{{< /admonition >}}

#### 5.4.2 关联 Role 与 ServiceAccount

关联 ServiceAccount 与 Role 的方式很简单，为 ServiceAccount 资源对象添加一个 Annotation，指向对应的 Role ARN。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn=arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
    eks.amazonaws.com/sts-regional-endpoints=true
```

* `eks.amazonaws.com/role-arn` 表明 ServiceAccount 关联的 Role
* `eks.amazonaws.com/sts-regional-endpoints` （可选）表明调用同 Region 的 STS 服务接口，而不是全局端口。这样可以减少延迟，但是降低了高可用性

Pod 使用 ServiceAccount 后，除了 Kubernetes 内部的 ServiceAccount 挂载，APIServer 会自动为 Pod 启动配置中加上 aws-iam-token 的挂载。

```yaml
  volumes:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: your-sa-token-gvztp
      readOnly: true
    - mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
      name: aws-iam-token
      readOnly: true
  serviceAccount: your-sa
  serviceAccountName: your-sa
  volumes:
  - name: aws-iam-token
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          audience: sts.amazonaws.com
          expirationSeconds: 86400
          path: token
  - name: your-sa-token-gvztp
    secret:
      defaultMode: 420
      secretName: your-sa-token-gvztp
```

Pod 启动后，相关的信息会保存在一些环境变量中：

```bash
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
AWS_ROLE_ARN=arn:aws:iam::ACCOUNT_ID:role/IAM_ROLE_NAME
AWS_STS_REGIONAL_ENDPOINTS=regional
```

在代码中使用 AWS SDK 时，会自动读取 Token 来作为访问 AWS 的配置。

## 参考

* 官方文档：[**EKS User Guide**](https://docs.aws.amazon.com/zh_cn/zh_cn/eks/latest/userguide/what-is-eks.html)


