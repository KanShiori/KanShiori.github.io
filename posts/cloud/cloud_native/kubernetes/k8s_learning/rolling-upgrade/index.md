# Kubernetes - Update 与 Rollback


## 1 操作

### 1.1 Update

当用户修改 Deployment DaemonSet StatefulSet 定义中的 `.spec.template` 任意字段时，就会触发 Pod 的升级，默认都是 Rolling Update。

```bash
$ kubectl set image deployment.v1.apps/nginx-deployment nginx=nginx:1.16.1
```

通过 `kubectl rollout status` 查看滚动升级的过程：

```bash
$ kubectl rollout status deployment/nginx-deployment

Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out
```

### 1.2 Rollback

使用 `kubectl rollout history` 来查看某个 Deployment DaemonSet StatefulSet 的修订历史。

```bash
$ kubectl rollout history deployment/nginx-deployment

deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

> 更新时使用 `--record=true` 参考可以记录更新的命令，并记录在 Change Cause 中。

使用 `--revision` 参数具体查看某个版本的 Pod template。

```bash
$ kubectl rollout history deployment/nginx-deployment --revision 2

deployment.apps/nginx-deployment with revision #2
Pod Template:
  Labels:       app=nginx
        pod-template-hash=ff6655784
  Containers:
   nginx:
    Image:      nginx:1.16.1
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```

使用 `kubectl rollout undo` 来回滚到上一个版本，或者使用 `--to-revision` 参数回滚到特定的版本。

```bash
$ kubectl rollout undo deployment/nginx-deployment

deployment.apps/nginx-deployment rolled back
```

{{< admonition note Note>}}
Rollback 本质上也是 Update，因此相关的控制参数都是适用的。
{{< /admonition >}}

### 1.3 流程控制

#### 1.3.1 Deployment

Deployment 升级流程中，会交替更新旧的与新的 ReplicaSet。有一些参数可以控制滚动升级。

`spec.strategy.type` 策略用于指定如何进行 Pod 升级：

* `RollingUpdate` - 默认值，滚动升级；
* `Recreate` - 先删除所有旧 Pod，后启动新 Pod；

在使用 `RollingUpdate` 策略下，可以通过下面两个字段来控制滚动升级的速率：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 10%
      maxSurge: 10%
```

* `maxUnavailable` - 指定升级过程中，不可用的 Pod 的个数上限。支持数字或者百分比（向下取整），默认值 25%。
  
  例如 `maxUnavailable: 30%` 时，那么滚动升级期间，会确保可用的 Pod 保持为总数的 70%。

  {{< image src="img1.png" height=200 >}}  

* `maxSurge` -  指定升级过程中，允许创建的超出期望的 Pod 的个数。支持数字或者百分比（向上取整），默认值为 25%。
  
  例如 `maxSurge: 30%` 时，表明升级过程中可能所有的 Pod 数量会是期望的 130%，然后再会交替的进行滚动升级。

  {{< image src="img2.png" height=200 >}}  

{{< admonition note Note>}}
Deployment 设计上是用于管理无状态应用的，因此滚动升级中通过多创建一些 Pod 作为 Buffer，以获得更好的升级体验。而 StatefulSet 相反，因为运行的是有状态的，因此整个滚动升级过程中，Pod 总数是不变的。
{{< /admonition >}}

其他一些相关的参数包括：

```yaml
spec:
  revisionHistoryLimit: 3
  minReadySeconds: 30
  paused: false
```

* `revisionHistoryLimit` - 保留的历史版本数量（也就是 ReplicaSet 数量），默认为 10。
  
  如果为 0 则表示不保留任何版本，并且会立即清理。

* `minReadySeconds` - 滚动升级过程中，新创建的 Pod 就绪后的等待时间。
  
  每当新创建的 Pod 为就绪状态后，等待 `minReadySeconds` 秒后会继续进行下一个 Pod 的升级。如果为 0，那么表明 Pod 就绪后立即可用。

* `paused` - 暂停 Deployment 的升级，修复 Pod template 不会触发滚动升级。
  
  使用 `kubectl rollout pause/resume` 本质上就是修改 Deployment 的该字段。

#### 1.3.2 DaemonSet

DaemonSet 代表着长期运行的守护程序，`spec.strategy.type` 支持:

* RollingUpdate -（默认值）滚动升级；
* OnDelete - 不自动升级，当 Pod 被删除时才会升级；

