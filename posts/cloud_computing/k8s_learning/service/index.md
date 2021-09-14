# K8s 学习 - Service


## 1 概述
首先我们需要明确 Service 出现的背景：因为 Pod 设计上是无状态的，可以说没有固定的 IP，所以在**其他组件访问某一个网络服务时，需要一个 “固定” 的地址**。

这个固定的地址，就需要由 **`Service`** 来提供。同样，因为访问的地址固定了，Service 也可以提供负载均衡这样的功能。

最简单的例子，有一个 Web 后端服务，有着 3 个 Pod 运行。而前端想访问该后端服务，想要的只是一个固定的域名或者地址，而不关系后端服务的 Pod 会怎样被调度。
{{< admonition note "为什么这样设计">}}
可以感觉，这就是分层。上下层之间只通过固定的地址通信，而不关心层内部是怎样运行的。
{{< /admonition >}}

## 2 Service 基础
### 2.1 定义
Service 基本的定义如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: string
  namespace: string
  labels:
    - name: string
  annotations:
    - name: string
spec:
  selector: []
  type: string
  clusterIP: string
  sessionAffinity: string
  ports:
  - name: string
    protocol: string
     port: int
     targetPort: int
     nodePort: int
  status:
    loadBalancer:
      ingress:
      ip: string
      hostname: string
  topologyKeys:
    - "key"
  externalName: string
```
* `spec.selector` ：用于 Service 选择被代理的 Pod；
* `spec.type` ： Service 类型，见 [**Service 的类型**](#3-service-类型)；
* `spec.clusterIP` ：固定的地址，为空那么随机提供；
* `spec.sessionAffinity` ： 设置负载均衡策略；
* `spec.ports` ：提供需要代理的协议，源端口，目的端口，宿主机端口（NodePort 类型）；
* `spec.status` ：使用 LoadBalancer 类型下，提供相关参数；
* `spec.topologyKeys` ：控制流量转发的拓扑控制，优先将流量转发到相同 key 的 Node 上的 Pod；
* `spec.externalName` ：ExternalName 类型 service 代理的集群外的服务域名；

### 2.2 负载均衡策略
k8s 默认提供两种负载均衡策略：
* **RoundRobin** ：轮询模式，将请求轮询到后端各个 Pod。默认模式
* **SessionAffinity** ：基于客户端 IP 地址进行会话保持的模式。

  即第一次将某个客户端发起请求到后端某个 Pod，之后相同客户端发起请求都会被转发到对应 Pod。

将 `spec.sessionAffinity` 指定为 "ClientIP"，就表明了开启 SessionAffinity 策略。

### 2.3 支持的网络协议
目前 Service 支持如下网络协议：
* TCP: 默认网络协议，可用于所有类型 Service
* UDP: 可用于大多数类型 Service。LoadBalancer 类型取决于云服务是否支持
* HTTP: 取决于云服务是否支持
* PROXY: 取决于云服务是否支持
* SCTP

从 1.17 版本开始，可以为 Service 和 Endpoint 资源对象设置 `spec.ports[].AppProtocol` 字段，用于表示后端服务在某端口停的应用层协议类型。
```yaml
# ...
spec:
  ports:
  - port: 8080
    targetPort: 8080
    AppProtocol: HTTP
```

## 3 服务发现
Kubernetes 提供了两种机制供客户端以固定的范式获取后端 Service 的访问地址：环境变量和 DNS。

### 3.1 环境变量方式
如果 Pod 在 Service 之后创建，那么集群中同 namespace **Service 地址信息会通过 ENV 传递给容器**（不包括 Headless Service）。

相关的环境变量包括：
* \<service>_PORT=\<proto>://\<clusterip>:\<port>
* \<service>\_PORT_\<port>_\<proto>=\<proto>://\<clusterip>:\<port>
* \<service>\_PORT_\<port>_\<proto>_ADDR=\<clusterip>
* \<service>\_PORT_\<port>_\<proto>_PORT=\<port>
* \<service>\_PORT_\<port>_\<proto>_PROTO=\<proto>
* \<service>_SERVICE_HOST=\<clusterip>
* \<service>_SERVICE_PORT=\<port>
* \<service>_SERVICE_PORT_\<NAME>=\<port>

默认相关的环境变量使用的都是 service.spec.port[0] 信息。如果使用的多 port 代理，需要使用 xxx_PORT_xxx 相关环境变量。

环境变量默认还会提供 kubernetes 的 service 地址，用于 Pod 来访问 APIServer。

### 3.2 DNS 方式
所有 Service 都会自动对应一个 DNS 域名，其命令方式为 `<service>.<namespace>.svc.cluster.<cluster_domain>`。由 CoreDNS 作为集群的默认 DNS 服务器，提供域名解析服务。

如果 Service 定义中设置了 `spec.ports[].name`，那么该端口号也会有一个域名：`_<port_name>._<protocol>.<service>.<namespace>.svc.<cluster_domain>。
```yaml
# 提供了 _http._tcp.<name>.<namspace>.svc.local 的 DNS 域名
spec:
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      name: http
```

