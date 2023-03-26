# Kubernetes - Volume


## 1 PV PVC StorageClass

### 1.1 PV

[PV]^(Persistent Volume) 代表一个实际可用的后端存储（也可能不是后端，而是 Local PV）。大多数情况下，PV 是一个网络文件系统，或者分布式存储，或者云厂商的云盘。

PV 的定义仅仅描述了存储的属性：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```
* **`capacity`** - 描述存储的大小
  
* `volumeMode` - 存储类型，见 [**Volume Modes**](#113-volume-mode)
  
* `accessModes` - 决定了是否允许多节点复用，并其读写权限，见 [**Access Mode**](#114-access-mode)

* **`storageClassName`** ：指定其所属的 StorageClass。

  也可以不指定 StorageClass，那么只有不指定 class 的 PVC 才可以绑定该 PV。

* `persistentVolumeReclaimPolicy` - PV 回收策略，见 [**Reclaim Policy**](#112-reclaim-policy)

* `mountOptions` ：PV 挂载到 Node 时添加的挂载选项

  Kubernetes 不会对挂载选项有任何的检查。针对不同类型的 PV，这个选项是不同的，具体见 [**Mount Points**](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#mount-options)。

* **`nodeAffinity`** ：设定节点亲和性来限制只有特定的节点可以使用其 PV

  在使用 Local PV 时必定要使用 nodeAffinity。

#### 1.1.1 Phase 与 Lifecycle

Phase 描述了 PV 所处的状态：

```yaml
apiVersion: v1
kind: PersistentVolume
spec:
  # ...
status:
  phase: Available