同样，DaemonSet 也支持滚动升级的速率控制：

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 10%
```

* `maxUnavailable` - 指定升级过程中，不可用的 Pod 的个数上限。支持数字或者百分比（向下取整），默认值 25%。
  
  例如 `maxUnavailable: 30%` 时，那么滚动升级期间，会确保可用的 Pod 保持为总数的 70%。

其他一些相关的参数包括：

```yaml
spec:
  revisionHistoryLimit: 3
  minReadySeconds: 30
```

* `revisionHistoryLimit` - 保留的历史版本数量（也就是 ControllRevision 数量），默认为 10。
  
  如果为 0 则表示不保留任何版本，并且会立即清理。

* `minReadySeconds` - 滚动升级过程中，新创建的 Pod 就绪后的等待时间，默认为 0。
  
  每当新创建的 Pod 为就绪状态后，等待 `minReadySeconds` 秒后会继续进行下一个 Pod 的升级。如果为 0，那么表明 Pod 就绪后立即可用。

#### 1.3.3 StatefulSet

与 DaemonSet 类似，StatefulSet 的 `spec.strategy.type` 支持：

* RollingUpdate -（默认值）滚动升级；
* OnDelete - 不自动升级，当 Pod 被删除时才会升级；

StatefulSet 的速率控制更加偏向于手动，基于分区的方式：

```yaml
spec:
  updateStrategy:
    rollingUpdate:
      partition: 3
    type: RollingUpdate
```

* `partition` - 只有 Pod 编号大于等于 `partition` 值的 Pod 才可以升级，其他 Pod 保持不变（删除也不会升级）。
  
  例如，4 副本并且 `partition: 2` 时，那么表明只有 pod-3 与 pod-2 会升级，而 pod-1 pod-0 不会升级。

StatefulSet 还支持并行地缩扩容，基于 `spec.podManagementPolicy` 配置：

* OrderedReady -（默认）按编号顺序依次创建或停止，前一个 Pod 完成后才会进行下一个 Pod 操作；
* Parallel - 所有 Pod 创建与停止是并行的；

```yaml
spec:
  revisionHistoryLimit: 3
  minReadySeconds: 30
```

* `revisionHistoryLimit` - 保留的历史版本数量（也就是 ControllRevision 数量），默认为 10。
  
  如果为 0 则表示不保留任何版本，并且会立即清理。

* `minReadySeconds` - 滚动升级过程中，新创建的 Pod 就绪后的等待时间，默认为 0。
  
  每当新创建的 Pod 为就绪状态后，等待 `minReadySeconds` 秒后会继续进行下一个 Pod 的升级。如果为 0，那么表明 Pod 就绪后立即可用。

### 1.4 流程观测

#### 1.4.1 Deployment

DaemonSet Status 中一些参数可以观察 Update 的状态：

```yaml
status:
  replicas: 3
  updatedReplicas: 3
  readyReplicas: 3
  availableReplicas: 3
  unavailableReplicas: 0
  conditions:
  - lastTransitionTime: "2022-05-16T12:13:27Z"
    lastUpdateTime: "2022-05-16T12:13:27Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2022-05-16T12:12:57Z"
    lastUpdateTime: "2022-05-18T12:21:22Z"
    message: ReplicaSet "nginx-deployment-5cc64f49d7" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 7
```

Pod 数量相关：

* `replicas` 表示当前运行中的 Pod 数量（所有 RS `status.replica` 之和）
  
* `updatedReplicas` 为已经升级的 Pod 数量（当前 RS `status.replica`）
  
* `readyReplicas` 为 Ready Pod 数量（所有 RS `status.readyReplicas` 之和）

* `availableReplicas` 为 Available Pod 数量（所有 RS `status.availableReplicas` 之和）
  
  `unavailableReplicas` 为 Unavailable Pod 数量（所有 RS `spec.replica` 之和 - `availableReplicas`

Condition 相关：

* `Progressing` 表示 Deployment 是否正常处理中。

* `Available` 表示是否大部分 Pod 处于 Available 状态，即 `availableReplicas` >= `spec.replica` - `spec.maxUnavailable`
  
* `ReplicaFailure` 表示是否有任意的 ReplicaSet 处于异常情况。

`observedGeneration` 为 Deployment 的 `metadata.generation` 值。当更新 Status 时，如果 `observedGeneration` 小于当前的就放弃更新，以防止使用了旧的 Deployment 定义。

#### 1.4.2 DaemonSet

DaemonSet Status 中一些参数可以观察 Update 的状态：

```yaml
status:
  desiredNumberScheduled: 3
  currentNumberScheduled: 3
  numberAvailable: 3
  numberMisscheduled: 0
  numberReady: 3
  updatedNumberScheduled: 3
  observedGeneration: 3