当执行 nslookup \<service> 时，自动在当前的 namespace 下访问。通过 nslookup \<service>.\<namespace> 可以跨 namespace 进行 DNS 解析。


## 4 Service 类型
Service 核心的功能是：代理。代理涉及到两个点：访问代理的前端，被代理的后端。

针对这个前后端的不同，Service 分为了 4 种类型：
* **`ClusterIP`**（默认）：**代理一组后端 Pod，提供一个固定的内部地址 ClusterIP + Port**，也称为 VIP；
* **`NodePort`** ：NodePort 在 ClusterIP 基础上，会**将 ClusterIP + Port 映射到 Node 上的某个端口**，使得集群外部可以访问内部服务；
* **`LoadBalancer`** ：LoadBalancer 在 NodePort 上，**将一部分 Node 的端口映射到一个 IP 上**，使得外部网络可以访问 Node；
* **`ExternalName`** ：**代理集群外部服务的 DNS 域名**，而不是通过 selector 来选择集群内部后端 Pod；

可以看到，ClusterIP NodePort LoadBalancer 是针对访问代理的前端做了区分，而 ExternalName 是在被代理后端的不同。

### 4.1 ClusterIP
ClusterIP 是最基本的 Service，通过一个 VIP + Port 来在集群内部代理了一组 Pod 的 Port。始终要记得，VIP 是在集群内部才能使用。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-python
spec:
  ports:
  - port: 3000  # 入口端口
    protocol: TCP  # 代理协议
    targetPort: 443  # 目标端口
  selector:
    run: pod-python  # 仅代理匹配的 Pod
  type: ClusterIP
```

上述定义创建了一个随机选择 IP，代理 TCP 3000 -> 443 端口的 Service。
{{< find_img "img1.png" >}}

### 4.2 NodePort
NodePort 是对 ClusterIP 的增强，增加一个宿主机上端口到代理源端口的转发，使得集群外部也可以访问集群内部的服务。

{{< admonition note port-forward>}}
当你执行 `kubectl port-forward svc/xxx` 命令时，其实也是增加了一个宿主端口到 Service 源端口转发，和 NodePort 一样。
{{< /admonition >}}

通过 `service.spec.ports[].nodePort` 指定映射的宿主机端口：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-python
spec:
  ports:
  - port: 3000
    protocol: TCP
    targetPort: 443
    nodePort: 30080
  selector:
    run: pod-python
  type: NodePort
```

上述定义在 ClusterIP 基础上，增加了宿主机 30080 的端口转发。
{{< find_img "img2.png" >}}

### 4.3 LoadBalancer
在云厂商环境下，Node 都是在云的托管集群中的，所以外网访问 k8s 集群内的路径为："外网 -> 云集群 -> k8s 集群"。而 NodePort 仅仅解决了 "云集群 -> k8s 集群" 这个问题。

因此，LoadBalancer Service 在 NodePort 基础上，提供了云厂商需要的负载均衡信息，而云厂商根据该信息设置好 "外网 -> 云集群" 的转发路径。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-python
spec:
  ports:
  - port: 3000
    protocol: TCP
    targetPort: 443
    nodePort: 30080
  selector:
    run: pod-python
  type: LoadBalancer
  externalTrafficPolicy: Local
```
* spec.externalTrafficPolicy 

spec 中定义仅仅是约定的规范，不同厂商所需要的更加细节的 LoadBalancer 的参数，大多数是通过 `service.metadata.annotations` 来提供：
```yaml
    metadata:
      name: my-service
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
        # Specifies whether access logs are enabled for the load balancer
        service.beta.kubernetes.io/aws-load-balancer-access-log-emit-interval: "60"
        # The interval for publishing the access logs. You can specify an interval of either 5 or 60 (minutes).
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "my-bucket"
        # The name of the Amazon S3 bucket where the access logs are stored
        service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: "my-bucket-prefix/prod"
        # The logical hierarchy you created for your Amazon S3 bucket, for example `my-bucket-prefix/prod`
```

对于 LoadBalancer，更多的信息见官方文档：[**LoadBalancer**](https://kubernetes.io/zh/docs/concepts/services-networking/service/#loadbalancer)。

### 4.4 ExternalName
前面三种 Service 代理的后端都是集群内部的 Pod，而 ExternalName 不再是代理 Pod，而是将请求域名重定向另一个域名。

因此，ExternalName Service 不再提供代理的功能，而是提供了**域名重定向**的功能。
```yaml
kind: Service
apiVersion: v1
metadata:
  name: service-python
