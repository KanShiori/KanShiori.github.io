# K8s 编程 - 1 - 基本概念


> **Kubernetes 编程系列** 主要记录一些开发 Controller 所相关的知识，大部分内容来自于[《Programming Kubernetes》](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)（推荐直接阅读）。


## 1 Controller 模式

### 1.1 Control Loop
所有 Controller 的基本逻辑都是执行 [控制循环]^(Control Loop)，其基本逻辑为：
1. 读取资源的状态。
   
   采用 Watch 方式监听资源，资源变更时会触发。

2. 改变集群中的对象的状态，或者集群外部对象的状态。
   
   例如，启动一个 Pod、创建云上资源等。

3. 通过 APIServer 更新资源的状态。
   
4. 循环执行这些逻辑。

{{< find_img "img1.png" >}}

### 1.2 事件驱动模式

大多数分布式系统都是 RPC 来触发一个动作，而 Kubernetes 中都会通过 APIServer 的 **Watch** 功能来监听对象的变化。

以通过一个 Deployment 来启动 Pod 为例：
1. Deployment Controller Watch 了 Deployment 的事件，因此触发后发现用户创建了一个 Deployment，执行相关的业务逻辑：创建一个 ReplicaSet 对象。
2. ReplicaSet Controller Watch 了 ReplicaSet 的事件，因此触发后发现新创建了一个 ReplicaSet，执行相关的业务逻辑：创建一个 Pod 对象。
3. Scheduler 本身也是 Controller，通过 Watch 机制发现有一个新创建的 Pod，并且 Pod 的 spec.nodeName 是空值，那么就会将其放入调度队列。
4. Scheduler 将 Pod 调度到一个合适的 Node，将 Node 更新到 Pod 的 spec.nodeName 字段，然后把它写入 APIServer。
5. kubectl Watch 了 Pod 事件，因此触发后发现 Pod 的 spec.nodeName 与自己所在 Node 相同，那么就会构建环境并启动 Pod。

**可以看到，所有的 Controller 都是运行着独立的 Control Loop，对象状态的变化通过会以事件的方式通知到 Controller。**

### 1.3 乐观并发

在系统中，Controller 可能存在着多个副本，因此并发的更新资源操作可能会产生写入冲突。

对此，Kubernetes APIServer 使用 [乐观并发]^(Optimistic Concurrency) 方式来解决冲突。

**简单地说，如果 APIServer 发现了两个并发的写请求，它就会拒绝后面的请求，返回一个错误，而发送请求的 Client 要处理这种失败。**
```go
var err error
for retries := 0; retries < 10; retries++ {
   foo, err = client.Get("foo", metav1.GetOptions{})
   if err != nil {
      break
   }

   // 更新操作 ...

   _, err = client.Update(foo)
   if err != nil && errors.IsConflict(err) {
      // 处理冲突 ...
      continue
   } else if err != nil {
      break
   }
}
```

问题是如何判断两个写操作是冲突的？答案是每个 Kubernetes 资源对象都会包含的 **`metadata.resourceVersion`** 字段。
```yaml
kind: Pod
metadata:
  name: foo
  resourceVersion: 57
spec:
  #...
status:
  #...
```
例如，你想要更新该 Pod 对象，其 `metadata.resourceVersion: 57`，APIServer 按此尝试将数据写入 etcd，而 etcd 中存储的该 Pod 为 `metadata.resourceVersion: 58`。 etcd 发现该资源版本与自己保存的不一致，那么就会拒绝写入。

写入冲突完全是很正常的事情，因此你编写的 Controller 代码中必须正确处理冲突的场景。

### 1.4 Operator
Operator 本质上就是一个或多个 Controller，只是比一般的 Controller 复杂，要负责一些产品的运维操作。

一般 Operator 包含：
* 一组 CRD，用于定义特定领域的 Schema。
* 一个或多个 Controller，用于完成特定的运维操作。

可以在 [**OperatorHub**](https://operatorhub.io/) 来查找特定领域的发布的 Operator。


## 2 API 基础

Kubernetes 的核心是 **`APIServer`**，其主要提供两个功能：
* 为所有组件提供 API。
* 提供 Proxy 功能。

{{< image src="img2.png" >}}

### 2.1 HTTP API
对于客户端来说，APIServer 提供了一个 RESTful HTTP API，可以返回 JSON/Protobuf 格式的内容。
{{< admonition tip 编码方式>}}
由于性能考虑，**集群内部组件通信主要使用 Protobuf**。而客户端访问为了通用性，默认使用 JSON 格式。
{{< /admonition >}}

基于 REST，每个 HTTP 请求的通过不同的 HTTP Verb（也称 Method）来表示不同类型的操作：
* HTTP GET - 查询一个或多个特定的资源；
* HTTP POST - 创建资源；
* HTTP PUT - 修改已有的资源，提供资源完整的定义；
* HTTP PATCH - 部分更新已有的资源，只提供资源部分定义即可；
* HTTP DELETE - 删除一个资源；

你可以通过 [**API 参考文档**](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/) 看到所有 HTTP API，以及对应的示例（注意版本）。

### 2.2 API 基本概念

#### 2.2.1 Kind
**`Kind`** 表示一个**对象的类型**，每个对象都会包含 Kind 字段（JSON 中为 kind，Golang 中为 Kind）。
```yaml
kind: Pod
```

目前存在着三种那类型的 kind：
* 特定的对象，例如 Pod；
* 对象的集合，例如 PodLists、NodeLists；
* 特定行为或者非持久化的实体，例如 /scale、Status；

#### 2.2.2 API Group

**`Group`** 为一组 **Kind 的集合**。例如，批量任务相关（Job 或者 ScheduledJob）都包含在 "batch" 这个 API Group。

#### 2.2.3 Version

每个 API Group 可以有着多个 **`Version`**。

特定版本的（比如 v1beta1）创建的对象，可以在其他支持的版本中获取并使用，**APIServer 支持将对象在多个版本间进行无损转换**。
{{< admonition note "Storage Version">}}
后续可以看到，APIServer 使用 "Storage Version" 作为多个版本之间的内部版本。
{{< /admonition >}}

#### 2.2.4 Resource

在 HTTP API Endpoint 中，会使用小写复数的单词，例如 .../pods。在 HTTP 中该 Endpoint 就被称为 **`Resource`**。

有些 Resource 有着更多的 Endpoint 来表示一些操作，例如 ../pod/nginx/port-forward、../pod/nginx/logs 等。这些 Endpoint 就被称为 SubResource。
{{< admonition note "CRD 中的 SubResource">}}
现在，可以理解为啥 CRD 中有个 `subresouces` 字段了：
```yaml
# ...
spec:
  group: stable.example.com
  versions:
    - name: v1
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas
```
{{< /admonition >}}

Kind 与 Resource 是不一样的概念，区别主要在于：
* Resource 会对应一个 HTTP API Path；
* Kind 是用于保存在对应定义中的，也会保存在 etcd 中。

大多数情况下，**Kind 与 Resource 是一对一的映射关系**。

## 参考
* [**OperatorHub**](https://operatorhub.io/)
* [**API 参考文档**](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/)
* [**《Programming Kubernetes》**](https://book.douban.com/subject/33393681/)
