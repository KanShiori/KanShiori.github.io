# Service 学习


## 1 概述
首先我们需要明确 Service 出现的背景：因为 Pod 设计上是无状态的，可以说没有固定的 IP，所以在**其他组件访问某一个网络服务时，需要一个 “固定” 的地址**。

这个固定的地址，就需要由 **`Service`** 来提供。同样，因为访问的地址固定了，Service 也可以提供负载均衡这样的功能。

最简单的例子，有一个 Web 后端服务，有着 3 个 Pod 运行。而前端想访问该后端服务，想要的只是一个固定的域名或者地址，而不关系后端服务的 Pod 会怎样被调度。
> 可以感觉，这就是分层。上下层之间只通过固定的地址通信，而不关心层内部是怎样运行的。

## 2 Service 基础
### 2.1 定义
Service 完整的定义如下：
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
* `selector` ：用于 Service 选择被代理的 Pod；
* `type` ： Service 类型，见后；
* `clusterIP` ：固定的地址，为空那么随机提供；
* `sessionAffinity` ： 设置负载均衡策略；
* `ports` ：提供需要代理的协议，源端口，目的端口，宿主机端口（NodePort 类型）；
* `status` ：使用 LoadBalancer 类型下，提供相关参数；
* `topologyKeys` ：控制流量转发的拓扑控制，优先将流量转发到相同 key 的 Node 上的 Pod；
* `externalName` ：ExternalName 类型 service 代理的集群外的服务域名；

### 2.2 Service 的 DNS
所有 Service 都会自动对应一个 DNS 域名，其命令方式为 `<service>.<namespace>.svc.cluster.local`。

当访问 nslookup \<service> 时，自动在当前的 namespace 下访问。通过 nslookup \<service>.\<namespace> 也可以跨 namespace 进行 DNS 解析。

### 2.3 Service 相关的环境变量
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

### 2.4 负载均衡策略
k8s 默认提供两种负载均衡策略：
* RoundRobin：轮询模式，将请求轮询到后端各个 Pod。默认模式
* SessionAffinity：基于客户端 IP 地址进行会话保持的模式。<br>
  即第 1 此将某个客户端发起请求到后端某个 Pod，之后相同客户端发起请求都会被转发到对应 Pod。

通过 service.spec.sessionAffinity 指定为 "ClientIP"，就表明了开启 SessionAffinity 策略。

## 3 Service 类型
Service 核心的功能是：代理。因此设计到两个点：访问代理的前端，被代理的后端。

针对这个前后端的不同，Service 分为了 4 种类型：
* **`ClusterIP`**（默认）：**代理一组后端 Pod，提供一个固定的内部地址 ClusterIP + Port**，也称为 VIP；
* **`NodePort`** ：NodePort 在 ClusterIP 基础上，会**将 ClusterIP + Port 映射到 Node 上的某个端口**，使得集群外部可以访问内部服务；
* **`LoadBalancer`** ：LoadBalancer 在 NodePort 上，**将一部分 Node 的端口映射到一个 IP 上**，使得外部网络可以访问 Node；
* **`ExternalName`** ：**代理集群外部服务的 DNS 域名**，而不是通过 selector 来选择集群内部后端 Pod；

可以看到，ClusterIP NodePort LoadBalancer 是针对访问代理的前端做了区分，而 ExternalName 是在被代理后端的不同。

### 3.1 ClusterIP
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

### 3.2 NodePort
NodePort 是对 ClusterIP 的增强，增加一个宿主机上端口到代理源端口的转发，使得集群外部也可以访问集群内部的服务。

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

上述定义在 ClusterIP 基础上，定义了宿主机 30080 的端口转发。
{{< find_img "img2.png" >}}

### 3.3 LoadBalancer
在云厂商环境下，Node 都是在云的托管集群中的，所以外网访问 k8s 集群内的路径为：外网 -> 云集群 -> k8s 集群。而 NodePort 仅仅解决了 云集群 -> k8s 集群 这个问题。