```

可见的 Phase 有：

* **Available** - PV 已经创建，没有绑定 PVC。

* **Bound** - PV 已经绑定到了 PVC。可以从 `spec.claimRef` 看到绑定了的 PVC。

  ```yaml
  # ...
  spec:
    claimRef:
      apiVersion: v1
      kind: PersistentVolumeClaim
      name: tikv-main-cluster-tikv-2
      namespace: default
      resourceVersion: "1121998"
      uid: 12612562-cc49-4f66-a76d-34bce001e1eb
  ```
  
* **Released** - 在 `persistentVolumeReclaimPolicy: Retain` 模式下，PV 不会随着 PVC 被删除而删除，会处于 Released 状态。

  Released 状态会保留 PVC 的绑定信息 `spec.claimRef`，如果删除 `spec.claimRef` PV 会重新变为 "Available"，因此可以被再次使用。
  
* **Failed** - 在 `persistentVolumeReclaimPolicy: Recycle` 模式下，清除 PV 数据时失败会变为 Failed 状态。

PV 生命周期包括 5 个阶段：

1. **Provisioning** ：即 PV 的创建，可以直接创建 PV（静态方式），也可以使用 StorageClass 动态创建；
2. **Binding** ：将 PV 分配给 PVC；
3. **Using** ：Pod 通过 PVC 使用该 Volume，并可以通过准入控制 `StorageObjectInUseProtection` 阻止删除正在使用的 PVC；
4. **Releasing** ：Pod 释放 Volume 并删除 PVC；
5. **Reclaiming** ：回收 PV，可以保留 PV 以便下次使用，也可以直接从云存储中删除；
6. **Deleting** ：删除 PV 并从云存储中删除后段存储；

{{< admonition note Note>}}
PV 的底层实现见 [**Kubernetes - 存储设计**](../volume/)。
{{< /admonition >}}

#### 1.1.2 Reclaim Policy

Reclaim Policy 指定了当其绑定的 PVC 删除时，该如何处理 PV。目前支持的策略包括：

* **Retain**
  
  当 PVC 删除时，PV 仍然存在（表明实际数据仍然存在），不给过 PV 会变为 Released 状态。因此，目前该 PV 是不能再次被绑定的。

  <important>如果删除 Released 状态的 PV，对应实际的数据盘（例如 AWS EBS、GCE PD 等）是不会被删除的</important>。这意味着，需要用户去手动删除对应的实际存储。

  如果期望重用 Released PV，可以创建 PV 定义 `spec.volumeName` 对应的 PVC。
  
* **Delete**
  
  当 PVC 删除时，PV 会自动被删除，并且<important>对应实际的数据盘也会被删除</important>。

* **Recycle**（Deprecated）
  
  如果底层的数据盘支持，`Recycle` 在 PVC 删除时，会清除 PV 上的数据（例如 `rm -rf /volume/*`），使得该 PV 可以直接被重新复用。

#### 1.1.3 Volume Mode

Volume Mode 表明了存储的类型，目前支持：

* **Filesystem**（默认）- 存储为文件系统，使用时会通过 Mount 挂载到 Pod 某个目录。如果使用的 Device 没有文件系统，那么 Kubernetes 会按照存储类型相关参数创建文件系统。
  
* **Block** - 存储为块设备，Pod 中可以直接使用该块设备。Kubernetes 不会为块设备创建文件系统，因此需要 Pod 自行处理块设备。

#### 1.1.4 Access Mode

Access Mode 描述了 Pod 访问 Volume 的模式，是否支持多节点同时挂载。目前支持：

* ReadWriteOnce - 只允许一个 Node 挂载使用（Pod 可以同时访问），并且支持 Read 与 Write。
  
* ReadOnlyMany - 允许多个 Node 同时挂载，但是仅仅支持 Read。

* ReadWriteMany - 允许多个 Node 同时挂载，并且支持同时 Read 与 Write。

* ReadWriteOncePod - 只允许一个 Pod 使用，支持 Read 与 Write。

当然，不同的数据盘类型对其 accessMode 支持程度不同，具体见 [**AccessMode**](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)。

#### 1.1.5 Finalizer

所有的绑定了 PVC 的 PV 会添加一个名为 `kubernetes.io/pv-protection` 的 Finalizer，以防止用户删除正在使用中的 PV。

同样，如果 PVC 被某个 Pod 使用中，也会添加一个名为 `kubernetes.io/pvc-protection` 的 Finalizer，以防止用户删除使用中的 PVC。

{{< admonition note Note>}}
实际存储的删除在 Finalizer 移除后开始了，不会参考 PV 上其他的 Finalizer。
{{< /admonition >}}

### 1.2 PVC

[PVC]^(Persistent Volume Claim) 描述一个 Pod 对 PV 的需求，是基于 namespace 下的。

{{< admonition tip "为什么需要 PV PVC？" >}}
感觉还是为了解耦，解耦使得 Pod 与存储形成了多对多的关系。

最直观的好处，一个 PV 可以通过多个 PVC 进行绑定，使得数据可以被多个 Pod 共享使用。也可以预定义多个 PV，而通过 PVC 可以直接由调度去选择合适的 PV 进行绑定。
{{< /admonition >}}

PVC 定义中主要是描述了对 PV 的需求，也就是**告诉调度如何选择一个合适的 PV**。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  volumeName: foo-pv
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow  # 指定绑定的 StorageClass
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

* `volumeMode` - 存储类型，见 [**Volume Modes**](#113-volume-mode)
  
* `accessModes` - 决定了是否允许多节点复用，并其读写权限，见 [**Access Mode**](#114-access-mode)
  
* `volumeName` ：指定绑定的 PV name，或者 Bind 后也会填充为对应的 PV name
  
* `resources` ：Pod 所需求的资源
  
* `selector`：进一步来过滤选择的 PV，只有满足匹配条件的 PV 才可以被绑定
  
* `storageClassName` ：指定所属的 StorageClass，见 [**StorageClass**](#13-storageclass)

#### 1.2.1 Phase

PVC `status.phase` 表明 PVC 所处于的阶段：

```yaml
status:
  phase: Bound
```

* **Pending** - PVC 还没有绑定到 PV，这时 `spec.volumeName` 为空。

* **Bound** - PVC 已经绑定了 PV，这是 `spec.volumeName` 为对应 PV。

* **Lost** - 与 PV 的绑定关系丢失。包括几种情况：

  * PVC 之前是 Bound 状态，但是 `spec.volumeName` 为空；
  * `spec.volumeName` 对应的 PV 不存在了；
  * `spec.volumeName` 对应的 PV 存在，但是 PV 并没有绑定该 PVC，也就是说，PV 与 PVC 之间绑定关系不是对应的；

#### 1.2.2 StorageClass

PVC `spec.storageClassName` 用于指定创建 PV 使用的 StorageClass。只有相同 StorageClass 的 PV 与 PVC 才可以完成绑定。

需要注意的是，不设置 `storageClassName` 与 `storageClassName: ""` 是两种情况：

* `storageClassName: ""` - 可以认为 "" 就是一种 StorageClass，只有都是 "" 的 PV 与 PVC 才可以完成绑定

* 不设置 `storageClassName` - 具体行为取决于集群是否有着 [**Default StorageClass**](#135-default-storageclass)。如果存在，那么 `storageClassName` 就会默认配置为 Default StorageClass。

如果不存在 Default StorageClass，那么不设置 `storageClassName` 等同于 `storageClassName: ""`。

#### 1.2.3 Pod 使用 PVC

在 Pod 定义或者 Pod Template 中，使用 pvc 类型 `volume` 来指定使用的 PVC。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:         # 添加 volume 挂载
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd            # volume 类型为 PVC
      persistentVolumeClaim:
        claimName: myclaim  # 指定使用的 PVC
```

#### 1.2.4 DataSource 与 DataSourceRef

PVC 提供了 `dataSource` 字段来实现 [**Volume Snapshot**](#5-volume-snapshot) 与 [**Volume Clone**](#6-clone-volume)。

PVC 还提供了 `dataSourceRef` 字段，该字段大致含义与 `dataSource` 几乎相同，大部分情况下该字段都是相同的内容。

不过，`dataSourceRef` 提供了更高级的功能，包括：

* `dataSourceRef` 可以包含不同任意类型的对象，而 `dataSource` 只支持 PVC 与 VolumeSnapshot。
* `dataSourceRef` 能给支持跨 Namespace 的 PVC 或 VolumeSnapshot，`dataSource` 只能支持同 Namespace 操作。

### 1.3 StorageClass

PV 的创建方法有两种：

* [静态创建]^(Static Provision)，由管理员手动创建所有支持的 PV；

* [动态创建]^(Dynamic Provision)，定义 StorageClass，按 PVC 来创建合适的 PV；

所以明确，**`StorageClass`** 是**根据 PVC，来创建/回收 PV 的**。

而**创建与回收的 Driver** 就称为 **`Provisioner`**，不同类型的 PV 的创建与回收方式都是不同的，所以有着许多的 Provisioner 实现。

StorageClass 中大多数属性是其创建出 **PV 的模板**：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs  # 指定 Provisioner
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```
* `provisioner` - 指定创建 PV 的插件，见 [**Provisioner**](#131-provisioner)

* `reclaimPolicy` - 指定创建的 PV 的回收策略，默认为 Delete

* `allowVolumeExpansion` - 是否允许通过修改 PVC 对象的属性，来对使用的 PV 进行扩容。

  大多数云存储都支持该选项，具体见 [**allow-volume-expansion**](https://kubernetes.io/docs/concepts/storage/storage-classes/#allow-volume-expansion)。

* `mountOptions` - 指定其创建的 PV 的 Mount Options
  
* `volumeBindingMode` - PV 与 PVC 的绑定操作的时机，见 [**Volume Binding Mode**](#132-volume-binding-mode)
  
* `allowedTopologies` - 指定允许创建 PV 的区域，见 [**Allowed Topologies**](#133-allowed-topologies)

* `parameters` - 存储参数，见 [**Parameters**](#134-parameters)

#### 1.3.1 Provisioner

StorageClass 代表了是创建 PV 的模板，为了支持多样的实际的数据盘类型，StorageClass 通过指定 Provisioner 来表明由哪个存储插件来创建 PV。

**Provisioner 只是一个标识，本质上由各个存储的管理程序（本质上就是 Controller）通过 `spec.provisioner` 来筛选 PV，进而去创建 PV**。

Kubernetes 原生支持的 Provisioner 以 `kubernetes.io/xxx` 的规范命名。当然，Provisioner 也可以以 Pod 方式运行，只要能够识别 `spec.provisioner` 即可。

所有 Kubernetes 内置支持的 Provisioner 见 [**provisioner**](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner)。

#### 1.3.2 Volume Binding Mode

Volume Binding Mode 决定了 PV 与 PVC 的绑定操作应该发生在什么时候。

* **Immediate**（默认）
  
  Immediate 表示创建了 PVC 后，就去完成 PV 与 PVC 的绑定。**这也意味着，PV 与 PVC 的绑定是不会考虑 Pod 的调度要求的**。

* **WaitForFirstConsumer**
  
  WaitForFirstConsumer 表明：等待使用该 PVC 的 Pod 被创建被调度后，才会去完成 PV 与 PVC 的绑定的。**因此，PVC 的绑定会考虑到 Pod 的调度要求**。

  在一些场景，WaitForFirstConsumer 就是必须的了，例如：

  * 在云厂商场景下，PV 往往是 AZ 级别的，所以要等到 Pod 调度后，去对应的 AZ 创建 PV。
  * 在 Local PV 场景下，PV 是每个 Node 单独的，因此需要等到 Pod 调度后，才绑定 Local PV。

{{< admonition note Note>}}
使用 WaitForFirstConsumer 时，不能使用 Pod `spec.nodeName` 来选择 Node。因为这种情况 Pod 不会经过 Scheduler 来调度，因此无法完成 PVC 的绑定。
{{< /admonition >}}

#### 1.3.3 Allowed Topologies

对于大部分云厂商，云盘是 AZ 级别的服务，无法跨 AZ 使用。Allowed Topologies 可以提供给 Provisioner，指定允许哪些区域创建。

例如，下面示例表明在 AZ `us-central-1a` 与 `us-central-1b` 下创建 PV。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: failure-domain.beta.kubernetes.io/zone
    values:
    - us-central-1a
    - us-central-1b
```

当然，不同的 Provisioner 支持的 Label Key 不同，使用时需要参考对应的文档。

{{< admonition note Note>}}
当使用了 `volumeBindingMode: WaitForFirstConsumer` 时，PV 会等待 Pod 调度后才会创建。此时 Pod 所属 Node 已经确定（AZ 已经确定），因此 PV 所属 AZ 也是确定的，所以不需要使用 Allowed Topologies 限定 PV 区域。
{{< /admonition >}}

#### 1.3.4 Parameters

Parameters 为特定存储类型的参数，是专门提供给 Provisioner 使用的。例如 AWS EBS 支持的参数：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "10"
  fsType: ext4
```

Kubernetes 内置的 Provisioner 支持的参数见 [**parameters**](https://kubernetes.io/docs/concepts/storage/storage-classes/#parameters)。

一个 StorageClass 最多可以定义 512 个参数。这些参数对象的总长度不能 超过 256 KiB, 包括参数的键和值。

#### 1.3.5 Default StorageClass

Default StorageClass 是没有设置 `storageClassName` 的 PVC 使用的 StorageClass。通过添加一个 Anno `storageclass.kubernetes.io/is-default-class` 来设置 Default StorageClass：

```yaml
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: gp2
```

### 1.4 绑定关系

PV PVC StorageClass 三者的关系如下图：

{{< image src="img1.png" height=250 >}}

PVC 绑定 PV 基于 `spec.volumeName` 字段：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  volumeName: pvc-f278fa2e-63d6-4086-af29-a4b7b91ec031
```

PV 绑定 PVC 基于 `spec.claimRef` 字段：

```yaml
apiVersion: v1
kind: PersistentVolume
spec:
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: grafana-storage
    namespace: monitoring
    resourceVersion: "59917623"
    uid: f278fa2e-63d6-4086-af29-a4b7b91ec031
```

本质上，PV 与 PVC 的绑定都是基于 Controller 来维护的。大致流程如下：

1. PVC Controller 通过 `pvc.spec.volumeName` 字段，判断出 PVC 需要选择一个 PV 绑定。
2. PVC Controller 查找所有 Available PV，选择出满足条件的 PV 进行绑定。
3. 如果没有满足的 PV，那么会基于 `pvc.spec.storageClassName` 指定的 StorageClass，创建一个 PV。

#### 1.4.1 预留 PV 与 PVC

基于上面的绑定关系字段，我们可以一开始再创建 PVC 或者 PV 时，就设置好对应的字段，那么 Controller 就不会去绑定对应的 PV 或 PVC。

因此，这样可以实现固定关系一多一的 PV 与 PVC。

## 2 Local PV

### 2.1 手动创建使用 LocalPV

在自己测试环境下，往往没有一个云存储可以作为 PV 使用，而使用 Local PV，使得 Pod 使用的 PV 来自于本地的目录或者块设备。

先明确 Local PV 如何定义，即 **PV 类型为 `hostPath`**，表明实际存储设备来自于宿主机的一个目录。

同时，只有调度到该节点的 Pod 才能使用这个 PV，所以要**设定 PV 的 `spec.nodeAffinity`**，仅仅允许该节点使用该 PV：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage  # 指定 storageClassName
  local:  # local 类型表明使用 Local PV
    path: /mnt/disks/vol1
  nodeAffinity:  # 设定节点亲和性，使得只能本地节点使用 PV
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1  # 仅仅允许 node-1 使用该 PV
```

而为了让 PVC 与 PV 绑定推迟到 Pod 创建后才决定绑定的 PV，我们需要一个 **StorageClass**，即使其并不能够支持动态创建 PV：

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner  # no-provisioner 不支持动态创建 PV
volumeBindingMode: WaitForFirstConsumer  # 让 PVC 绑定推迟
```

而在使用时候，我们只需要一个普通的 PVC，指定好 `spec.storageClassName` 后，就可以使用 LocalPV 了。

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: example-local-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage  # 指定 StorageClass，来绑定 PV
```

### 2.2 local pv static provisioner

[**local pv static provisioner**](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner) 将上面的步骤进行了简化，其启动了一个 DaemonSet，**根据配置好的目录，自动生成对应目录的 Local PV**。

所以带来的好处就是，我们不需要为一个个节点去创建对应的 PV 了，有 Pod 自动负责该事。

#### 2.2.1 原理

local pv static provisioner 为创建一个 DamonSet，在每个节点创建 Daemon Pod。

每个节点上的 Daemon Pod 会根据其 ConfigMap 配置好的 **`discovery dir`**，在目下查找可以作为 PV 的子目录。

在 discovery dir 下挑选目录有两个条件：

* **目录必须是一个挂载点**，也就是 mountinfo 中可以看到的；

  {{< admonition tip "为什么必须是挂载点？">}}
  我猜测这是为了通过挂载点得到目录容量大小，从而填充 PV 容量相关属性。
  {{< /admonition >}}

* **满足其 ConfigMap 中指定的目录匹配方式 namePattern**。

#### 2.2.2 示例

而为了使用 LocalPV，我们也需要定义一个 StorageClass，让创建出的 LocalPV 能够属于正确的 StorageClass。

所以使用的步骤为：

1. 每个节点上**准备作为 LocalPV 的目录**，该目录必须是一个挂载点，且位于 discovery dir 下；
1. **创建 StorageClass**，作为后续创建出的 Local PV 的 StorageClass；
1. **定义并创建 ConfigMap**，为了将相关的信息传递给 provisioner 的 DaemonSet Pod；
1. **定义并创建 ServiceAccount**，因为后续的 Daemon Pod 需要创建 Local PV 结构，所以需要相关的权限；
1. **定义并创建 DaemonSet**，根据 config map 信息，其 Pod 里程序会自动创建 Local PV，其定义就与上述的 PV 一致；

看一下其 StorageClass、ConfigMap 和 DaemonSet 定义：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-disks  # name 要与下面 ConfigMap 相同
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete  # 支持 Delete Retain
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-provisioner-config 
  namespace: default 
data:
  storageClassMap: |     
    fast-disks:  # 上面创建的 StorageClass 名字
       hostDir: /mnt/fast-disks  # 宿主机 discovery dir
       mountDir:  /mnt/fast-disks   # Daemon Pod 中查找的 discover dir
       blockCleanerCommand:
         - "/scripts/shred.sh"
         - "2"
       volumeMode: Filesystem
       fsType: ext4
       namePattern: "*"  # 查找子目录的匹配方式
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: local-volume-provisioner
  namespace: default
  labels:
    app: local-volume-provisioner
spec:
  selector:
    matchLabels:
      app: local-volume-provisioner 
  template:
    metadata:
      labels:
        app: local-volume-provisioner
    spec:
      serviceAccountName: local-storage-admin
      containers:
        - image: "k8s.gcr.io/sig-storage/local-volume-provisioner:v2.4.0"
          imagePullPolicy: "Always"
          name: provisioner 
          securityContext:
            privileged: true
          env:
          - name: MY_NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          volumeMounts:
            - mountPath: /etc/provisioner/config 
              name: provisioner-config
              readOnly: true             
            - mountPath:  /mnt/fast-disks 
              name: fast-disks
              mountPropagation: "HostToContainer" 
      volumes:
        - name: provisioner-config  # 能够访问到 ConfigMap
          configMap:
            name: local-provisioner-config      
        - name: fast-disks  # 能够查看到 discovery dir
          hostPath:
            path: /mnt/fast-disks 
```

等到自动创建出 Local PV 后，后续的流程其实就是使用一个 Local PV 的流程了。

### 2.3 local path provisioner（推荐）

上面的 local pv static provisioner 需要自行挂载一个磁盘或者目录，在测试环境中这显得比较繁琐。

在测试环境中推荐使用 [**local path provisioner**](https://github.com/rancher/local-path-provisioner)，其会在指定的目录自动为 PV 创建目录，PV 清理时自动删除目录，不需要为每个 PV 进行一次挂载。

## 3 Expand Volume

### 3.1 前置条件

不同的存储类型对于扩容的支持不同，目前支持的存储有：

* 大部分云厂商磁盘
* CSI

因为 CSI 是一个存储框架，需要 CSI Driver 支持 `Expand Volume` 相关的接口。因此，需要参考具体 CSI 插件的文档以确定是否支持扩容。

并且，只有当 StorageClass 中设置 `allowVolumeExpansion: true` 时，表明才允许扩容 PVC。

对于 Filesystem 类型的 PVC，还有一些额外的条件：

* 文件系统必须是 XFS、Ext3、Ext4；
* Access Mode 必须是 **ReadWrite**；

### 3.2 扩容行为

目前，扩容可以分为两种情况：

* **Pod 启动时扩容**
  
  对于块设备，扩容行为是动态的，但是对于文件系统的扩容仅仅在 Pod 启动期间完成。也就是说，当改变 PVC 的大小后，需要重启 Pod 才会在 Pod 中生效。

* **Pod 运行中扩容**
  
  对于文件存储，是天然支持动态扩容的，因此 Pod 运行中可以感知到扩容，不需要重启 Pod。

  目前，Kubernetes 还支持 `ExpandInUsePersistentVolumes` 特性（目前还是 beta 状态），支持部分块设备在 Pod 运行中扩容。包括：GCE-PD, AWS-EBS, Cinder, and Ceph RBD。

  该特性默认是不打开的，需要设置 kubelet 的启动参数 `--feature-gates ExpandInUsePersistentVolumes=true`，不过部分云厂商的集群默认是打开的，需要参考对应的文档。

从 PVC 的 `status.condition` 中可以看到扩容相关的状态：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
  conditions:
  - lastProbeTime: null
    lastTransitionTime: 2019-04-14T03:51:36Z
    message: Waiting for user to (re-)start a pod to finish file system resize of
      volume on node.
    status: "True"
    type: FileSystemResizePending
```

condition 可能的类型包括：

* **Resizing** - 扩容操作正常进行中；
* **FileSystemResizePending** - 底层块设备扩容已经完成，文件系统扩容需要等待 Pod 重启；

### 3.3 原理

从控制面来看，扩容操作的触发就是基于 PVC 与 PV 的大小是否相等。如果不相等，那么 Controller 就会开始执行扩容操作。

{{< admonition note Note>}}
如果手动修改 PV 的大小与 PVC 相等，那么实际上就会跳过扩容。
{{< /admonition >}}

扩容操作本质上可以分为两个阶段：

1. **扩容底层块设备**
   
   因为本质上还是扩容底层块设备，因此会受到不同云盘的限制，先要参考对应的文档。

   例如，AWS EBS 扩容间隔为 6h。

2. **扩容文件系统**

   部分文件系统是支持挂载时扩容的，例如 `xfs_growfs` 扩容 XFS 或者 `resize2fs` 扩容 Ext4 文件系统。

### 3.4 错误处理

当 PVC 扩容失败时，Kubernetes 会不断的重试扩容操作，直到成功。这时候，需要手动采取一些操作才能恢复。

* 重建 PVC
  
  通过删除 PVC 与创建新的 PVC 来放弃扩容。大致流程如下：

  1. 修改 PV 为 `Retain` 策略，防止数据丢失。
  2. 删除 PVC 对象，并且将 PV 的 `spec.claimRef` 设置为控制，设置 PV 可以被重新绑定。
  3. 重新创建 PVC，改变 PVC 大小。通过 PVC 的 `spec.volumeName` 来复用 PV。

* 降低 PVC 扩容的大小（v1.23）[alpha]
  
  在开启 `ExpandPersistentVolumes` 与 `RecoverVolumeExpansionFailure` 特性情况下，可以多次修改 PVC 的大小。并且，通过 PVC 的 `status.resizeStatus` 字段可以观察扩容的状态。

  但是，新的大小仍然高于 `status.capacity`，也就是说，不允许进行缩容。

## 4 Ephemeral Volume

一些应用可能需要临时的存储作为缓存，不关心重启后是否可用。一般情况下会使用 `EmptyDir` 来作为临时的存储使用，但是 `EmptyDir` 使用的是 Node 上的存储，会占用 Node 的存储空间，并且性能完全由 Node 上的存储决定。

为此，Kubernetes 提供了 Ephemeral Volume 的功能。Ephemeral Volume 的生命周期完全遵从于 Pod 的生命周期，随着 Pod 一起创建与删除。

目前支持的 Ephemeral Volume 有：

* `EmptyDir` - EmptyDir 可以看做为一种 Ephemeral Volume，使用的是 Node 存储或者内存
* `ConfigMap` `DownwardAPI` `Sercret` - 不同类型的 Kubernetes 数据注入到 Pod 中
* CSI Ephemeral Volume - 由 CSI Driver 支持的 Ephemeral Volume
* Generic Ephemeral Volume - 由 Kubernetes 基于 PV 使用的 Ephemeral Volume

### 4.1 Generic Ephemeral Volume

Generic Ephemeral Volume 类似于 `EmptyDir`，但是是由 PV 实现的。因此，Generic Ephemeral Volume 能够实现一些额外的功能：

* Volume 可以是 Local PV 或者 Cloud PV
* Volume 有着限制的大小，不会像 `EmptyDir` 能够超限使用
* Volume 支持 MountOption 配置
* 支持 Volume 相关操作，例如 Snapshot、Expand 等

一个典型的例子如下：

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: my-app
spec:
  containers:
    - name: my-frontend
      image: busybox:1.28
      volumeMounts:
      - mountPath: "/scratch"
        name: scratch-volume
      command: [ "sleep", "1000000" ]
  volumes:
    - name: scratch-volume
      ephemeral:             # 指定 ephemeral volume
        volumeClaimTemplate:
          metadata:
            labels:
              type: my-frontend-volume
          spec:
            accessModes: [ "ReadWriteOnce" ]
            storageClassName: "scratch-storage-class"
            resources:
              requests:
                storage: 1Gi
```

#### 4.1.1 Lifecycle 与 PVC

当 Pod 创建后，Kubernetes 会创建 Ephemeral Volume 对应的 PVC，从而走对应的 PV 创建与绑定操作。

当 Pod 被删除时，Kubernetes 会删除对应的 PVC，而 PV 的保留策略仍然是由 [**Reclaim Policy**](#112-reclaim-policy) 决定。

因为有着对应的 PVC，所以 PVC 相关的 Snapshot、Expand 等功能都是可以使用的。

#### 4.1.2 PVC 的命名

Kubernetes 为 Ephemeral Volume 创建的 PVC 命名为：`${pod-name}-${volume-name}`。因为命名规则是固定的，所以要小心不同 Pod 创建的 PVC 可能冲突。

## 5 Volume Snapshot

Volume Snapshot 利用云厂商的功能，通过 CR 来对存储进行快照备份。目前，Volume Snapshot 仅部分 CSI Driver 支持。

Volume Snapshot 涉及到以下 CRD：

* **VolumeSnapshotContent** - 实际创建出的 Volume Snapshot（概念类似与 PV）
  
* **VolumeSnapshot** - 执行 Volume Snapshot 的请求（概念类似于 PVC）

* **VolumeSnapshotClass** - 指定 VolumeSnapshot 的不同属性（概念类似于 StorageClass）

Kubernetes 提供了 `csi-snapshotter` 的 Sidecar Container 和 CSI Driver 共同部署。`csi-snapshotter` 负责 Watch 和管理集群中的 VolumeSnapshot 和 VolumeSnapshotContent 资源，涉及实际操作时调用 CSI Driver 的 `CreateSnapshot` 和 `DeleteSnapshot` 接口。

### 5.1 VolumeSnapshot

VolumeSnapshot 中定义了对某个 PVC 执行 Volume Snapshot 的请求。

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: new-snapshot-test
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: pvc-test
```

`volumeSnapshotClassName` 定义了使用的 VolumeSnapshotClass，那么执行快照时会使用对应的属性。如果没指定，会使用默认 Class。

Snapshot 操作执行后，将对应的 VolumeSnapshotContent 设置在定义中。当然，你也可以直接绑定一个已经存在的 VolumeSnapshotContent。

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: test-snapshot
spec:
  source:
    volumeSnapshotContentName: test-content
```

### 5.2 VolumeSnapshotContent

VolumeSnapshotContent 代表一个具体的快照资源。

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: snapcontent-72d9a349-aacd-42d2-a240-d775650d2455
spec:
  deletionPolicy: Delete
  driver: hostpath.csi.k8s.io
  source:
    volumeHandle: ee0cfb94-f8d4-11e9-b2d8-0242ac110002
  sourceVolumeMode: Filesystem
  volumeSnapshotClassName: csi-hostpath-snapclass
  volumeSnapshotRef:
    name: new-snapshot-test
    namespace: default
    uid: 72d9a349-aacd-42d2-a240-d775650d2455
```

`volumeHandle` 指定了执行 Snapshot 的 Volume ID，而不是对应的 Snapshot ID。

创建快照后，对应的 Snapshot ID 记录在 `snapshotHandle` 上。当然，也可以通过该字段将已有的 Snapshot 绑定到 VolumeSnapshotContent。

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: new-snapshot-content-test
spec:
  deletionPolicy: Delete
  driver: hostpath.csi.k8s.io
  source:
    snapshotHandle: 7bdd0de3-aaeb-11e8-9aae-0242ac110002  # snapshot id
  sourceVolumeMode: Filesystem
  volumeSnapshotRef:
    name: new-snapshot-test
    namespace: default
```

#### 5.2.1 Source Volume Mode

`sourceVolumeMode` 字段表明执行 Snapshot 的 Volume 的模式，包括：Filesystem 或 Block。

如果你希望使用 Snapshot 创建 PVC 时，支持转换 Volume Mode，那么需要添加一个 Annotation：

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: new-snapshot-content-test
  annotations:
    - snapshot.storage.kubernetes.io/allow-volume-mode-change: "true" # 表明改变 Mode
spec:
  deletionPolicy: Delete
  driver: hostpath.csi.k8s.io
  source:
    snapshotHandle: 7bdd0de3-aaeb-11e8-9aae-0242ac110002
  sourceVolumeMode: Filesystem
  volumeSnapshotRef:
    name: new-snapshot-test
    namespace: default
```

### 5.3 VolumeSnapshotClass

VolumeSnapshotClass 提供了执行 Snapshot 的参数。

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass
driver: hostpath.csi.k8s.io
deletionPolicy: Delete
parameters:
  # ...
```

`driver` 指定了使用哪个 CSI Driver，因为由 Driver 具体执行 Snapshot 操作。

`deletionPolicy` 表明 VolumeSnapshot 删除时，如何处理 VolumeSnapshotContent。

* **Retain** - 保留 VolumeSnapshotContent，意味着底层的 Snapshot 也会保留
* **Delete** - 删除 VolumeSnapshotContent，底层的 Snapshot 也会删除

### 5.4 基于 Snapshot 创建 PV

定义 PVC 时，可以指定让其使用 VolumeSnapshot 创建 PV。那么 PV 对应的存储将从实际 Snapshot 恢复，从而恢复数据。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-pvc
spec:
  storageClassName: csi-hostpath-sc
  dataSource:
    name: new-snapshot-test
    kind: VolumeSnapshot  # 基于快照恢复
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## 6 Clone Volume

Clone Volume 的意思是复制现有的 Volume。利用大部分云厂商的 Clone Disk 的功能。

从 Kubernetes 角度看，Clone 只能是在创建新的 PVC 时，指定一个现有的 PVC 作为数据源，并且源 PVC 必须是 Bound 状态并且不在使用中。新的 PVC 会绑定到新的 PV，对应后面会绑定到 Clone 出的新的 Disk。

实际上的 Volume Clone 操作完全由 CSI Driver 负责。新的

通过 `dataSource` 来表明从一个先有的 PVC 克隆。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: clone-of-pvc-1
    namespace: myns
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: cloning
  resources:
    requests:
      storage: 5Gi  # 大于等于现在的 storage
  dataSource:       # 指向一个 PVC
    kind: PersistentVolumeClaim
    name: pvc-1
```

Clone 后的 PVC 是完全独立的，源 PVC 的修改或删除都不会影响新的 PVC。

## 参考

* 官方文档：[**StorageClass**](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/)
* 官方文档：[**Persistent Volume**](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/)
