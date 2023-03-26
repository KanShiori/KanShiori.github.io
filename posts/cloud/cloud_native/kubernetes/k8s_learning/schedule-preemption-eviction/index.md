# Kubernetes - Pod 调度机制


## 1 概述

无论是基本的副本控制器，还是自定义资源，其控制的底层 Pod 的调度都是都通过 Scheduler 完成的。

## 2 Schedule

### 2.1 nodeSelector 

Pod 的 `spec.nodeSelector` 可以用于控制 Pod 能被调度到哪些节点上。其内容是一组 kv 键值对，**只有节点 label 包含所有设定的 kv，才可以被调度 Pod**。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd  # 只有 label 包含 disktype:ssd 的节点才能被调度
```

除了你手动为节点添加 label 外，每个节点会默认添加上一些 label：

* kubernetes.io/hostname
* failure-domain.beta.kubernetes.io/zone
* failure-domain.beta.kubernetes.io/region
* topology.kubernetes.io/zone
* topology.kubernetes.io/region
* beta.kubernetes.io/instance-type
* node.kubernetes.io/instance-type
* kubernetes.io/os
* kubernetes.io/arch

### 2.2 nodeName

`spec.nodeName` 是最简单的选择节点方法，**指定 Pod 只能在一个指定节点上运行**。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: kube-01  # 指定调度到节点 kube-01
```

### 2.3 affinity

#### 2.3.1 nodeAffinity

`spec.affinity.nodeAffinity` 与 nodeSelector 类似，可以**根据节点的 label 来控制 Pod 调度到哪些节点**。

目前包含两种类型的节点亲和性：

* **requiredDuringSchedulingIgnoredDuringExecution** ：指定调度到的节点必须满足的条件，与 nodeSelector 一样但是表达性更高；

* **preferredDuringSchedulingIgnoredDuringExecution** ：指定调度到节点的偏好条件，也就是优先调度到满足条件的节点；

{{< admonition note "只影响调度">}}
目前两种类型节点亲和性都仅仅影响调度时的选择，而不会驱逐已经运行的 Pod。
{{< /admonition >}}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # 必须满足条件
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:  # 优先级条件
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

* nodeSelectorTerms 下的数组之间是 “或” 关系，也就是满足其中一个条件就可以被调度。
* matchExpressions 下的数组之间是 “与” 关系，需要满足所有条件才可以被调度。
* weight 字段范围 1-100，如果满足其指定的条件，那么节点优选算分时就会加上 weight 的值。

#### 2.3.2 Pod 亲和性与反亲和性

`spec.affinity.podAffinity` 亲和性允许根据节点上已经运行的 Pod 的 label 来控制是否调度到该节点。

Pod 亲和性也包含两种类型：

* **requiredDuringSchedulingIgnoredDuringExecution** ：必须满足的条件

* **preferredDuringSchedulingIgnoredDuringExecution** ：优选的条件

`spec.affinity.podAntiAffinity` 与亲和性相反，表明将 Pod 尽量与其他 Pod 分开部署。

对于 Pod 亲和性与反亲和性，判断范围都是针对拓扑域来说的。通过 topologyKey 指定判断拓扑域的 label，具有相同 <label>:<v> 的节点会认为属于用一个拓扑域下：

```yaml
# 如果两个节点具有相同的 topology.kubernetes.io/zone:<val> 的 label，那么它们属于同一个拓扑域。
topologyKey: topology.kubernetes.io/zone 
```

所以，节点亲和性的规则为：**对将被调度的节点，如果其相同拓扑域下的某个节点运行着满足条件的 Pod，那么就可以调度到该节点**。

对应的，节点反亲和性的规则为：**对将被调度的节点，如果其相同拓扑域下的某个节点运行着满足条件的 Pod，那么就尽量不要调度到该节点**。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"  # 拓扑域为节点
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname" 
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

上面例子中，podAntiAffinity 表明不要调度到同节点已经运行着 app:web-store 的 Pod 的节点上，podAffinity 表明调度到同节点运行着 app:store 的 Pod 的节点上。通俗点说，该 Pod 不能重复部署在同一个节点，并且每次部署要与 app:store 的 Pod 绑定。

### 2.4 Taint 与 Tolerations

与 affinity 相反，taint 使节点排斥一类特定的 Pod。

为了能使 taint 节点能够被调度到一些特殊的 Pod，可以设置 Pod 的 toleration，表明不在意某些节点的 taint 。

