# Schedule Preemption Eviction


## 1 概述
无论是基本的副本控制器，还是自定义资源，其控制的底层 Pod 的调度都是都通过 scheduer 完成的。

## 2 调度

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
topologyKey: topology.kubernetes.io/zone  # 如果两个节点具有相同的 topology.kubernetes.io/zone:<val> 的 label，那么它们属于同一个拓扑域。
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

### 2.4 taint 与 tolerations
与 affinity 相反，taint 使节点排斥一类特定的 Pod。
为了能使 taint 节点能够被调度到一些特殊的 Pod，可以设置 Pod 的 toleration，表明不在意某些节点的 taint 。

#### 2.4.1 taint
通过 `kubectl taint` 为节点增加一个 taint：
```bash
$ kubectl taint nodes node1 key1=value1:NoSchedule
```
* 为 node1 添加 key1:value1 的 traint，其触发的效果是不能被调度（NoSchedule）

当然，你也可以为删除某个节点的 taint：
```bash
kubectl taint nodes node1 key1=value1:NoSchedule-
```
* 结尾的 - 号表示是删除一个 taint；

设置的 kv 对用于来判断 toleration 是否匹配 taint。

目前包含几种类型的 effect ：
* **NoSchedule** ：不将 Pod 调度到该节点，但是不影响已经运行的 Pod；
* **PreferNoSchedule** ：尽量不降 Pod 分配到该节点，是个软性条件；
* **NoExecute** ：不将 Pod 调度到该节点，并且会驱逐已经运行并且不能容忍污点的 Pod；

Kubernetes 会在一些条件下，自动会节点添加一些内置的污点：
* node.kubernetes.io/not-ready + NoExecute ：节点为准备好，Ready 为 false；
* node.kubernetes.io/unreachable + NoExecute ：节点不可达，Ready 为 unknown；
* node.kubernetes.io/memory-pressure + ：节点存在内存压力；
* node.kubernetes.io/disk-pressure + NoSchedule ：节点存在磁盘压力；
* node.kubernetes.io/pid-pressure + NoSchedule ：节点 PID 压力；
* node.kubernetes.io/network-unavailable + NoSchedule ：节点网络不可用；
* node.kubernetes.io/unschedulable + NoSchedule ：节点不可调度；
* node.cloudprovider.kubernetes.io/uninitialized + NoSchedule ：节点未被云平台初始化；

{{< admonition note "DaemonSet 创建的 Pod">}}
DaemonSet 创建的 Pod 会自动加上上面两个 NoExecute 的 taint，使得其 Pod 不会被驱逐。
{{< /admonition >}}
#### 2.4.2 tolerations
当 Pod 设置的 `spec.tolerations` 能够 “匹配” 节点某个 taint 时，就可以认为该 taint 不存在。

“匹配” 有两个含义：
* 如果 operator 是 Exist，那么相同的 key 即可。如果 operator 为 Equal，那么 key val 都要相同
* effect 相同

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

## 3 驱逐

### 3.1 节点压力驱逐
kubelet 会监控 CPU、Mem、磁盘空间、文件系统 inode 数量等资源，一旦某个资源消耗达到一个阈值，**kubelet 会主动驱逐节点上的一个或多个 Pod，以回收资源**。

驱逐时，kubelet 会**将 Pod 设置为 Failed 状态，并停止 Pod**，而上层的副本控制器可能会在其他地方创建 Pod 来替代。

驱逐的阈值分为：
* **soft eviction thresholds** ：达到软阈值后并持续了一段时间没有恢复，就会通过 graceful 的方式驱逐一些 Pod；
* **hard eviction thresholds** ：一旦触发阈值，立即通过 force 方式进行驱逐；
#### 3.1.2 配置驱逐参数
驱逐相关的配置参数都需要配置 kubelet 的启动参数来进行配置。
#### 3.1.3 驱逐策略
kubelet 会按照下面参数来决定驱逐 pod 的顺序：
1. Pod 资源使用量是否超过 spec.request；
1. Pod 优先级；
1. Pod 相对于 spec.request 的资源使用情况；
因此，kubelet 会按照下面顺序进行驱逐：。
1.  如果 BestEffort 或者 Burstable Pod 资源使用量超过 request。超出的越多的 Pod 优先被驱逐；
1.  如果都是 Guaranteed 和 Burstable Pod 并小于 request，那么基于 Pod Priority 驱逐。