```

下面表明对应状态的 Pod 数量，状态是按顺序判断的（处于后面的状态必定包含前面的状态）：
  
  * `desiredNumberScheduled` 表明期望运行 Pod 的数量
  
    * `currentNumberScheduled` 表示调度到 Node 上的 Pod 数量，也就是 Pod 总数

      * `updatedNumberScheduled` 表示已升级 Pod 数量

      * `numberReady` 表示 Ready Pod 数量

        * `numberAvailable` 表明 Available Pod 数量

  * `numberMisscheduled` 表明不期望运行，但是运行中的 Pod 数量

`observedGeneration` 为 DaemonSet 的 `metadata.generation` 值。当更新 Status 时，如果 `observedGeneration` 小于当前的就放弃更新，以防止使用了旧的 DaemonSet 定义。

#### 1.4.3 StatefulSet

StatefulSet Status 中一些参数可以观察到 Update 状态：

```yaml
status:
  replicas: 3
  readyReplicas: 3
  availableReplicas: 3
  collisionCount: 0
  currentReplicas: 3
  updatedReplicas: 3
  currentRevision: main-cluster-pd-6584676774
  updateRevision: main-cluster-pd-6584676774
  observedGeneration: 4
```

Pod 数量相关：

* `replicas` 为期望运行的 Pod 数量

* `readyReplicas` 为 Ready Pod 数量

* `availableReplicas` 为 Available Pod 数量

* `currentReplicas` 为当前 Revision 的运行中的 Pod 数量，`updatedReplicas` 为已经升级 Revision 的运行中的 Pod 数量

  在升级流程中，`currentReplicas` 表示旧版本的 Pod，`updatedReplicas` 为升级完成的 Pod。两者会交替变化。

* `currentRevision` 为当前 Revision，`updatedReplicas` 为需要升级的 Revision
  
  `currentRevision` != `updatedReplicas` 就表明正在升级中

## 2 Deployment 实现

### 2.1 架构

Deployment 不是直接管理 Pod，而是通过 ReplicaSet 来间接管理。Deployment 仅仅会操作 ReplicaSet，由 ReplicaSet 来操作 Pod 的缩扩容。

{{< image src="img3.png" height=350 >}}

* Deployment 与 ReplicaSet
  
  ReplicaSet 与 Deployment 通过 `ownerReferences` 来表明所属关系，通过 `revision` annotation 来表示 ReplicaSet 的修订版本。

* ReplicaSet 与 Pod
  
  基于 Pod Template 会生成一个 Hash 值，设置到 ReplicaSet 的 Labels 中，并该 Label 作为筛选 ReplicaSet 管理的 Pod。

Hash 值其实也是 ReplicaSet 命名的一部分，通过 Hash 值使得多个 ReplicaSet 正确管理着属于自己的 Pod。

```yaml
apiVersion: apps/v1
kind: ReplicaSet
  labels:
    pod-template-hash: ff6655784
  name: nginx-deployment-ff6655784
```

每一次 Pod Template 的更新，Deployment 都会创建 New ReplicaSet。Deployment 会保存多个 ReplicaSet，每个 ReplicaSet 会有着一个版本号 `revision`，最大的就是当前 Deployment 对应的。因此，通过 ReplicaSet 实现了版本的机制。

```bash
$ kubectl get rs

NAME                                DESIRED   CURRENT   READY   AGE
nginx-deployment-9456bbbf9          3         3         3       47m
nginx-deployment-ff6655784          0         0         0       45m
```

Deployment 相关代码都位于 [**Deployment Controller**](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go)。

### 2.2 Trigger

当用户修改 Deployment 的 Pod Template 定义后，Controller 判断出当前 Deployment 与任何 ReplicaSet 的 Pod Template 不相同，那么就会创建新的 ReplicaSet。

{{< image src="img4.png" height=300 >}}

新的 ReplicaSet 的 revision 会加 1，使得整体有了版本的概念。

### 2.3 Update

Rolling Update 流程就是一个交替缩扩容的流程：

{{< image src="img5.png" height=250 >}}

1. 扩容 New RS
   
   满足当前所有 ReplicaSet 的 Pod 总数不能超过 `desired replica` + `max surge` 前提下，扩容 New RS。

2. 缩容 Old RS
   
   满足当前所有 ReplicaSet 的 Pod 总数不低于 `desired replica` - `max unavailable` 前提下，缩容 Old RS。

3. 循环执行，直到 New RS 满足 `desired replica`，那么剩余的 Old RS Pod 可以被删除。

### 2.4 Rollback

前面看到，Deployment 都是基于 ReplicaSet 来管理 Pod 的，并且会保留一定数量的 ReplicaSet。

当执行 `kubectl rollback` 时，kubectl 会读取对应 `revision` 的 ReplicaSet 的 Pod Template，对比当前的 Deployment 的 Pod Template，生成 Patch 请求更新 Deployment。

在 [**Trigger**](#121-trigger) 阶段，如果 Deployment 的 Pod Template 与存在的 ReplicaSet 相等，那么就会将 ReplicaSet 更新 revision。那么剩下的流程又是 [**Update**](#122-update) 阶段了。

{{< image src="img6.png" height=300 >}}

## 3 DaemonSet/StatefulSet 实现

DaemonSet 与 StatefulSet 都是直接管理的 Pod，两者在 Pod 管理与 Update 实现上基本相同，因此下面以 DaemonSet 为例来说明。

### 3.1 架构

DaemonSet 是基于 Pod Label 的方式来管理的 Pod。对于每一个 Node，会生成绑定到 Node 上的一个 Pod（通过 `pod.spec.affinity`）。

{{< image src="img7.png" height=300 >}}

因为直接管理 Pod，DaemonSet 需要一种方式记录所有的历史版本。DaemonSet 将所有的历史版本记录到 ControllRevision 资源。

```bash
$ kubectl get controllerrevisions.apps

