# K8s 学习 - API 概念



## 1 概述
### 1.1 组织对象的方式
Kubernetes 中，组织对象的方式，就是按照 Group、Version、Resource 三个层级。
* **`Group`** 用以来对 API 进行分组（分类）；
* **`Version`** 用以对相同的 API 进行版本控制；
* **`Resource`** 代表着一个具体的资源对象的 API；

{{< find_img "img1.png" >}}
	
{{< admonition note "etcd 中如何组织对象">}}
这不是在 etcd 中组织资源对象的方式，etcd 中还是按照资源类型，以及命名来进行分类存储
```bash
$ alias edctl="etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key

$ edctl get /registry  --prefix  --keys-only
/registry/pingcap.com/tidbinitializers/mycluster/mycluster-init
/registry/pingcap.com/tidbmonitors/mycluster/mycluster
/registry/pods/kube-system/coredns-7c7788d75c-cggn5
…
```
{{< /admonition >}}

这三个层次，也会体现在资源定义的 yaml 文件中：
```yaml
apiVersion: batch/v2alpha1  # Group/Version
kind: CronJob  # Resource
#  …
```

而 Kubernetes 就是通过比较 Group Version Resource，再加上资源对象的 name 来寻找一个资源。

### 1.2 REST 风格
Kubernetes 系统大部分情况下，API 定义和标准都符合 HTTP REST 标准，即通过 POST、PUT、GET、DELETE 等 HTTP method 来进行对象的增删改查操作。

目前，Kubernetes 使用 OpenAPI 文档格式生成接口文档。你可以通过 `https://<master_ip>:<master_port>/openapi/v2` 来访问 API 文档。
```
# 获取 TOEKN
$ TOKEN=$(kubectl describe secrets $(kubectl get secrets -n kube-system |grep admin |cut -f1 -d ' ') -n kube-system |grep -E '^token' |cut -f2 -d':'|tr -d '\t'|tr -d ' ')

# 获取 APIServer 地址
$ APISERVER=$(kubectl config view |grep server|cut -f 2- -d ":" | tr -d " ")

$ curl -H "Authorization: Bearer $TOKEN" $APISERVER/openapi/v2 -k  | jq | more
{
  "swagger": "2.0",
  "info": {
    "title": "Kubernetes",
    "version": "v1.21.1"
  },
  "paths": {
# ...
```


## 2 版本控制
Kubernetes 对于版本的控制体现在 URL 中，每个 API 对应的 URL 基于 `/apis/<group>/<version>/namespaces/<ns>/<resource>` 的格式构建，所以不同的版本有着不同的 URL。

对于一个特定的版本，例如 v1，包含三种级别的版本：
* **Alpha** ：预览版本

  version 以 **`<v>alpha<nr>`** 格式，例如 v1alpha1。

  表明软件是不稳定，软件可能会有 Bug，某些新增特性支持可能随时被删除，某些新增 API 可能会出现不兼容性更改。

  因此仅仅使用于测试集群，不适合生产环境。
* **Beta** ：测试版本

  version 以 **`<v>beta<nr>`** 格式，例如 v2beta3。

  表明软件经过测试，并且特性都是安全的。如果特性与 API 发生兼容性更改，那么会提供迁移的说明。
* **GA** ：稳定版本

  verison 以 **`<v>`** 格式，例如 v1。


## 3 API Groups
为了更容易扩展 API，Kubernetes 将 API 分组为多个逻辑集合，称为 `API Groups`。
当前支持两类的 API Groups：
* **Core Groups**

  又称为 **Legacy Groups**，大部分核心资源对象都在该组里，例如 Container、Pod、Service 等。

  其 URL 路径前缀为 **`/api/v1`**。其 Group 为空，因此使用 `spec.apiVersion : v1`。