因此，LoadBalancer service 在 NodePort 基础上，提供了云厂商需要的负载均衡信息，而云厂商根据该信息设置好 外网 -> 云集群的转发路径。
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
```

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

### 3.4 ExternalName
前面三种 Service 代理的后端都是集群内部的 Pod，而 ExternalName 不再是代理 Pod，而是将请求域名重定向另一个域名。
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

## 4 Endpoint
当创建一个 Service 时，都自动创建一个 **`Endpoints`** 对象来代表其匹配到的代理目标，而其每一个被代理的目标称之为 **`Endpoint`**。
```bash
kubectl get endpoints -A
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

## 5 Headless Service
Service 会自动发现一组 Pod，并提供代理服务与负载均衡。不过有时候，Pod 中程序并不想使用 Service 的代理功能，而是仅仅想让 Service 作为一个服务发现的作用，例如，peer2peer 程序想要知道有哪些对端的程序。

通过 Service 的定义看，这种情况也就是不需要 `service.spec.clusterIP`，但是需要 `service.spec.selector`。因此 Service 提供了 **`Headless Service`**。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-balance-svc
  namespace: mysql-space
  labels:
    name: mysql-balance-svc
spec:
  #type: NodePort
  ports:
  - port: 3308
    protocol: TCP
    targetPort: 3306
  clusterIP: None
  selector:
    name: mysql-balance-pod
```

上述定义将 `service.spec.clusterIP` 定义为了 "None"，表明创建的是一个 Headless Service。但是 `service.spec.selector` 还是能够让 Service 选择到 Pod，并创建对应的 Endpoints。也就是说，我们可以通过 Endpoints 来知道哪些对应服务的 Pod 正在运行。

最最重要的，每个 Endpoint 对应的 Pod 是存在一个对应的 DNS 记录：`<podname>.<service>.<namespace>.svc.cluster.local`。

这使得 Headless Service 在 StatefulSet 中有很好的应用，因为 StatefulSet 中每个 Pod 的命名是固定的，所以也就是其域名 \<podname>.\<service>.\<namespace>.svc.cluster.local 也固定了，那么 Pod 之间通过域名访问就不需要关心 Pod IP 的变化了。

## 6 Ingress
Service 提供了基于 4 层的代理，也就是基于 IP + Port 的代理。而 **`Ingress` 出现就是为了支持 7 层的代理**，典型的就是支持 HTTP/HTTPS 协议的代理。

{{< find_img "img4.png" >}}

Ingress 类似于 nginx 的配置，提供应用层的路由，将流量路由给基于传输层的 Service，而由 Service 将流量路由给底层的 Pod。

不过 Ingress 类似于 Service，仅仅是一个规则的定义，其代理的实现依赖于 Ingress Controller。

### 6.1 Ingress Controller
**`Ingress Controller`** 基于定义好的 Ingress 来**实现实际的路由**，可以理解为就是实际上的 nginx。

Ingress Controller 是不包含在 controller manager 默认启动的 controller 的，需要手动进行 controller。

当然，目前有着许多种的 Ingress Controller 的实现，具体见 [**Ingress Controller**](https://kubernetes.io/zh/docs/concepts/services-networking/ingress-controllers/#%E5%85%B6%E4%BB%96%E6%8E%A7%E5%88%B6%E5%99%A8)。

### 6.2 Ingress 定义
Ingress 的配置和配置 nginx 类似，基于 HTTP path 路径进行路由的配置。

看一个示例定义：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-backend
spec:
  defaultBackend:
    resource:
      apiGroup: k8s.example.com
      kind: StorageBucket
      name: static-assets
  rules:
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
* `defaultBackend` ： 用于配置默认的后端，当 rules 中所有都不满足时，就会使用 default 路由；
* `rules` ：定义路由策略


## 参考
* [**《Kubernetes in Action》**](https://book.douban.com/subject/30418855/)
* [**Blog: 详解 k8s 4 种类型 Service**](https://segmentfault.com/a/1190000023125587)


