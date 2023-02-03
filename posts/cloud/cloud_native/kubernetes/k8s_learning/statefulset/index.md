# Kubernetes - StatefulSet


## 1 概述

Pod 在设计的理念上是**无状态**的，Pod 可以在任意时刻被销毁，可以在任意时刻被在别的节点创建相同的副本。

Deployment 与 ReplicaSets 就是基于这个理念设计的，它们仅仅保证 Pod 的副本数量，而不关心其他。

对于大多数程序，其都是需要 “状态” 的，也就是重启能够恢复之前的数据。这个状态可能包括：

* **存储**
* **网络**
* **启停顺序**

### 1.1 “状态” 是什么

对于存储很好理解，大多数程序会将数据保存到磁盘或者云盘，而重启后重新读取数据，来恢复状态。所以，**Pod 重启前后读取到的存储要是一样的**。

对于网络，基于 Service 的存在，一般的后端服务可以不用关系其所处于的网络环境。但是对于分布式程序或者 p2p 程序，每个程序是需要知道其他程序的网络地址的。所以，**Pod 重启前后的网络地址要是一样的**。

还有一点是基于 Pod 之间的关系，有时候**多个 Pod 实例之间的启停顺序也应该是一样的**。

### 1.1 如何保存 “状态”

对于存储，PVC 是与 Pod 生命周期不耦合的，所以我们让 Pod 重启前后都使用同一个 PVC，那么其数据就是相同的。所以问题变为了：<important>建立 Pod 与 PVC 的映射关系</important>。

对于网络，Pod 的 IP 不是恒定的，这个是 Pod 的基本概念，无能更改。所以问题变为了：**要找到一个可以代表这个 Pod 的恒定网络地址**。

对于启动顺序，要保证 Pod 之间的运行顺序，那么需要认得每一个 Pod，也就是 Pod 的命名必须是有规范与固定的。

所有的问题都由固定的 Pod 命名来解决，Pod 会按照 0-N 的方式来进行命名，而这样 Pod0 可以固定到 Pod0 PVC，固定到 Pod0 DNS 域名，启动顺序按照 Pod0 Pod1 … 来启动。

这就是 StatefulSet 做的事情，总结一下：

* **固定的持久化存储**，通过 PVC；