spec:
  ports:
  - port: 3000
    protocol: TCP
    targetPort: 443
  type: ExternalName
  externalName: remote.server.url.com
```

通过 `serivce.spec.externalName` 指定被代理的域名，而集群内部的 Pod 就可以通过该 Service 访问外部服务了。
{{< find_img "img3.png" >}}

## 5 Endpoint
当创建一个 Service 后，会根据 `service.spec.selector` 自动来匹配作为后端的 Pod。实际上，会对应一个 **`Endpoints`** 对象来代表其匹配到的代理目标，而其每一个被代理的 Pod 称之为 **`Endpoint`**。
```shell
$ kubectl get endpoints -A
NAMESPACE     NAME                      ENDPOINTS                                                     AGE
default       kubernetes                172.16.4.169:6443                                             3d1h
kube-system   kube-dns                  10.36.0.4:53,10.36.0.5:53,10.36.0.4:53 + 3 more...            3d1h
```

当然，你也可以不提供 `service.spec.selector`，也不使用 ExternalName Service，而指定创建一个 Endpoints 对象。这样可以实现，Service 只代理指定的地址：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: plat-dev
spec:
  ports:
    - port: 3306
      protocol: TCP
      targetPort: 3306

---
apiVersion: v1
kind: Endpoints
metadata:
  name: plat-dev
subsets:
  - addresses:
      - ip: "10.5.10.109"
    ports:
      - port: 3306
```

同名的 Endpoint 与 Service 自动被认为是相绑定的。

## 6 Headless Service
Service 会自动发现一组 Pod，并提供代理服务与负载均衡。不过有时候，Pod 中程序并不想使用 Service 的代理功能，而是仅仅想让 Service 作为一个服务发现的作用，例如，peer2peer 程序想要知道有哪些对端的程序。

通过 Service 的定义看，这种情况也就是不需要 `service.spec.clusterIP`，但是需要 `service.spec.selector`。这种特殊的 Service 被称为 **`Headless Service`**。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-balance-svc
  namespace: mysql-space
spec:
  ports:
  - port: 3308
    protocol: TCP
    targetPort: 3306
  clusterIP: None  # clusterIP 指定为 None 表明不需要
  selector:
    name: mysql-balance-pod
  publishNotReadyAddresses: false
```
* `spec.clusterIP`: None 表明其为 Headless Service
* `sepc.publishNotReadyAddresses` ：为 true 时，即使 Pod 还不是 Ready 状态，也会提供 DNS 记录

`service.spec.selector` 让 Service 匹配到后端 Pod，并创建对应的 Endpoints。

当我们直接使用 Service DNS 进行解析时，会得到 DNS 系统返回的全部 Endpoint 地址。
```shell
$ nsloopup mysql-balance-svc.mysql-space.svc.cluster.local

Server: 169.169.0.100
Address: 169.169.0.100#53
Name :mysql-balance-svc.mysql-space.svc.cluster.local
Address: 10.0.95.13
Name :mysql-balance-svc.mysql-space.svc.cluster.local
Address: 10.0.95.12
Name :mysql-balance-svc.mysql-space.svc.cluster.local
```

通过 Headless Service，然后设置 Pod 的 hostname 与 subdomain，就可以实现通过固定 DNS 记录访问某个 Pod。也就是 StatefulSet 使用 Headless Service 的方式。

这里提及一下，StatefulSet Pod 对应的 DNS 记录为：`<podname>.<service>.<namespace>.svc.cluster.local`。
{{< admonition note "DNS 记录不是由 Headless Service 提供的">}}
要注意，StatefulSet Pod 的 DNS 记录不是由 Headless Service 提供的。
{{< /admonition >}}

## 7 EndpointSlice 与 Service Topology
由前面知道，Service 后端是一组 Endpoint，随着集群规模的扩大，Endpoint 数量不断的增长，使得 kube-proxy 需要维护非常多的负载分发规则。

通过 EndpointSlice 与 Service Topology 配合，可以让 kube-proxy 仅仅转发部分的节点，实现对 Endpoint 的分片管理。

### 7.1 EndpointSlice
1.16 版本引入了 **`EndpointSlice`** 机制，包括 EndpointSlice 资源对象和 EndpointSlice Controller。

EndpointSlice 就是代表着一组 Endpoint，kube-proxy 可以使用 EndpointSlice 中的 Endpoint 进行路由转发。
{{< find_img "img5.png" >}}

{{< admonition tip "kube-proxy 开启 EndpointSlice">}}
kube-proxy 默认仍然使用 Endpoint 对象，通过设置其启动参数 --feature-gates="EndpointSlice=true" 来让其使用 EndpointSlice 对象。
{{< /admonition >}}

默认情况下，创建一个 Service 后，就会存在对应的 EndpointSlice 对象，包含匹配到的 Endpoint 对象。
```shell
$ kubectl get endpointslices.discovery.k8s.io
NAME                                                   ADDRESSTYPE   PORTS              ENDPOINTS                                         AGE
my-tidb-cluster-dev-discovery-sm8ns                    IPv4          10262,10261        192.168.54.139                                    2d3h
my-tidb-cluster-dev-pd-dsktv                           IPv4          2379               192.168.135.18,192.168.166.186,192.168.54.140     2d3h
my-tidb-cluster-dev-pd-peer-n6wbg                      IPv4          2380               192.168.135.18,192.168.166.186,192.168.54.140     2d3h
```

### 7.2 Service Topology

Service Topology 可以根据业务需求对 Node 进行分组，设置有意义的指标值来标识 Node 是 “近” 或者 “远”。对于公有云环境来说，通常会进行 Zone 或 Region 的划分。

通过 Service Topology 就实现了对 Endpoint 进行分组，也就是创建多个 EndpointSlice，进行分片管理。

{{< admonition tip "开启 Service Topology">}}
通过设置 kube-apiserver 和 kube-proxy 启动参数 *--feature-gates="ServiceTopology=true,EndpointSlice=true* 开启。
{{< /admonition >}}

通过 Service 定义上的 `spec.topologyKeys` 字段来进行 Service 流量控制。转发流量时，会去按照 topologyKeys 字段顺序匹配 Node 的 label，同样的 key 对应的 value 算作匹配成功，那么才能将流量转发到该 Node 上。
```yaml
spec:
  topologyKeys:
  - "kubernetes.io/hostname"  # 匹配 Node 与要转发的 Node，该 key 的 label 相同才可以转发流量