#### 2.4.1 Taint

通过 `kubectl taint` 为节点增加一个 Taint：

```bash
$ kubectl taint nodes node1 key1=value1:NoSchedule
```

* 为 node1 添加 key1:value1 的 Taint，其触发的效果是不能被调度（NoSchedule）

当然，你也可以为删除某个节点的 taint：

```bash
kubectl taint nodes node1 key1=value1:NoSchedule-
```

* 结尾的 - 号表示是删除一个 taint；

设置的 k/v 对用于来判断 Toleration 是否匹配 Taint。

目前包含几种类型的 effect：

* **NoSchedule** ：不将 Pod 调度到该节点，但是不影响已经运行的 Pod；
* **PreferNoSchedule** ：尽量不降 Pod 分配到该节点，是个软性条件；
* **NoExecute** ：不将 Pod 调度到该节点，并且会驱逐已经运行并且不能容忍污点的 Pod；

当达到条件时，Kubernetes 可能会给 Node 添加某个 Taint，包括：

* `node.kubernetes.io/not-ready` + NoExecute - 节点为准备好，Ready 为 false；
* `node.kubernetes.io/unreachable` + NoExecute - 节点不可达，Ready 为 unknown；
* `node.kubernetes.io/memory-pressure` + NoSchedule - 节点存在内存压力；
* `node.kubernetes.io/disk-pressure` + NoSchedule - 节点存在磁盘压力；
* `node.kubernetes.io/pid-pressure` + NoSchedule - 节点 PID 压力；
* `node.kubernetes.io/network-unavailable` + NoSchedule - 节点网络不可用；
* `node.kubernetes.io/unschedulable` + NoSchedule - 节点不可调度；
* `node.cloudprovider.kubernetes.io/uninitialized`+ NoSchedule - 节点未被云平台初始化；

{{< admonition note "DaemonSet 创建的 Pod">}}
DaemonSet 创建的 Pod 会自动加下面两个 NoExecute 的 Taint，使得其 Pod 不会被驱逐。

* `node.kubernetes.io/unreachable`
* `node.kubernetes.io/not-ready`
{{< /admonition >}}

#### 2.4.2 Tolerations

当 Pod 设置的 `spec.tolerations` 能够 “匹配” 节点某个 Taint 时，就可以认为该 Taint 不存在。

“匹配” 有两个含义：

* 如果 `operator` 是 Exist，那么相同的 key 即可。如果 `operator` 为 Equal，那么 key val 都要相同
* `effect` 相同

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```

* 可以容忍 key 为 "example-key"，effect 为 "NoSchedule" 的 taint；

通过 `spec.tolerations.tolerationSeconds` 可以指定匹配到容忍的污点后，能够持续容忍的时间。

```yaml
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoExecute"
    tolerationSeconds: 3600  # 匹配到 taint 后，3600 内不会被驱逐
```

### 2.5 Topology Spread Constraints 

Pod 提供了 `spec.topologySpreadConstraints` 字段来描述多个副本之间的拓扑关系。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  topologySpreadConstraints:
    - maxSkew: <integer>
      minDomains: <integer> # 可选；自从 v1.25 开始成为 Beta
      topologyKey: <string>
      whenUnsatisfiable: <string>
      labelSelector: <object>
      matchLabelKeys: <list> # 可选；自从 v1.25 开始成为 Alpha
      nodeAffinityPolicy: [Honor|Ignore] # 可选；自从 v1.26 开始成为 Beta
      nodeTaintsPolicy: [Honor|Ignore] # 可选；自从 v1.26 开始成为 Beta
```

* `maxSkew` - 描述最大能够接受不同区域间副本的偏差值。
  
  例如，如果 `maxSkew` 为 1，存在三个部署区域，那么最大副本数量与最小副本数量最大差值为 1。

* `minDomains` - 

* `topologyKey` - 划分区域时使用的 Node Label，相同 Label Value 的 Node 将被认为是同一个区域。
  
  例如，通常可能会使用 `topology.kubernetes.io/zone` 来划分区域，表示 Zone 之间副本数量的要求。

* `whenUnsatisfiable` - 指定不满足分布约束时的处理方式：

  * **DoNotSchedule** - 调度器不调度，Pod 处于 Pending 状态
  * **ScheduleAnyway** - 仍然调度，调度器会尽量满足分布约束

* `labelSelector` - 计算各个区域的 Pod 副本数时，使用的 Label Selector