> BestEffort Burstable Guaranteed 是 [**QosClass**](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)。

### 3.2 API 驱逐
与节点压力驱逐的不同，API 驱逐是指通过 [**Eviction API**](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#create-eviction-pod-v1-core) 来进行主动的驱逐，并且停止 Pod 是 graceful。

`kubectl drain` 就是通过 API 进行驱逐，停止某个节点上的所有 Pod。

API 驱逐会受到 [**PodDisruptionBudgets**](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) 和 [**terminationGracePeriodSeconds**](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-termination) 的控制。

### 3.3 污点驱逐
在第 2 部分看到，自定义的污点也会导致 Pod 的驱逐，不能容忍 NoExecute 污点的 Pod 都会被驱逐。

## 4 Preemption
Pods 可以被提供一个优先级。高优先级的 Pod 会被优先调度，scheduler 甚至会尝试抢占低优先级的 Pod，来让高优先级的 Pod 先运行。

要使用优先级与抢占功能：
1. 创建 PriorityClass；
1. Pod 或者 Pod template 定义中指定` spec.priorityClassName` 为一个特定的 PriorityClasses。

### 4.1 PriorityClass
**`PriorityClass`** 是一个 non-namespaced 资源，其包含一个 value 来描述优先级。
* Kubernetes 内置两个 PriorityClass ：system-cluster-critical system-node-critical，表明是系统关键的组件、

一个基本的 PriorityClass 优先级如下：
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```
* `value` ：优先级值，32 位整型，越大表示优先级越高。
* `globalDefault` ：是否是系统默认优先级，没有指定 PriorityClass 的 Pod 使用默认优先级；<br>
  如果系统没有设置 globalDefault，那么默认优先级是 0。
* `description` ：文本描述；

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
当 Pod 优先级设置后，**scheduler 会按照优先级对 pending Pods 进行排序，高优先级的 pending Pod 优先于低优先级的进行处理**。

如果高优先级的 Pod 无法被调度到节点，那么 scheduler 才会继续调度到低优先级 Pod。

### 4.3 抢占
当 scheduler 发现一个 Pod 无法被调度到任何节点时，就会触发抢占的逻辑。scheduler 会尝试计算：是否移除某个节点的一个或多个低优先级的 Pod，使得节点能够满足被调度的条件。

如果能够找到该节点，新 Pod 状态信息中的 `nominatedNodeName` 为被设置为节点名，使得用户可以看到抢占信息。

之后，节点上被低优先级的 Pod 会被驱逐（graceful stop 30s）。因此，这里会导致新 Pod 调度到该节点之间有一个需要等待的时间差。

所以，Nominated Node 这不代表新 Pod 必定会调度到该节点，也许驱逐期间出现别的节点满足调度条件，那么就会被调度。

如果新 Pod 与将被驱逐的 Pod 之间有 pod affinity 关系，那么抢占后亲和性关系就不再会被满足，因此 scheduler 不会选择这样的节点来进行抢占。同样，推荐在同优先级或者高优先级的 Pod 间设置 pod affinity。

同样，针对拓扑域下的 pod affinity，也会有上述的问题，因此 scheduler 不会进行跨节点抢占。


## 参考
* Blog：[**k8s 节点资源预留与 pod 驱逐**](https://segmentfault.com/a/1190000021402192) 
* 官方文档：[**Scheduling, Preemption and Eviction**](https://kubernetes.io/docs/concepts/scheduling-eviction/) 