---
spec:
  topologyKeys:
  - "topology.kubernetes.io/zone"   # 优先转发到同 zone
  - "topology.kubernetes.io/region" # 其次转发到同 region
  - "*" # 最后随意转发
```

## 8 Ingress
Service 提供了基于 4 层的代理，也就是基于 IP + Port 的代理。而 **`Ingress` 出现就是为了支持 7 层的代理**，典型的就是支持 HTTP/HTTPS 协议的代理。
{{< find_img "img4.png" >}}

Ingress 类似于 nginx 的配置，提供应用层的路由，将流量路由给基于传输层的 Service，而由 Service 将流量路由给底层的 Pod。

不过 Ingress 类似于 Service，仅仅是一个规则的定义，其代理的实现依赖于 Ingress Controller。

### 8.1 Ingress Controller
**`Ingress Controller`** 基于定义好的 Ingress 来**实现实际的路由**，可以理解为就是实际上的 nginx。

Ingress Controller 是不包含在 controller manager 默认启动的 controller 的，需要手动进行 controller。

当然，目前有着许多种的 Ingress Controller 的实现，具体见 [**Ingress Controller**](https://kubernetes.io/zh/docs/concepts/services-networking/ingress-controllers/#%E5%85%B6%E4%BB%96%E6%8E%A7%E5%88%B6%E5%99%A8)。

### 8.2 Ingress 定义
Ingress 的配置和配置 nginx 类似，基于 HTTP path 路径进行路由的配置。

看一个示例定义：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-backend
spec:
  ingressClassName: istio
  tls:
    - hosts: []
    secretName: secret
  defaultBackend:
    service:
      name: server-name
      port:
        name: port-name
        number: 80
    resource:
      apiGroup: k8s.example.com
      kind: StorageBucket
      name: static-assets
  rules:
    host: ""
    - http:
        paths:
          - path: /icons
            pathType: ImplementationSpecific
            backend:
              resource:
                apiGroup: k8s.example.com
                kind: StorageBucket
                name: icon-assets
```
* `spec.ingressClassName` - 指定使用的 Ingress Controller 的名字，为保持兼容 annotation `kubernetes.io/ingress.class` 依旧可以使用
* `spec.tls` - 证书对应的 secret 与允许的 hosts
* `spec.defaultBackend` - 用于配置默认的后端，当 rules 中所有都不满足时，就会使用 default 路由
  * `service` - 转发到一个 Kubernetes Service
    * `name` - service 的名字
    * `port` - 转发给 service 的 port，可以是名字或者端口号
  * `resource` - 转发给 Ingress 对象同 Namespace 下的一个 Resource（与 service 选项互斥）
* `spec.rules` - 定义路由策略
  * `host` - 匹配请求的 host，可以是精确匹配或者通配符匹配
  * `http` - http 层的多个匹配路径


## 参考
* [**《Kubernetes in Action》**](https://book.douban.com/subject/30418855/)
* [**《Kubernetes 权威指南》**](https://book.douban.com/subject/35458432/)
* [**Blog: 详解 k8s 4 种类型 Service**](https://segmentfault.com/a/1190000023125587)


