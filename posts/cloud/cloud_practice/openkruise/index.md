# OpenKruise


## 1 概述

OpenKruise 通过自定义一些 CRD 来提供一些原生 Resource 不含有的功能。

其基本架构如下：

{{< image src="img1.png" height=300 >}}

* Manager

  kruise-manager 是各个 CRD 的 Controller 和 Webhook，通过 `Deployment` 部署。

  每个 CRD Controller 逻辑上都是独立的，不过都运行在 kruise-manager 一个进程中。

* Daemon
  
  kruise-daemon 通过 `DaemonSet` 部署在每个 Node 上，用于实现镜像预热、容器重启等功能。

## 2 Pod InPlace Update

### 2.1 什么是 InPlace Update

InPlace Update 是 OpenKruise 提供的核心功能，不同于原生 Workload 的 Recreate Update，InPlace Update 指的是不删除 Pod 情况下升级 Container。

{{< image src="img2.png" height=350 >}}

官方文档给出的两种方式的对比：

* Recreate Update
  
  通过 Delete Pod 和 Create Pod 来进行升级，因此对于 `Deployment` 等部署的， Pod Name 和 UID 都会变化。

  新创建的 Pod 会重新走调度流程，因此所在的 Node 可能会发生变化。

  新创建的 Pod 很大可能是分配一个新的 Pod IP。

* InPlace Update
  
  仅仅重建 Pod 内部的 Container，升级前后仍然是相同的 Pod，因此 Pod Name 和 UID 不会变化，Pod 使用的资源（IP、Volume 等）也不会变化。

  一个 Container 升级时，Pod 中其他的 Container 不会受到影响。

目前，OpenKruise 实现的几个 Workload 都是支持 InPlace Update：

* `CloneSet`
* `Advanced StatefulSet`
* `Advanced DaemonSet`
* `SidecarSet`

其中，`CloneSet`、`Advanced StatefulSet`、`Advanced DaemonSet` 使用相同的代码实现的 InPlace Update，`SidecarSet` 逻辑有一些不一样。

### 2.2 使用场景

InPlace Update 对应 Workload 中的升级类型为 `InPlaceIfPossible`。顾名思义，只有在一定场景下才能进行 InPlace Update，其他场景还是必须使用 Recreate Update。

InPlace Update 支持的场景有：

* 更新 `spec.template.metadata.*` 字段，例如 Label 和 Annotation。Kruise 会将改动应用当当前所有的 Pod 上。

* 更新 `spec.template.spec.container[x].image` 字段，Kruise 会原地升级该特定的 Container，而不会重建 Pod。

* 更新 `spec.template.metadata.labels/annotations`，并且 Container Env 引用了这些 Label 和 Annotation，那么 Kruise 会原地升级该 Container，使得信息 Env 值生效。

对于其他字段的更改，InPlace Update 是无法支持的，因此还是会执行 Recreate Update。

### 2.3 实现

官网给出的 InPlace Update 一个 Pod 的流程图：

{{< image src="img3.png" height=350 >}}

Kubernetes 原生 Workload 都是基于 Delete + Create 来进行 Pod 升级的，不过 Pod 本身是支持直接更新 Container Image 来仅仅升级某个 Container 的。

Kruise 就是利用了这个能力，加上 `kruise-daemon` 的配合，实现了 InPlace Update 的功能。

## 3 Workloads

### 3.1 CloneSet

CloneSet 对标原生的 `Deployment`，提供了增强的功能。

#### 3.1.1 PVC Template

PVC Template 功能可以为每个 Pod 创建对应的 PVC，类似于 `StatefulSet`。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
# ...
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:    # mount the volume to a container
        - name: data-vol
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:  # volume template
  - metadata:
      name: data-vol
    spec:
      resources:
        requests:
          storage: 20Gi