* **其他 API Groups**
  
  URL 路径前缀为 **`/apis/<Group>/<Version>`**，使用 `spec.apiVersion: <Group>/<Version>`。

  常见的分组包括：
    * `apps/v1` ：主要包含与用户发布、部署有关资源，包括 Deployments，RollingUpdates，ReplicaSet。
    * `extensions/<Version>` ：扩展 API 组，包括 DaemonSet、ReplicaSet，Ingresses。
    * `batch/<Version>` ：批处理和作业任务的对象，包括 Job。
    * `autoscaling<Version>` ：HPA 相关资源对象。
    * `certificate.k8s.io/<Version>` ：集群证书操作相关资源对象。
    * `rbac.authorization.k8s.io/v1` ：RBAC 权限相关资源对象。
    * `policy/<Version>` ：Pod 权限性相关的资源。
    * 其他 CRD 定义的分组

Kubernetes 内置的所有 API Group 可以见 [**API 参考文档**](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#-strong-api-groups-strong-)。

### 3.1 启用或禁用 API Group
所有 API Group 默认情况都是可用的。可以通过 API Server 启动配置开启用或禁用某个 API 组。

例如 `--runtime-config=batch/v1=false` 表明禁用 batch/v1 下的所有 API。


## 4 对象
Kubernetes API 所有资源类型都是以 **对象** 方式看待，并且以 REST 方式提供 HTTP API 接口。

每个对象的 API 以以下格式表示：
* non-namespace 资源：
  * `GET /apis/<Group>/<Version>/<Resource>` ：返回资源集合
  * `GET /apis/<Group>/<Version>/<Resource>/<Name` ：操作指定 name 的资源对象
* namespace 资源：
  * `GET /apis/<Group>/<Version>/<Resource>` ：返回所有 namespace 下资源对象
  * `GET /apis/<Group>/<Version>/namespaces/<Namespace>/<Resource>` ：返回指定 namespace 下的所有资源对象
  * `GET /apis/<Group>/<Version>/namespaces/<Namespace>/<Resource>/<Name>` ：返回 namespace 下指定 name 的资源对象

### 4.1 watch 对象
基于 etcd 的机制，每个 Kubernetes 对象都有一个 `resourceVersion` 字段，**代表着资源在 etcd 中存储的版本**。

因此，在 client 的 watch 连接断开，重新连接后，可以通过指定 `resourceVersion` 来得到断连期间其对象所有的更改。
```http
GET /api/v1/namespaces/test/pods?watch=1&resourceVersion=10245
---
200 OK
Transfer-Encoding: chunked
Content-Type: application/json

{
  "type": "ADDED",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "10596", ...}, ...}
}
{
  "type": "MODIFIED",
  "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "11020", ...}, ...}
}
...
```

当然，etcd 只会保存一定时间内的资源变更历史（默认 5min）。如果请求的历史版本不存在，那么会返回 HTTP Code `410 Gone`。

所以 client 必须能够处理 410 错误，清理本地对象缓存，调用 list 操作，重新建立 watch。
{{< admonition note Reflector>}}
官方提供了封装好这些功能的代码组件，例如 Go 中 **Reflector** 组件已经实现了这些逻辑
{{< /admonition >}}

为了解决历史窗口太短的问题，为 event 引入了 `bookmark event` 的概念。

### 4.2 查询对象
一般情况下，查询对象只需要使用 HTTP GET URL 就可以对象集合或者指定对象。

不过有时候，当你查询大量的对象时，可能返回的数据很大，使得浪费网络资源。从 1.9 开始，Kubernetes 支持**分段查询对象**。

在进行查询请求时，可以指定 `limit` 和 `continue` 参数。
1. 查询 500 个 Pod，指定 "limit=500"。
   ```http
   GET /api/v1/pods?limit=500
   ---
   200 OK
   Content-Type: application/json
   
   {
     "kind": "PodList",
     "apiVersion": "v1",
     "metadata": {
       "resourceVersion":"10245",
       "continue": "ENCODED_CONTINUE_TOKEN",
       ...
     },
     "items": [...] // returns pods 1-500
   }
   ```
2. 继续前面调用，查询剩余的 Pod，指定 "continue=ENCODED_CONTINUE_TOKEN" 表明继续上一次查询。
   ```http
   GET /api/v1/pods?limit=500&continue=ENCODED_CONTINUE_TOKEN
   ---
   200 OK
   Content-Type: application/json
   
   {
     "kind": "PodList",
     "apiVersion": "v1",
     "metadata": {
       "resourceVersion":"10245",
       "continue": "", // continue token is empty because we have reached the end of the list
       ...
     },
     "items": [...] // returns pods 1001-1253
   }
   ```

注意，当你使用 continue 继续查询时，**查询的永远是同一个 resourceVersion 的对象**。

### 4.3 对象表现形式
默认 Kubernetes 都是通过 JSON 格式来进行数据的传输。通过下面方式，允许 client 使用 protobuf 形式进行数据传输。

* 请求数据时，添加头部 `Accept: application/vnd.kubernetes.protobuf` 表明**期望返回 protobuf 类型数据**；
* 发送数据时，通过指定 `Content-Type: application/vnd.kubernetes.protobuf` 表明传**输的是一个 protobuf 类型数据**；

不过，部分 CRD 或者 API 扩展加入的资源不支持 Protobuf，可以使用 Accept 头部指定多种类型来允许回退。
```head
Accept: application/vnd.kubernetes.protobuf, application/json
```

### 4.4 对象的删除
资源对象删除经过两个阶段：
1. **finalization 终止**

   资源的 `metadata.deletionTimestamp` 被设置，然后随机顺序执行 Finalizers。
2. **delete 删除**

   当所有 Finalizers 执行结束后，资源才从 etcd 中被真正删除。


## 5 访问控制
Kubernetes 对 API 有着三种访问控制方式：
* [认证]^(Authentication) 
* [授权]^(Authorization) 
* [准入控制]^(Admission Control) 

这里仅仅会介绍三种方式的概念，更多的细节后面才深入去学习。

### 5.1 Authentication
Authentication 用于验证请求者是否是合法的，也就是检查身份。

每次收到请求时，APIServer 都会通过令排链进行认证，某一个认证成功即可：
* x509 处理程序 ：认证 HTTP 请求的证书是否有效；
* bearer token 处理程序 ：验证 token 是否有效；
* 基本认证处理程序 ：确保 HTTP 请求的基本认证凭证与本地的状态匹配；

### 5.2 Authorization
Authorization 用于验证请求者是否由对应操作的权限，也就是检查权限。

APIServer 会组合一系列授权方法，只要有一个授权者批准了操作，那么请求就可以继续。
> 使用哪些授权方法，可以在 APIServer 启动时通过 `--authorization_mode` 参数设置

目前支持的几种授权方法：
* webhook ：允许集群外的 HTTP(s) 服务进行授权；
* ABAC ：执行静态文件中定义的策略；
* RBAC ：通过 RBAC 机制进行授权，这允许管理员进行动态的配置；
* Node ：确保 kubelet 只能访问自己节点上资源；

### 5.3 Admission Control
Admission Control 是作为对授权模块的补充，用于拦截请求以确保其符合集群的期望与规则。

准入控制仅仅对于对象的创建、删除、更新、连接请求其作用，对于对象读取操作不起作用。

只有所有的准入控制器都通过，请求才能继续。如果任一准入控制器检查不通过，请求就会被立即拒绝。

准入控制是最后的关卡，一旦通过后，资源对象就会写入 etcd 中。


## 参考
* [**Blog：Kubernetes API 的版本控制，分组，对象，访问控制**](https://blog.csdn.net/qq_33121481/article/details/107331450) 
* [**官方文档：Kubernetes API Concepts](https://kubernetes.io/zh/docs/reference/using-api/api-concepts/)
* [**Blog: what-happens-when-k8s](https://github.com/jamiehannaford/what-happens-when-k8s)

