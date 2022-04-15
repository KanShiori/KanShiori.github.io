# Kubernetes 使用 ExternalDNS 同步 DNS


## 1 概述

[**ExternalDNS**](https://github.com/kubernetes-sigs/external-dns) 用于将 Kubernetes 集群中的 Service/Ingress 暴露的服务同步给外部的 DNS 服务商，例如 AWS Route53、GCP CloudDNS 等。

ExternalDNS 本身并不是一个 DNS Nameserver，而是负责将 Kubernetes 集群的 DNS 记录写入到外部的 DNS Nameserver。

## 2 为 AWS Route53 设置 External DNS

### 2.1 启动 EKS 集群

执行 `eksctl create cluster` 命令启动一个 EKS 集群。

```bash
$ eksctl create cluster --name test-cluster --with-oidc --managed --region us-west-2 --nodes 3
```

注意，这里必须使用 `--with-oidc` 参数来创建 OIDC 服务，因为要让 ServiceAccount 绑定 IAM Policy 需要 OIDC 服务提供。

### 2.2 准备 ServiceAccount

因为 ExternalDNS 需要将 DNS 记录同步给 AWS Route53，因此需要为 ExternalDNS 准备一个绑定了 IAM Role 的 ServiceAccount。

ExternalDNS 所需权限的 IAM Policy 如下，可以看到其需要 Route53 的修改与查看权限。
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        // 这里给予了访问所有 hostzone 的权限，也可以限制为你需要的
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

执行 `aws iam create-policy` 命令来创建 IAM Policy，其名为 `external-dns`
```json
$ aws iam create-policy --policy-name external-dns --policy-document file://external-dns-policy.json
{
    "Policy": {
        "PolicyName": "external-dns",
        "PolicyId": "ANPAVTR2JPDXDMH3BN6RZ",
        "Arn": "arn:aws:iam::385595570414:policy/external-dns",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2021-12-22T14:24:39+00:00",
        "UpdateDate": "2021-12-22T14:24:39+00:00"
    }
}
```
记录这里的 IAM Policy 的 ARN，后续创建 ServiceAccount 时需要使用。

通过 `eksctl create iamserviceaccount` 命令创建绑定 IAM Role 的 ServiceAccount：
```
eksctl create iamserviceaccount --cluster=test-cluster --name=external-dns --namespace=external-dns-admin --attach-policy-arn=arn:aws:iam::385595570414:policy/external-dns --approve --override-existing-serviceaccounts
```
* `--cluster` 与 `--region` 指定了 EKS 集群
* `--name` 与 `--namepsace` 指定创建的 ServiceAccount 的 name 与 ns
* `--attach-policy-arn` 指定了要绑定的 IAM Policy
* `--override-existing-serviceaccounts` 表明允许覆盖已经存在的 ServiceAccount
* `--approve` 表明应用修改

{{< admonition note Note>}}
也可以通过 aws cli 来创建 ServiceAccount，具体见官方文档：[**Creating an IAM role and policy for your service account**](https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html)。
{{< /admonition >}}

### 2.3 准备 Route53 Host Zone

创建 ExternalDNS 同步 DNS 的 Route53 Host Zone，当然如果需要使用已有的 Host Zone，那么可以直接跳过。

通过 `aws route53 create-hosted-zone` 创建一个新的 Host Zone：
```json
$ aws route53 create-hosted-zone --name "external-dns-test.my-org.com." --caller-reference "external-dns-test-$(date +%s)"
{
    "Location": "https://route53.amazonaws.com/2013-04-01/hostedzone/Z10422583QOCYXWPPFU3S",
    "HostedZone": {
        "Id": "/hostedzone/Z10422583QOCYXWPPFU3S",
        "Name": "external-dns-test.my-org.com.",
        "CallerReference": "external-dns-test-1640156982",
        "Config": {
            "PrivateZone": false
        },
        "ResourceRecordSetCount": 2
    },
    // ...
}
```

记录这里的 Host Zone 的 ID，在配置 External DNS 时需要使用。

### 2.4 部署 ExternalDNS

创建 ExternalDNS 的定义 yaml，其中包含了 ClusterRole、ClusterRoleBinding 与 Deployment。
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: external-dns-admin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: external-dns-admin
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: k8s.gcr.io/external-dns/external-dns:v0.7.6
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=external-dns-test.my-org.com
        - --provider=aws
        - --policy=upsert-only
        - --aws-zone-type=public
        - --registry=txt
        - --txt-owner-id=my-hostedzone-identifier
```

需要注意的是，因为 ExternalDNS 的 Pod 需要使用前面创建的 ServiceAccount，因此需要部署在相同的 ns，ClusterRoleBinding 也是如此。

看下 ExternalDNS 的启动参数：
* `--source` 指定了从哪些资源查找需要同步的 Endpoint
* `--domain-filter` 限制处理的目标 DNS Zone
* `--provider` 指明了 DNS 提供商
* `--policy` 指明与 DNS Provider 的数据同步方式，默认 sync，可选 upsert-only, create-only

### 2.5 部署 Service

以 LB Service 为例，来验证 ExternalDNS 的工作。

先创建需要使用的 Service，其定义如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    external-dns.alpha.kubernetes.io/hostname: nginx.external-dns-test.my-org.com
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
    targetPort: 80
  selector:
    app: nginx
```

其关键在于，添加了 key 为 `external-dns.alpha.kubernetes.io/hostname` 的 annotation，其 value 为需要同步到 Route 53 的 DNS 域名。
{{< find_img "img1.png" >}}

等待几分钟后，查询 Route 53 的 DNS 记录可以看到其创建了一个新的 DNS 记录：
```json
$ aws route53 list-resource-record-sets --output json --hosted-zone-id /hostedzone/Z10422583QOCYXWPPFU3S
{
    "ResourceRecordSets": [
        // ...
        {
            "Name": "nginx.external-dns-test.my-org.com.",
            "Type": "A",
            "AliasTarget": {
                "HostedZoneId": "Z368ELLRRE2KJ0",
                "DNSName": "a8ffeecc9c0c048e5bdff2d90d66f307-118096740.us-west-1.elb.amazonaws.com.",
                "EvaluateTargetHealth": true
            }
        }
       // ...
}
```

可以看到，"nginx.external-dns-test.my-org.com." 为一个 DNS 别名，其指向的是 Service 的 LB 域名。

### 2.6 验证

我们部署一个 nginx 服务来验证 DNS 记录已经被同步，其定义如下：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
          name: http
```

然后通过访问在 Service annotation 指定的需要同步的 DNS 域名，来测试 DNS 的同步是否成功：
```bash
$ curl nginx.external-dns-test.my-org.com.
```

## 参考

* [**Github：external-dns**](https://github.com/kubernetes-sigs/external-dns)
* [**Doc：Setting up ExternalDNS for Services on AWS**](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md)
* [**Blog：通过 ExternalDNS 集成外部 DNS 服务**](https://blog.gmem.cc/integrate-with-external-dns-provider)
