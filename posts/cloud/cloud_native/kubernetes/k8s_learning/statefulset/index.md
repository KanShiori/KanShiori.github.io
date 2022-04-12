# Kubernetes StatefulSet


## 1 概述

Pod 在设计的理念上是**无状态**的，Pod 可以在任意时刻被销毁，可以在任意时刻被在别的节点创建相同的副本。

Deployment 与 RelicSets 就是基于这个理念设计的，它们仅仅保证 Pod 的副本数量，而不关心其他。

对于大多数程序，其都是需要 “状态” 的，也就是重启能够恢复之前的数据。这个状态可能包括：

* **存储**
* **网络**
* **启停顺序**

### 1.1 “状态” 是什么

对于存储很好理解，大多数程序会将数据保存到磁盘或者云盘，而重启后重新读取数据，来恢复状态。所以，**Pod 重启前后读取到的存储要是一样的**。

对于网络，基于 Service 的存在，一般的后端服务可以不用关系其所处于的网络环境。但是对于分布式程序或者 p2p 程序，每个程序是需要知道其他程序的网络地址的。所以，**Pod 重启前后的网络地址要是一样的**。

还有一点是基于 Pod 之间的关系，有时候**多个 Pod 实例之间的启停顺序也应该是一样的**。

### 1.1 如何保存 “状态”

对于存储，PVC 是与 Pod 生命周期不耦合的，所以我们让 Pod 重启前后都使用同一个 PVC，那么其数据就是相同的。所以问题变为了：**建立 Pod 与 PVC 的映射关系**。

对于网络，Pod 的 IP 不是恒定的，这个是 Pod 的基本概念，无能更改。所以问题变为了：**要找到一个可以代表这个 Pod 的恒定网络地址**。

对于启动顺序，要保证 Pod 之间的运行顺序，那么需要认得每一个 Pod，也就是 Pod 的命名必须是有规范与固定的。

所有的问题都由固定的 Pod 命名来解决，Pod 会按照 0-N 的方式来进行命名，而这样 Pod0 可以固定到 Pod0 PVC，固定到 Pod0 DNS 域名，启动顺序按照 Pod0 Pod1 … 来启动。

这就是 StatefulSet 做的事情，总结一下：

* **固定的持久化存储**，通过 PVC；

* **固定的网络标识**，通过 [**Headless Service**](../2-service//#5-headless-service) 使得 `<podname>.<service>.<namespace>` 与固定命名的 Pod 绑定；

* **按照编号进行有序的启动与停止**；

## 2 StatefulSet 基础

### 2.1 Spec

StatefuleSet 基本定义如下：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
      storageClassName: shared-ssd-storage
```
* `serviceName` ：使用的 Headless Service 命名；
* `replicas` ：副本数量；
* `selector` ：Pod selector，决定要管理哪些 Pod；
* **`template`** ：Pod template；
* **`volumeClaimTemplates`** ：创建的 PVC 模板；
  * `metadata.name` ：创建的 PVC 基础命名，命名以 `<name>-<podname>` 格式；
  * `spec` ：与创建一个 PVC 的 spec 相同；

### 2.2 Status

StatefulSet 会将升级与版本信息记录到 status 字段：

```yaml
status:
  collisionCount: 0
  currentReplicas: 3
  currentRevision: my-cluster-69545dcd7d
  observedGeneration: 1
  readyReplicas: 3
  replicas: 3
  updateRevision: my-cluster-69545dcd7d
  updatedReplicas: 3
```
* currentReplicas ：当前版本的 Pod 数量；
* currentRevision ：当前版本的标识；
* updatedReplicas ：升级完成的 Pod 数量；
* updateRevision ：需要升级的版本标识；

而在对应的 Pod label 中，`controller-revision-hash` 会保存着其对应的 Revision，也就是创建其 Pod 的 StatefulSet 添加的：
```yaml
metadata:
  labels:
    controller-revision-hash: my-cluster-69545dcd7d
```

通过对比 Pod 的 `controller-revision-hash` label 与 StatefulSet 保存的 `updateRevision` 就可以判断这个 Pod 是否已经升级过了。因此，StatefulSet 可以不断对未升级的 Pod 执行升级操作。

### 2.3 使用

注意事项：
* PV 或者 StorageClass 需要预先创建，不会自动创建与删除；
* 虽然 PVC 是由 StatefulSet 自动创建的，但是不会进行自动删除，这是为了保留数据；
* Headless Service 需要预先创建，不会自动创建与删除；
* 删除 StatefulSet 不会使其清理所有的 Pod，应该通过将 .spec.replicas 设置为 0 来让其清理所有的 Pod；

### 2.4 Pod 管理策略

通过 `spec.podManagementPolicy` 可以设置管理的 Pod 的策略：
* **OrderedReady** ：默认策略，按照 Pod 的顺序依次创建或停止，前一个 Pod 完成后才会进行下一个 Pod 的操作；
* **Parallel** ：所有 Pod 创建或停止都是并行的，也就是说你不需要启停顺序的特性；

### 2.5 Pod 升级策略

通过 `spec.updateStrategy` 可以设置 Pod 的升级策略，也就是当你更新 `spec.template` 的镜像版本时，触发的操作。
```yaml
# …
spec:
  updateStrategy:
    rollingUpdate:
      partition: 3
    type: RollingUpdate
```
* `type` 指定升级策略
* `updateStrategy` ：指定 RollingUpdate 时控制滚动升级行为

目前支持两种策略：
* **OnDelete** （默认）：需要升级时，并不会自动删除旧版本 Pod，需要用户手动停止旧版本的 Pod，才会创建对应新版本 Pod。
* **RollingUpdate** ：自动删除旧版本的 Pod，并创建新版本 Pod。管理方式依赖于 Pod 管理策略。

#### 2.5.1 Partitions

设置 `spec.updateStrategy.rollingUpdate.partition` 可以对 Pod 进行**分区管理**。只有序号大于或等于的 Pod 才会进行滚动升级，其他 Pod 保持不变（即使被删除后重新创建）。

```bash
# 设置 partition 为 1
$ kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":3}}}}'
statefulset "web" patched

# 更新 StatefulSet
$ kubectl patch statefulset web --type='json' -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"gcr.io/google_containers/nginx-slim:0.7"}]'
statefulset "web" patched

# 验证更新，可以看到从 web-1 web-2 都升级了，而 web-0 没有变化
kubectl get pods -n mytest
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          3m2s
web-1   1/1     Running   0          30s
web-2   1/1     Running   0          50s
```

## 总结
StatefulSet 在原理上设计的让人感觉很优雅，仅仅从固定的 Pod 出发来实现 “有状态” 这个复杂的事情。

不过也许还有好多场景 StatefulSet 也无法满足，可能需要更多的开发。

需要弄清楚以下几点：
* [**为了需要有状态应用**](#1.1-“状态”是什么)
* [**如何为 Pod 实现有状态**](#1.1-如何保存“状态”)
* [**StatefulSet 基本定义与使用**](#2.1-定义)
* [**StatefulSet 的 Pod 管理策略**](#2.3-Pod-管理策略)
* [**StatefulSet 的升级策略**](#2.4-Pod-升级策略)

## 参考
* [**《Kubernetes 指南》**](https://kubernetes.feisky.xyz/concepts/objects/statefulset)