* `matchLabelKeys` - 与 `labelSelector` 类似，但是是匹配的 Label Value 来计算 Pod 数量

## 3 Eviction

### 3.1 节点压力驱逐

kubelet 会监控 CPU、Mem、磁盘空间、文件系统 inode 数量等资源，一旦某个资源消耗达到一个阈值，**kubelet 会主动驱逐节点上的一个或多个 Pod，以回收资源**。

{{< admonition note Note>}}
压力驱逐不同于 API 驱逐，不会参考 `PodDisruptionBudget` 或者 Pod 的 `terminationGracePeriodSeconds`。
{{< /admonition >}}

驱逐时，kubelet 会**将 Pod 设置为 Failed 状态，并停止 Pod**，而上层的副本控制器可能会在其他地方创建 Pod 来替代。

驱逐方式分为：

* **Soft Eviction Thresholds**
  
  达到软阈值后并持续了一段时间没有恢复，就会通过 Graceful 的方式驱逐一些 Pod，但是等待时间是参考 kubelet 配置而不是 Pod 配置。

  通过 kubelet 的配置参数来指定软驱逐的阈值。

* **hard eviction thresholds**
  
  一旦触发阈值，kubelet 立刻杀死 Pod。

  kubelet 默认有以下的硬驱逐条件：

  * `memory.available<100Mi`
  * `nodefs.available<10%`
  * `imagefs.available<15%`
  * `nodefs.inodesFree<5%`

#### 3.1.1 Node Condition

kubelet 会将 Node 状态通过 Condition 方式暴露出来（默认 10s），以提供驱逐的信息：

* **MemoryPressure** - Node Mem 已经满足驱逐条件。

* **DiskPressure** - Node Fs 或者 Image Fs 或 Inode 已经满足驱逐条件。

* **PIDPressure** - PID 以满足驱逐条件

#### 3.1.2 如何选择被驱逐的 Pod

kubelet 会按照下面参数来决定哪个 Pod 被驱逐：

1. Pod 资源使用量是否超过 `spec.request`。
   
2. Pod 优先级。

3. Pod 相对于 `spec.request` 的资源使用情况；

因此，kubelet 会按照下面顺序进行驱逐：

1.  如果 BestEffort 或者 Burstable Pod 资源使用量超过 request。超出的越多的 Pod 优先被驱逐；
2.  如果都是 Guaranteed 和 Burstable Pod 并小于 request，那么基于 Pod Priority 驱逐。

> BestEffort Burstable Guaranteed 是 [**QosClass**](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)。

#### 3.1.3 最小驱逐回收

某些情况下，驱逐 Pod 可能只能回收少量资源，导致 kubelet 会反复驱逐。

通过配置 kubelet，可以让其在驱逐回收资源时，至少回收多少资源才停止驱逐。

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
evictionHard:
  memory.available: "500Mi"
  nodefs.available: "1Gi"
  imagefs.available: "100Gi"
evictionMinimumReclaim:
  memory.available: "0Mi"
  nodefs.available: "500Mi"
  imagefs.available: "2Gi"
```

上面配置表明，NodeFS 驱逐回收时，至少回收 500Mi。

### 3.2 API 驱逐

与节点压力驱逐的不同，API 驱逐是指通过 [**Eviction API**](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#create-eviction-pod-v1-core) 来进行主动的驱逐，并且停止 Pod 是 Graceful Stop。

例如，`kubectl drain` 就是通过 API 进行驱逐，停止某个节点上的所有 Pod。

API 驱逐会受到 [**PodDisruptionBudgets**](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) 和 [**terminationGracePeriodSeconds**](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-termination) 的控制。

### 3.3 污点驱逐

在第 2 部分看到，自定义的污点也会导致 Pod 的驱逐，不能容忍 NoExecute 污点的 Pod 都会被驱逐。

## 4 Preemption

Pods 可以被提供一个优先级。高优先级的 Pod 会被优先调度，Scheduler 甚至会尝试抢占低优先级的 Pod，来让高优先级的 Pod 先运行。

要使用优先级与抢占功能：

1. 创建 PriorityClass。
  
2. Pod 或者 Pod Template 定义中指定 `spec.priorityClassName` 为一个特定的 PriorityClasses。

### 4.1 PriorityClass

**`PriorityClass`** 包含一个 `value` 来描述优先级，值越大优先级越高。

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

{{< admonition note Note>}}
Kubernetes 内置两个 PriorityClass：`system-cluster-critical` 和 `system-node-critical`，表明是系统关键的组件。
{{< /admonition >}}

* `value` - 优先级值，32 位整型，越大表示优先级越高。
  
* `globalDefault` - 是否是系统默认优先级，没有指定 PriorityClass 的 Pod 使用默认优先级。
  
  如果系统没有设置 globalDefault，那么默认优先级是 0。

* `description` ：文本描述。

### 4.2 Pod 优先级

创建 Pod 时通过指定 `spec.priorityClassName` 来指定一个特定的 PriorityClass。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

当 Pod 优先级设置后，**scheduler 会按照优先级对 Pending Pods 进行排序，高优先级的 Pending Pod 优先于低优先级的进行处理**。

如果高优先级的 Pod 无法被调度到节点，那么 Scheduler 才会继续调度到低优先级 Pod。

### 4.3 非抢占式 PriorityClass

PriorityClass 支持一个配置 `preemptionPolicy` 配置抢占的行为。

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-nonpreempting
value: 1000000
preemptionPolicy: Never
globalDefault: false
description: "This priority class will not cause other pods to be preempted."
```