```

不过，与 `StatefulSet` 的 PVC Template 功能的区别在于：

* 创建的 PVC 会有 ownerReference 指向 CloneSet。也就是说，其 PVC 会随着 CloneSet 删除而删除。

* 创建的 PVC 会自动打上 `apps.kruise.io/cloneset-instance-id: xxx`，使得能够的和 CloneSet 关联。

* Pod 被缩容删除时，其对应的 PVC 也会被删除。但是如果 Pod 是被手动删除或者重调度，还是会复用的 PVC。

* 如果 Pod 是 Recreate Update，PVC 会被删除新建。如果 Pod 是 InPlace Update，PVC 会复用。

#### 3.1.1 指定 Pod 缩容

`StatefulSet` 按照顺序缩容 Pod，`Deployment` 随机缩容 Pod，CloneSet 能够支持指定需要删除的 Pod。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
spec:
  # ...
  replicas: 4
  scaleStrategy:
    podsToDelete:
    - sample-9m4hp  # pod name
```

当 Controller 执行缩容操作时，会优先处理 `podsToDelete` 中的 Pod，并且删除后自动清除该记录。

#### 3.1.2 缩容顺序

CloneSet 会按照以下优先级缩容 Pod：

1. 未调度 < 已调度

2. PodPending < PodUnknown < PodRunning

3. Not ready < ready

4. 较小 `pod-deletion-cost` < 较大 `pod-deletion-cost`
   
   `pod-deletion-cost` 是一个特殊的 Pod 的 Annotation，CloneSet 缩容时会参考该值。它表示该 Pod 的删除代价，代价越小的 Pod 会被优先删除。

5. 较大打散权重 < 较小
   
   CloneSet 支持同 Node 打散 Pod，或者 Pod Topology Spread Constraint 打散 Pod。如果 Pod Template 中包含 Pod Topology Spread Constraint 定义，那么会优先按照拓扑约束删除 Pod。否则，会按照同节点 Pod 优先删除的策略。

6. 处于 Ready 时间较短 < 较长

7. 容器重启次数较多 < 较少

8. 创建时间较短 < 较长

#### 3.1.3 InPlace Update

CloneSet 提供三种升级方式：

* `Recreate` - 默认的方式，重建 `Pod` 来升级
* `InPlaceIfPossible` - 允许情况下进行 InPlace Update，否则使用 Recreate Update
* `InPlaceOnly` - 进允许 InPlace Update，不允许的情况会禁止更新字段

此外，还提供了 Grace Period 功能。配置了情况下，Controller 变更 Pod Status 为 Not-Ready 后，会等待 Grace Period 时间，然后才会去更新 Pod Spec 触发升级。这样，为 endpoints-controller 留出时间将 `Pod` 从 `Endpoints` 列表中移除。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
spec:
  # ...
  updateStrategy:
    type: InPlaceIfPossible
    inPlaceUpdateStrategy:
      gracePeriodSeconds: 10
```

#### 3.1.4 Partition 分批灰度

Partition 表示保留旧版本 Pod 的数量或者百分比。这表明，CloneSet 会保留部分 Pod 不进行升级，用以实现分批次灰度。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
metadata:
  # ...
  generation: 2
spec:
  replicas: 5
  updateStrategy:
    partition: 3
  # ...
status:
  observedGeneration: 2
  readyReplicas: 5
  replicas: 5
  currentRevision: sample-d4d4fb5bd 
  updateRevision: sample-56dfb978d4
  updatedReadyReplicas: 2           
  updatedReplicas: 2  # 只有两个 Pod 升级了
```

如果启动了 `CloneSetPartitionRollback` Feature Gate，还支持将 Partition 扩大时，回滚升级。

#### 3.1.5 Lifecycle Hook

每个 CloneSet 管理的 Pod 会处于明确的状态，在 Pod Label 中 `lifecycle.apps.kruise.io/state` 标记：

{{< image src="img4.png" height=300 >}}

* Normal 正常状态
* PreparingUpdate 准备原地升级
* Updating 原地升级中
* Updated 原地升级完成
* PreparingDelete 准备删除

Lifecycle Hook 指的就是上述状态转换过程中的卡点。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
spec:
  lifecycle:
    preDelete:
      finalizersHandler:  # 通过 finalizer 定义 hook
      - example.io/unready-blocker
    inPlaceUpdate:
      finalizersHandler:
      - example.io/unready-blocker
  lifecycle:
    inPlaceUpdate:
      labelsHandler:     # 或者也可以通过 label 定义
        example.io/block-unready: "true" 