* **固定的网络标识**，通过 [**Headless Service**](../service-and-endpoint/#6-headless-service) 使得 `<podname>.<service>.<namespace>` 与固定命名的 Pod 绑定；

* **按照编号进行有序的启动与停止**；

## 2 StatefulSet 基础

### 2.1 Spec 与 Status

StatefuleSet 基本定义如下：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx" # Headless Service 命名
  replicas: 2          # 副本数量
  selector:            # Pod Selector，决定要管理哪些 Pod
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
  volumeClaimTemplates: # 创建的 PVC 模板
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
      storageClassName: shared-ssd-storage
```

StatefulSet 会将升级与版本信息记录到 `status` 字段：

```yaml
status:
  collisionCount: 0
  observedGeneration: 1                  # 更新 Status 时的 STS 的 generation
  replicas: 3                            # 期望运行的 Pod 数量
  readyReplicas: 3
  currentReplicas: 3                     # 当前版本（升级前）的 Pod 数量
  currentRevision: my-cluster-69545dcd7d # 当前版本的标识（name + hash）
  updatedReplicas: 3                     # 升级版本的 Pod 数量
  updateRevision: my-cluster-69545dcd7d  # 升级版本的标识（name + hash）
```

通过这些字段可以推断出当前的升级进度，见 [**Kubernetes - Update 与 Rollback**](../rolling-upgrade/#143-statefulset)

### 2.2 Pod 序号

为了让每个 Pod 有着唯一标识，每个 Pod 会被分配一个整数序号。默认情况下为 0 到 N-1。

如果开启了 `StatefulSetStartOrdinal` Feature Gate，可以通过配置 `.spec.ordinals.start` 设置 Pod 的起始序号。

### 2.3 存储

对于 StatefulSet 中定义的 VolumeClaimTemplate，会为每个 Pod 创建对应的 PVC。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  volumeClaimTemplates: # 创建的 PVC 模板
  - metadata:
      name: www
    spec:
      resources:
        requests:
          storage: 1Gi
```

创建的 PVC 命名也是固定的格式：`<template_name>-<pod_name>`。因此，当 Pod 重建时，会关联上相同的 PVC，从而复用相同的 PV。

默认下，Pod 或者 StatefulSet 被删除时，对应的 PVC 与 PV 都不会被删除，需要手动删除。

#### 2.5.1 PVC Retention Policy

当开启 `StatefulSetAutoDeletePVC` Feature Gate 后，允许通过配置 `.spec.persistentVolumeClaimRetentionPolicy` 控制是否自动删除 PVC。

```yaml
apiVersion: apps/v1
kind: StatefulSet
# ...
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Delete
# ...
```

目前支持两个阶段的配置：

* **whenDeleted**
  
  配置 StatefulSet 被删除时，对于 PVC 的保留策略。注意，只在 StatefulSet 被删除时生效，Pod 重建依旧会复用 PVC。

  也就是说，支持了 StatefulSet 删除的同时删除 PVC。

* **whenScaled**
  
  配置 Pod 缩容时，对于 PVC 的保留策略。

每个阶段都支持如下策略：

* **Retain** - 默认值，保留 PVC

* **Delete** - Pod 被删除同时删除 PVC（不影响 PV）

### 2.4 网络

StatefulSet 中每个 Pod 会根据 StatefulSet 名称以及 Pod 序号设置 Pod 的 Hostname，格式为：`<sts_name>-<pod_number>`。

```yaml
apiVersion: v1
kind: Pod
spec:
  hostname: webapp-1
  # ...
```

根据[**生成 Pod DNS 的机制**](../dns-in-k8s/#12-自定义-hostname-和-subdomain)，那么创建对应的 Headless Service 后，Pod 就会得到一个固定的 DNS Domain。

注意，用户需要负责创建 Headless Service，StatefulSet 不会创建。

### 2.5 缩扩容

默认下，StatusfulSet 是依次扩容或缩容 Pod 的。缩扩容某个 Pod 前，它前面的所有 Pod 必须是 Running 和 Ready 状态。

通过 `spec.podManagementPolicy` 可以配置缩扩 Pod 的策略：

* **OrderedReady** ：默认策略，按照 Pod 的顺序依次创建或停止，前一个 Pod 完成后才会进行下一个 Pod 的操作。

* **Parallel** ：所有 Pod 创建或停止都是并行的，不会等待 Pod 进入 Running 和 Ready 或者完全删除。该策略仅仅影响缩扩容，不会影响升级流程。

### 2.6 升级

当用户更新 `spec.template` 时，就会触发 Pod 的升级操作。

`spec.updateStrategy` 用于设置 Pod 的升级策略：

```yaml
# …
spec:
  updateStrategy:
    rollingUpdate:
      partition: 3
    type: RollingUpdate
```

目前支持两种策略：

* **OnDelete** - 需要升级时，并不会自动删除旧版本 Pod，需要用户手动停止旧版本的 Pod，才会创建对应新版本 Pod。

* **RollingUpdate** - 默认策略，自动删除旧版本的 Pod，并创建新版本 Pod。管理方式依赖于 Pod 管理策略。

对于 RollingUpdate，StatefulSet Controller 会按照序号从大到小的顺序重建依次重建每个 Pod，每个 Pod 需要等待前面的 Pod 进入 Running 和 Ready 才会继续。

#### 2.6.1 Revision

`StatefulSet` 使用 Revision 来标识特定的版本，在管理的 Pod Label 中，`controller-revision-hash` 会保存着其对应的 Revision。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    controller-revision-hash: my-cluster-69545dcd7d
```

通过对比 Pod Label `controller-revision-hash` 与 StatefulSet Status `updateRevision` 就可以判断这个 Pod 是否已经升级过了。因此，StatefulSet 可以不断对未升级的 Pod 执行升级操作。


#### 2.6.2 Partitions

设置 `.spec.updateStrategy.rollingUpdate.partition` 可以对 Pod 进行**分区管理**。只有序号大于或等于的 Pod 才会进行滚动升级，其他 Pod 保持不变（即使被删除后重新创建）。

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

#### 2.4.2 Max Unavailable

指定 `.spec.updateStrategy.rollingUpdate.maxUnavailable` 来控制滚动升级期间，最大的不可用的 Pod 数量。

该值可以是绝对值（Pod 数量）或者百分比（Pod 期望数量百分比）。默认值为 1，表明只允许一个 Pod 不可用，所以默认滚动升级是 one by one 的。

当配置值大于 1 后，那么表明滚动升级每次可能升级多个 Pod。

### 2.7 Min Ready Second

`spec.minReadySeconds` 可以指定新创建的 Pod 在没有崩溃的情况下，运行多久后才被认为是可用的。

该功能对于平滑升级很有用。例如，有两个 Pod 提供服务，在升级第一个 Pod 后等待一段时间，使得第一个 Pod 接受流量并处理后，再升级下一个 Pod，防止流量瞬间掉低。

## 参考

* 官方文档：[**StatefulSet**](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/statefulset)