目前支持：

* **PreemptLowerPriority** - 允许使用该 PriorityClass 的 Pod 抢占其他优先级的 Pod。

* **Never** - 使用该 PriorityClass 放置在调度队列的中较低优先级的 Pod 之前，但是不能抢占其他 Pod

### 4.3 抢占

当 Scheduler 发现一个 Pod 无法被调度到任何节点时，就会触发抢占的逻辑。Scheduler 会尝试计算：**是否移除某个节点的一个或多个低优先级的 Pod，使得节点能够满足被调度的条件**。

如果能够找到该节点，新 Pod 状态信息中的 `nominatedNodeName` 为被设置为 Node Name，使得用户可以看到抢占信息。

之后，Node 被低优先级的 Pod 会被驱逐（Graceful Stop）。因此，这里会导致新 Pod 调度到该 Node 之间有一个需要等待的时间差。所以，Nominated Node 这不代表新 Pod 必定会调度到该节点，也许驱逐期间出现别的节点满足调度条件，那么就会被调度。

如果新 Pod 与将被驱逐的 Pod 之间有 Pod Affinity 关系，那么抢占后亲和性关系就不再会被满足，因此 Scheduler 不会选择这样的节点来进行抢占。同样，推荐在同优先级或者高优先级的 Pod 间设置 pod affinity。

同样，针对拓扑域下的 Pod Affinity，也会有上述的问题，因此 Scheduler 不会进行跨节点抢占。

## 5 Pod Qos

Pod 中的 `status.qosClass` 表明了 Pod 的服务质量。Kubernetes 在创建 Pod 时会将其设置到 Pod 上。

Qos 类别包括：

* Guaranteed
* Burstable
* BestEffort

### 5.1 Guaranteed Pod

Guaranteed 表示 Pod 的资源收到最严格的控制。其要求包括：

* Pod 中每个 Container 都必须指定 Mem Request 和 Mem Limits。
* Pod 中每个 Container 的 Mem Request 必须等于 Mem Limits。
* Pod 中每个 Container 都必须指定 CPU Request 和 CPU Limits。
* Pod 中每个 Container 的 CPU Request 必须等于 CPU Limits。

每个 Container 包括 InitContainer 和普通 Container。不过 Ephemeral Container 不支持配置资源，所以不受限制。

```yaml
spec:
  containers:
    # ...
    resources:
      limits:
        cpu: 700m
        memory: 200Mi
      requests:
        cpu: 700m
        memory: 200Mi
    # ...
status:
  qosClass: Guaranteed
```

### 5.2 Burstable Pod

Burstable 指定的 Pod 的资源能一定程度上的超频，但是还是有着上限的控制。其要求包括：

* Pod 不符合 Guaranteed 标准。
* Pod 中至少一个 Container 具有 Mem Request/Limits 或者 CPU Request/Limits

```yaml
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: qos-demo-2-ctr
    resources:
      limits:
        memory: 200Mi
      requests:
        memory: 100Mi
  # ...
status:
  qosClass: Burstable
```

### 5.3 BestEffort Pod

BestEffort 表明 Pod 没有收到任何的资源限制，也就是没有配置任何的 Mem 或 CPU 配置。

```yaml
spec:
  containers:
    # ...
    resources: {}
  # ...
status:
  qosClass: BestEffort
```