```

#### 3.1.6 流式扩容

指定 `scaleStrategy.maxUnavailable` 可以限制扩容的步长。支持绝对值或者百分比。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
spec:
  # ...
  minReadySeconds: 60
  scaleStrategy:
    maxUnavailable: 1
```

### 3.2 Advanced StatefulSet

`Advanced StatefulSet` 对标原生的 `StatefulSet`，提供了更多功能。其类型名也是 `StatefulSet`，只是 `apiVersion` 不一样。

`Advanced StatefulSet` 是完全兼容原生的 `StatefulSet` 行为的。因此迁移时，只需要将 `apiVersion` 修改后提交即可：

```yaml
-  apiVersion: apps/v1
+  apiVersion: apps.kruise.io/v1beta1
   kind: StatefulSet
   metadata:
     name: sample
   spec:
     #...
```

#### 3.2.1 InPlace Update

配置与前述相同 [**InPlace Update**](#313-inplace-update)。

#### 3.2.2 自定义升级顺序

`Advanced StatefulSet` 支持不按照 Pod 序号顺序进行升级。通过 `spec.updateStrategy.rollingUpdate.unorderedUpdate` 配置来自定义升级顺序。

注意，`unorderedUpdate` 只能配合 `podManagementPolicy: Parallel` 使用。

目前，只支持 `priorityStrategy` 一个优先级策略，支持两种优先级方式：

* weight
  
  通过给不同 Label 的 Pod 分配权重，权重大的 Pod 优先升级。

  ```yaml
  apiVersion: apps.kruise.io/v1beta1
  kind: StatefulSet
  spec:
    # ...
    updateStrategy:
      rollingUpdate:
        unorderedUpdate:
          priorityStrategy:
            weightPriority:
            - weight: 50
              matchSelector:
                matchLabels:
                  test-key: foo
            - weight: 30
              matchSelector:
                matchLabels:
                  test-key: bar
  ```

* order
  
  通过给 Pod 配置特定的 Label 来表示 Pod 序号，而不使用 Pod 名称作为序号。

  注意，使用的 Label Value 类似于 Pod 名称，默认为 Int 值，例如 "sts-10" 表明需要为 10。

  ```yaml
  apiVersion: apps.kruise.io/v1beta1
  kind: StatefulSet
  spec:
    # ...
    updateStrategy:
      rollingUpdate:
        unorderedUpdate:
          priorityStrategy:
            orderPriority:
              - orderedKey: some-label-key
  ```

#### 3.2.3 序号跳过

通过在 `reserveOrdinals` 字段中写入跳过的序号，`Advanced StatefulSet` 会自动跳过创建这些序号的 Pod。如果对应 Pod 已经存在，那么会删除 Pod。

```yaml
apiVersion: apps.kruise.io/v1beta1
kind: StatefulSet
spec:
  # ...
  replicas: 4
  reserveOrdinals:
  - 1
```

举个例子，对于上面示例，当前实际运行的 Pod 序号为 `[0,2,3,4]`。

* 如果要移除 Pod 3，设置 `reserveOrdinals: [1,3]`。那么 Pod 3 会被删除，并且 Pod 5 会被创建出来。
* 如果想删除 Pod 3，设置 `reserveOrdinals: [1,3]` 与 `replicas: 3`。那么 Pod 3 会被删除，不会有新的 Pod 创建。

#### 3.2.4 流式扩容

指定 `scaleStrategy.maxUnavailable` 可以限制扩容当前不可用 Pod 的最大数量。注意，该功能只在 `podManagementPolicy: Parallel` 时生效。

例如，下面示例中，一开始会一次性创建 10 个 Pod。后续，每当一个 Pod 变为 Running 和 Ready 后，才会创建新的 Pod。

```yaml
apiVersion: apps.kruise.io/v1beta1
kind: StatefulSet
spec:
  # ...
  replicas: 100
  scaleStrategy:
    maxUnavailable: 10% # percentage or absolute number
```

### 3.3 BroadcastJob

`BroadcastJob` 会在每个 Node 上运行一个短期的 Pod，用于为每个 Node 做一个一次性的工作。

此外，`BroadcastJob` 还支持维持每个 Node 都跑成功一个 Pod 任务，即使后面新 Node 加入后，也会自动分配 Pod 执行。

基本的定义如下：

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: BroadcastJob
metadata:
  name: broadcastjob-ttl
spec:
  template:          # template 描述 Pod 模板
    spec:
      containers:
        - name: pi
          image: perl
          command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  parallelism: 1    # 允许同时运行多少个 Pod
  completionPolicy:
    type: Always
    ttlSecondsAfterFinished: 30
```

`CompletionPolicy` 指定了管理 Pod 的行为，支持：

* Always
  
  意味着 Job 最终会完成，并配置以下参数：

  * `ActiveDeadlineSeconds` 指定 Pod 运行超时时间，超时后 Job 会停止并标记为 Failed。
  * `BackoffLimit` 指定重试次数，超过次数后 Job 才会标记为 Failed。
  * `TTLSecondsAfterFinished` 指定 `BroadcastJob` 在完成后的保留时间，超过时间就会删除对应的 Job 和 Pod。

* Never
  
  意味着 BroadcastJob 会一直存在。也就是说，当新的 Node 加入后，就会自动创建对应的 Job 执行。

## 4 Sidecar

### 4.1 SidecarSet

`SidecarSet` 通过 Admission Webhook 的方式自动为集群中的 Pod 注入 Sidecar 容器，并且支持原地升级 Sidecar 容器。

基本的定义如下：

```yaml
# sidecarset.yaml
apiVersion: apps.kruise.io/v1alpha1
kind: SidecarSet
metadata:
  name: test-sidecarset
spec:
  namespace: ns          # 管理的 Namespace，默认为全集群生效
  selector:              # 注入的 Pod 的 label selector
    matchLabels:
      app: nginx
  updateStrategy:        # 升级策略
    type: RollingUpdate
    maxUnavailable: 1
  containers:            # sidecar 容器
  - name: sidecar1
    image: centos:6.7
    command: ["sleep", "999d"] # do nothing at all
    volumeMounts:
    - name: log-volume
      mountPath: /var/log
  volumes: # this field will be merged into pod.spec.volumes
  - name: log-volume
    emptyDir: {}
```

在 SidecarSet 中 `status` 记录了容器信息：

```yaml
status:
  matchedPods: 1
  observedGeneration: 1
  readyPods: 1
  updatedPods: 1
```

#### 4.1.1 Container 注入

`SidecarSet` 扩展了 `container` 字段，用于方便注入：

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: SidecarSet
metadata:
  name: sidecarset
spec:
  selector:
    matchLabels:
      app: sample
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - mountPath: /nginx/conf
      name: nginx.conf
    # extended sidecar container fields
    podInjectPolicy: BeforeAppContainer # container 注入 spec 位置
    shareVolumePolicy:
      type: disabled | enabled
    transferEnv:
    - sourceContainerName: main
      envName: PROXY_IP
  volumes:
  - Name: nginx.conf
    hostPath: /data/nginx/conf
  injectionStrategy:
    paused: false
```

* Container 注入位置
  
  `podInjectPolicy` 支持配置为 **BeforeAppContainer** 或者 **AfterAppContainer**，定义 Sidecar Container 注入到 Pod 原 Container 的位置。

* 共享 Volume
  
  * 共享特定 Volume：通过 `spec.volumes` 挂载特定 Volume；
  * 共享所有 Volume：配置 `shareVolumePolicy: true` 可以将 Pod 所有的 Volume 挂载到 Sidecar Container；

* 共享 Env
  
  通过 `transferEnv` 配置将别的 Container Env 设置给 Sidecar Container。

* 注入暂停
  
  `injectionStrategy.paused: true` 表明暂停 Container 注入，但是不会影响存量的 Container。

#### 4.1.2 Sidecar Container 版本控制

因为 Sidecar Container 仅仅在 Pod 创建时注入，所以如果 `SidecarSet` 变更后，可能多个 Pod 会注入不同版本的 Sidecar Container。

为此，`SidecarSet` 会创建 `ControllerRevision` 记录历史的版本，并支持使用历史版本进行注入。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: SidecarSet
metadata:
  name: sidecarset
spec:
  # ...
  injectionStrategy:
    revision: 
      revisionName: <specific-controllerRevision-name>
```

#### 4.1.3 Sidecar Container 热升级

如果 Sidecar Container 是影响服务可用性的，那么 Sidecar Container 升级就会导致 Pod 不可用。为此，提供了热升级的解决方案。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: SidecarSet
metadata:
  name: hotupgrade-sidecarset
spec:
  # ...
  containers:
  - name: sidecar
    image: openkruise/hotupgrade-sample:sidecarv1
    imagePullPolicy: Always
    lifecycle:
      postStart:
        exec:
          command:
          - /bin/sh
          - /migrate.sh
    upgradeStrategy:
      upgradeType: HotUpgrade
      hotUpgradeEmptyImage: openkruise/hotupgrade-sample:empty
```

* `upgradeType: HotUpgrade` 表明执行热升级方案。
* `hotUpgradeEmptyImage` 指定热升级过程中的切换容器。
* `lifecycle.postStart` 执行脚本处理状态迁移。

当 Pod 创建时，将会注入两个 Sidecar Container：

* `<sidecarContainer.name>-1` 为当前实际运行的 Sidecar Container
* `<sidecarContainer.name>-2` 为配置的 `hotUpgradeEmptyImage` 容器，作为热升级的预备

{{< image src="img5.png" height=300 >}}

执行热升级的流程如下：

1. 将 `hotUpgradeEmptyImage` 容器升级为 Sidecar Container
2. 新容器启动后执行 PostStart Hook，执行业务的状态迁移
3. 将原先运行的 Sidecar Container 设置为 `hotUpgradeEmptyImage` 容器

#### 4.1.4 Metadata 注入

`SidecarSet` 支持注入 Pod Annotations：

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: SidecarSet
spec:
  containers:
  # ...
  patchPodMetadata:
  - annotations:
      oom-score: '{"log-agent": 1}'
      custom.example.com/sidecar-configuration: '{"command": "/home/admin/bin/start.sh", "log-level": "3"}'
    patchPolicy: MergePatchJson
  - annotations:
      apps.kruise.io/container-launch-priority: Ordered
    patchPolicy: Overwrite | Retain
```

支持的策略包括：

* Retain - 原 Pod 中不存在的 Annotation 才会注入
* Overwrite - 会覆盖原 Pod 中的 Annotation
* MergePatchJson - 用于 Annotation 是 JSON 字符串的场景，支持与原 Pod Anno 的 JSON 字符串合并后注入

### 4.2 ResourceDistribution

`ResourceDistribution` 支持将 `Secret` 或者 `ConfigMap` 同步到多个 Namespace 中。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: ResourceDistribution
metadata:
  name: sample
spec:
  resource:               # 原始 Secret 或 ConfigMap
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: game-demo
    data:
      # data ...
  targets:                  # 筛选出需要同步的 Namespace
    allNamespaces: false    # 同步给所有 Namespace
    includedNamespaces:     # 同步给特定的 Namespace
      list:
      - name: ns-1
      - name: ns-4
    namespaceLabelSelector: # 基于 label selector 筛选
      matchLabels:
        group: test
    excludedNamespaces:     # 排除特定 Namespace
        list:
      - name: ns-3
```

当 `resource` 中定义的配置更新后，会自动地对所有目标 Namespace 中的资源进行同步更新。`ResourceDistribution` 通过记录资源的 Hash 来判断是否需要同步。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
  annotations:  
    kruise.io/resourcedistribution.resource.from: sample
    kruise.io/resourcedistribution.resource.distributed.timestamp: 2021-09-06 08:44:52.7861421 +0000 UTC m=+12896.810364601
    kruise.io/resourcedistribution.resource.hashcode: 0821a13321b2c76b5bd63341a0d97fb46bfdbb2f914e2ad6b613d10632fa4b63
```

由于配置了 OwnerReference，当 `ResourceDistribution` 被删除时，所有相关的资源都会被删除。

## 5 多区域部署

### 5.1 UnitedDeployment

UnitedDeployment 提供了多区域创建 Workload 的能力。每个区域被称为 `Subset`，对应会部署一个 Workload。

目前支持部署的 Workload 包括：`StatefulSet` `AdvancedStatefulSet` `CloneSet` `Deployment`。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: UnitedDeployment
metadata:
  name: sample-ud
spec:
  replicas: 6            # 所有 subset 部署的 replica 总和
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: sample
  template:               # 创建的 Workload 的模板
    statefulSetTemplate:  
      metadata:
        labels:
          app: sample
      spec:
        # ...
  topology:              # 部署拓扑
    subsets:
    - name: subset-a    # zone a 部署一个 sts，replica 为 1
      nodeSelectorTerm:
        matchExpressions:
        - key: node
          operator: In
          values:
          - zone-a
      replicas: 1
    - name: subset-b   # zone b 部署一个 sts，replica 为 3
      nodeSelectorTerm:
        matchExpressions:
        - key: node
          operator: In
          values:
          - zone-b
      replicas: 50%
    - name: subset-c  # zone b 部署一个 sts，replica 为 2
      nodeSelectorTerm:
        matchExpressions:
        - key: node
          operator: In
          values:
          - zone-c
  updateStrategy:
    manualUpdate:
      partitions:
        subset-a: 0
        subset-b: 0
        subset-c: 0
    type: Manual
# ...
```

可以看到，在 `topology.subsets` 中我们定义了多个 `Subset`。当一个 `Subset` 增加或去除时，UnitedDeployment 控制器会创建或删除相应的 Workload。

对于创建的 Workload：

* 每个 Workload 有着独立的命名，前缀为 `<UnitedDeployment_name>-<Subset_name>`。

* Workload 基于 `spec.template` 来创建的，同时会将 `Subset` 定义中的相关配置合并（例如 `nodeSelectorTerm` 等）

#### 5.1.1 升级方式

如果修改了 `spec.template` 那么就会更新各个 Workload，从而触发 Pod 的升级。

对于 `AdvancedStatefulSet` 与 `StatefulSet` 作为 Workload，还能够通过 Subset 配置 Partition 策略，从而控制升级流程。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: UnitedDeployment
metadata:
  name: sample-ud
spec:
  updateStrategy:
    manualUpdate:    # 设置各个 workload 的 partition
      partitions:
        subset-a: 0
        subset-b: 0
        subset-c: 0
    type: Manual
```

### 5.2 WorkloadSpread

`WorkloadSpread` 也用于多区域部署，与 `UnitedDeployment` 区别在于，`WorkloadSpread` 是基于一个 Workload 进行多区域的部署。

`WorkloadSpread` 利用 Webhook 为 Workload Pod 注入 `Subset` 的部署信息，同时控制 Pod 的缩扩容顺序。

```yaml
apiVersion: apps.kruise.io/v1alpha1
kind: WorkloadSpread
metadata:
  name: workloadspread-demo
spec:
  targetRef:
    apiVersion: apps/v1 | apps.kruise.io/v1alpha1
    kind: Deployment | CloneSet
    name: workload-xxx
  subsets:                      # subsets 中定义多区域的部署
    - name: subset-a
      requiredNodeSelectorTerm:
        matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
              - zone-a
    preferredNodeSelectorTerms:
      - weight: 1
        preference:
        matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
              - another-node-label-value
      maxReplicas: 3
      tolertions: []
      patch:
        metadata:
          labels:
            xxx-specific-label: xxx
    - name: subset-b
      requiredNodeSelectorTerm:
        matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
              - zone-b
  scheduleStrategy:
    type: Adaptive | Fixed
    adaptive:
      rescheduleCriticalSeconds: 30
```

## 参考

* 官方文档：[**OpenKruise**](https://openkruise.io/zh/docs/)。
