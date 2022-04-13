# Kubernetes 集群中的 DNS


## 1 Pod 的 DNS

### 1.1 Pod 的 DNS 域名

对每个 Pod 来说，CoreDNS 会为其设置一个 `<pod_ip>.<namespace>.pod.<clust-domain>` 格式的 DNS 域名。

```shell
# 访问 POD IP DNS
$ nslookup 192-168-166-168.tidb-cluster-dev.pod.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      192-168-166-168.tidb-cluster-dev.pod.cluster.local
Address 1: 192.168.166.168 my-tidb-cluster-dev-tidb-0.my-tidb-cluster-dev-tidb-peer.tidb-cluster-dev.svc.cluster.local
```

对于 Deployment 或 DaemonSet 类型创建的 Pod，CoreDNS 会为其管理的每个 Pod 设置一个 `<pod_ip>.<depolyment/daemonset_name>.svc.<cluster_domain>` 格式的 DNS 域名。

### 1.2 自定义 hostname 和 subdomain

默认情况下，Pod 内容器的 hostname 被设置为 Pod 的名称。因此，使用副本控制器时 Pod 名称会包含随机串，因此 hostname 无法固定。

可以通过 `spec.hostname` 字段定义容器环境的 hostname，通过 `spec.subdomain` 字段定义容器环境的子域名。

```yaml
spec:
  hostname: webapp-1
  subdomain: mysubdomain
```

Pod 创建成功后，Kubernetes 为其设置 DNS 域名为 `<hostname>.<subdomain>.<namespace>.svc.<cluster_domain>`。这时通过部署一个 Headless Service 就可以在 DNS 服务器中自动创建对应 DNS 记录。这样就可以通过该 DNS 域名来访问 Pod。

实际上，StatefulSet 就是通过这种方式来使用 Headless Service 的。查看一个 StatefulSet 管理的 Pod：

```yaml
$ kubectl get po my-tidb-cluster-dev-pd-0 -o yaml
# ...
spec:
  hostname: my-tidb-cluster-dev-pd-0 # 固定 pod 的 hostname
  subdomain: my-tidb-cluster-dev-pd-peer  # 在 StatefulSet 中定义的 Headless Service name
```

### 1.3 自定义 DNS 配置

通过 Pod 定义中的 `spec.dnsPolicy` 可以定义使用的 DNS 策略。

目前包含以下的 DNS 策略：
* Default ：从 Node 继承 DNS
* ClusterFirst ：与配置的集群 domain 后缀不匹配的 DNS 查询，都会转发到从 Node 继承的上游服务器
* ClusterFirstWithHostNet ：如果 Pod 使用 hostNetwork 运行，DNS 应该使用 host
* None ：忽略 Kubernetes DNS 设置，由用户通过 `spec.dnsConfig` 字段进行配置

通过 `spec.dnsConfig` 字段进行更加细节的 DNS 相关配置。

```yaml
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 8.8.8.8
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

Pod 创建后，容器内的 */etc/resolv.conf* 文件会被设置：

```shell
nameserver 8.8.8.8
search ns1.svc.cluster-domain.example my.dns.search.suffix
option natods:2 eth0
```

## 2 Service 中的 DNS

Service DNS 相关已经在 [**Service**](../service-and-endpoint/) 一文中说明，这里仅仅简单提及一下。

每个 Service 会自动对应一个 DNS 域名，命名方式为 `<service>.<namespace>.svc.<cluster_domain>`，cluster_domain 默认为 cluster.local。

通过改变 namespace 可以访问不同 namespace 下的 Service

## 3 Node 本地 DNS 缓存

集群中的 DNS 服务都是通过 "kube-dns" 的 Service 提供的，所有容器都可以通过其 ClusterIP 地址去进行 DNS 域名解析。

为了缓解 DNS 服务的压力，Kubernetes 引入了 Node 本地 DNS 缓存，使得 DNS 域名解析可以在 Node 本地缓存，不用每次跨主机去 CoreDNS 进行解析。

使用 Node 本地 DNS 缓存后，Pod 进行 DNS 解析的流程如下：
{{< find_img "img1.png" >}}

Node 本地 DNS 缓存的功能是通过部署一个 DaemonSet 实现的，其 Pod 运行的 *k8s.gcr.io/k8s-dns-node-cache* 镜像实现了 DNS 缓存功能。

具体部署方式见文档：[**在 Kubernetes 集群中使用 NodeLocal DNSCache**](https://kubernetes.io/zh/docs/tasks/administer-cluster/nodelocaldns/)

## 4 CoreDNS

从 1.11 版本开始，Kubernetes 集群的 DNS 服务由 CoreDNS 提供，其用 Go 实现的一个高性能、插件式、易扩展的 DNS 服务器。

### 4.1 配置 CoreDNS

CoreDNS 的主要功能是通过插件系统实现的，CoreDNS 实现了一种链式插件结构，将 DNS 逻辑抽象为一个个插件，能够组合灵活配置。

常见的插件如下：

* loadbalance ：提供基于 DNS 负载均衡功能
* loop ：检查 DNS 解析出现的循环问题
* cache ：提供前端缓存功能
* health ：对 Endpoint 进行健康检查
* kubernetes ：从 Kubernetes 中读取 zone 数据
* etcd ：从 etcd 中读取 zone 记录，可用于自定义域名记录
* file ：从 RFC11035 格式文件读取 zone 数据
* hosts ：使用 /etc/hosts 文件或者其他文件读取 zone 数据，可用于自定义域名记录
* auto ：从磁盘中自动加载区域文件
* reload ：定时重新加载 Corefile 配置文件内容
* forward ：转发域名查询到上游 DNS 服务器上
* prometheus ：为 Prometheus 系统提供采集性能指标数据 URL
* pprof ：在 URL 路径 /debug/pprof 提供性能数据
* log ：对 DNS 查询进行日志记录
* errors ：对错误信息进行日志记录

默认下，CoreDNS 的 Pod 会使用 *coredns* ConfigMap 提供对 CoreDNS 的配置。因此，你可以通过配置此 ConfigMap 进行 CoreDNS 的配置。

```shell
$ k get configmaps  coredns -n kube-system -o yaml
# ...
kind: ConfigMap
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

## 参考

* [**《Kubernetes 权威指南》**](https://book.douban.com/subject/35458432/)
* [**Core DNS**](https://coredns.io/)
* [**DNS 域名解析系统**](../../../../../net/dns/)