NAME                              CONTROLLER                              REVISION   AGE
nginx-daemonset-664df6cb85        daemonset.apps/nginx-daemonset          3          62m
nginx-daemonset-7686cf8cb5        daemonset.apps/nginx-daemonset          1          63m
nginx-daemonset-7f9fcbbcc8        daemonset.apps/nginx-daemonset          2          62m
```

ControllRevision 记录了历史版本的 `revision` 以及具体配置：

```yaml
apiVersion: apps/v1
kind: ControllerRevision
metadata:
  labels:
    app: nginx-ds
    controller-revision-hash: 664df6cb85
  name: nginx-daemonset-664df6cb85
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: DaemonSet
    name: nginx-daemonset
    uid: df462976-70a2-43e2-9f46-e0a92a452b93
revision: 3
data:
  spec:
    template:
      # ...
```

* `revision` - 版本号
* `ownerReferences` - 基于 `ownerReferences` 表明所属哪个 DaemonSet/StatefulSet
* `data` - 记录了具体的 Pod Template
* `labels.controller-revision-hash` - 记录 Pod Template 的 Hash 值

### 3.2 Update 实现

当用户修改 DaemonSet 的 Pod Template 定义后，Controller 判断出当前 DaemonSet 与任何的 ControllerReview 记录的 Pod Template 不相同，那么就会创建新的 ControllerRevision。

每个 ControllerRevision 会记录 Label "controller-revision-hash"，为当前 Pod Template 的 Hash 值（也可能有随机加盐）。该 Hash 值也会记录在 Pod Label 中。

```yaml
apiVersion: apps/v1
kind: ControllerRevision
metadata:
  labels:
    controller-revision-hash: 664df6cb85
# ...
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    controller-revision-hash: 664df6cb85
# ...
```

控制循环逻辑中，Controller 会遍历所有的运行中的 Pod，如果 Pod 与 ControllerRevision 的 "controller-revision-hash" 不相等，说明 Pod 就需要升级。

对 Pod 执行升级操作实际上就是 Delete 操作，等待在下一次控制循环中会认为 Pod 不存在，而按照当前 DS 创建一个新的 Pod。

{{< image src="img8.png" height=300 >}}

{{< admonition note Note>}}
如果是 OnDelete 升级策略，Controller 就不会走上面的判断升级与删除逻辑，而是依赖着用户删除。
{{< /admonition >}}

### 3.3 Rollback 实现

当执行 `kubectl rollout`，kubectl 会读取对应 ControllerRevision 的 Pod Template，对比 DaemonSet 生成 Patch 请求，并更新 DaemonSet。

在 [**Update 实现**](#33-rollback-实现) 中第一步看到，Controller 会比较所有 ControllerRevision 的 Pod Template。在 Rollback 场景下，Controller 就会发现一个 Old ControllerRevision 的 Pod Template 匹配。

{{< image src="img9.png" height=300 >}}

那么 Controller 就会更新该 ControllerRevision 的 `spec.revision` 大小，表明该 ControllerRevision 是当前使用中的，然后以该 ControllerRevision 的 Pod Template 和 Hash 值去执行 Update 操作。

## 4 PodDisruptionBudget

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

### 4.1 Spec 与 Status

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

## 参考

* 官方文档：[**Pod Disruptions**](https://kubernetes.io/zh/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets)
* 官方文档：[**Deployments**](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/)