### 5.4 Qos 对于 OOMKill 的影响

kubelet 会根据 Qos 等级为每个 Container 设置一个 `oom_score_adj` 的值。

* Guaranteed 值为 -997
* BestEffort 值为 1000
* Burstable 值为 `min(max(2, 1000 - (1000 * memoryRequestBytes) / machineMemoryCapacityBytes), 999)`

当 Node 出现内存压力时，内核的 OOMKill 就会参考对应的分值去 Kill 进程。分值越高，优先级越低。因此优先会 Kill Guaranteed 的 Container。

## 6 PodDisruptionBudget

根据 Pod 销毁的场景，Kubernetes 将其分为了两个概念：

* Involuntary Disruptions
  
  Pod 环境出现异常，导致 Pod 不得不被销毁。例如：

  * Node 硬件故障
  * Node 失联
  * Node 资源不足，导致 Pod 被驱逐

* Voluntary Disruptions
  
  由 Pod 管理员或程序发起的主动操作。例如：

  * 更新了 Deployment
  * 执行了 Drain Node

因为 Involuntary Disruptions 是未知的，因此只能通过一些高可用的方式来避免。而对于 Voluntary Disruptions，Kubernetes 提供了 PodDisruptionBudget 来作为一个全局的限制，后续称为 PDB。

PDB 基于 Eviction API 来进行限制，因此只有驱逐操作时才会触发 PDB 的检查。因此，使用 PDB 后应该通过 Eviction API 来改变 Pod 数量。

{{< admonition note Note>}}
PDB 仅仅限制 Evict 的操作，典型的就是使用 Drain Node 操作。
{{< /admonition >}}

### 6.1 Spec 与 Status

PDB 定义如下：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: zookeeper
```

`selector` 就是用于筛选哪些 Pod 会收到管理。如果为空表明集群所有的 Pod。

PDB 基于两种方式来控制 Pod 数量（只能选择其中一个）：

* `minAvailable` - 驱逐后，必须保证可用的 Pod 最小数量，支持数值与百分比。

* `maxUnavailable` - 驱逐后，允许不可用的 Pod 最大数量，支持数值与百分比。

如果使用 Kubernetes 内置的 Workload（例如 Deployment、StatefulSet 等），那么上述两中方式都能使用。使用其他管理 Pod 方式时，有一些其他的限制：

* 只允许使用 `minAvailable`，并且只能使用数值。

PDB 的状态中可以看到一些相关的信息：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
# …
status:
  currentHealthy: 3
  desiredHealthy: 2
  disruptionsAllowed: 1
  expectedPods: 3
  observedGeneration: 1
```

### 6.2 PodDisruptionConditions

开启 Feature Gate 后，会给 Pod 添加一个 `DisruptionTarget` 的 Condition，用来表明 Pod 因为发生 Disruptions 而被删除。

Condition 中的 `reason` 字段会给出 Pod 被终止的具体原因，包括：

* **PreemptionByKubeScheduler**
  
  Pod 被抢占，用于接受优先级更高的 Pod。

* **DeletionByTaintManager**

  Pod 不能容忍 Node 的 `NoExcute` Taint，导致 Pod 被驱逐。

* **EvictionByEvictionAPI**

  Pod 被 Kubernetes API 进行驱逐。

* **DeletionByPodGC**

  Node 不存在了，导致 Pod 被 GC 删除。

* **TerminationByKubelet**

  Pod 由于 Node 压力驱逐或者 Node 关闭被 kubelet 终止。

## 7 Pod Overhead

运行 Pod 时，除了 Pod 程序占用的内存外，Pod 环境本身可能占用一些系统资源。这些额外的资源可以通过定义 RuntimeClass 的 `overhead` 字段来指出。

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-fc
handler: kata-fc
overhead:
  podFixed:
    memory: "120Mi"
    cpu: "250m"
```

上面示例表明，CRI `kata-fc` 创建一个 Pod 会用到的 Mem 和 CPU，那么 Kubernetes 中计算 Pod Mem 和 CPU 会加上指定的值。

## 参考

* Blog：[**k8s 节点资源预留与 pod 驱逐**](https://segmentfault.com/a/1190000021402192) 
* 官方文档：[**Scheduling, Preemption and Eviction**](https://kubernetes.io/docs/concepts/scheduling-eviction/) 
* 官方文档：[**Pod Disruptions**](https://kubernetes.io/zh/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets)



