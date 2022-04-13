# Kubernetes PV PVC StorageClass


## 1 PV

[PV]^(Persistent Volume) 代表一个实际可用的后端存储（也可能不是后端，而是 Local PV）。大多数情况下，PV 是一个网络文件系统，或者分布式存储，或者云厂商的云盘。

### 1.1 Spec

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
* **`capacity`**：描述存储的大小；
* `volumeMode`：存储的使用类型，包括：
    * Filesystem 文件系统：使用时会通过 Mount 挂载到 Pod 某个目录。也支持自动为 device 创建文件系统
    * Block 块设备：通过 device 方式提供给 Pod
* `accessModes` ：设备在实际使用时会先 mount 到宿主机上，accessMode 决定了是否允许多节点复用，并其读写权限，包括：
    * ReadWriteOnce：只允许一个节点挂载使用，并提供读写；
    * ReadOnlyMany：允许多个节点挂载使用，提供读权限；
    * ReadWriteMany：允许多个节点挂载使用，提供读写权限；

	不同的 PV 类型对其 accessMode 支持程度不同，具体见 [**AccessMode**](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)。
* **`storageClassName`** ：指定其所属的 StorageClass。

  也可以不指定 StorageClass，那么只有不指定 class 的 PVC 才可以绑定该 PV。
* `persistentVolumeReclaimPolicy` ：ReclaimPolicy 指定了当其绑定的 PVC 删除时，该如何处理 PV。<br>
  目前的回收策略包括：
    * **Retain** ：保留 PV，PV 会变为 Released 状态；
    * **Recycle** ：自动删除 PV 数据；
    * **Delete** ：同时删除后端磁盘（AWS EBS、GCE PD 等云盘也会被删除）；
	
  目前，NFS 和 HostPath 类型的 PV 仅仅支持 Recycle，云厂商磁盘可以支持 Delete。
* `mountOptions` ：当 PV 挂载到节点上时，添加的附加选项。

  针对不同类型的 PV，这个选项是不同的，具体见 [**Mount Points**](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#mount-options)。
* **`nodeAffinity`** ：PV 可以设定节点亲和性，来限制只有特定的节点可以使用其 PV。

{{< admonition note Note>}}
nodeAffinity 在使用 Local PV 时必定要使用。
{{< /admonition >}}

### 1.2 PV 生命周期

PV 生命周期包括 5 个阶段：
1. **Provisioning** ：即 PV 的创建，可以直接创建 PV（静态方式），也可以使用 StorageClass 动态创建；
1. **Binding** ：将 PV 分配给 PVC；
1. **Using** ：Pod 通过 PVC 使用该 Volume，并可以通过准入控制 `StorageObjectInUseProtection` 阻止删除正在使用的 PVC；
1. **Releasing** ：Pod 释放 Volume 并删除 PVC；
1. **Reclaiming** ：回收 PV，可以保留 PV 以便下次使用，也可以直接从云存储中删除；
1. **Deleting** ：删除 PV 并从云存储中删除后段存储；

对应 5 个阶段，一个 PV 可能处于以下状态：
* **Available**：PV 已经创建，并且没有 PVC 绑定；
* **Bound**：PVC 绑定了该 PV；
* **Released**：PVC 已经被删除，而该 PV 还没有被回收；
* **Failed**：PV 自动回收失败；

{{< admonition note Note>}}
PV 的底层实现见 [**Kubernetes Volume 实现**](../volume-implementation/)。
{{< /admonition >}}

## 2 PVC

[PVC]^(Persistent Volume Claim) 描述一个 Pod 对 PV 的需求，是基于 namespace 下的。

{{< admonition tip "为什么需要 PV PVC？" >}}
感觉还是为了解耦，解耦使得 Pod 与存储形成了多对多的关系。

最直观的好处，一个 PV 可以通过多个 PVC 进行绑定，使得数据可以被多个 Pod 共享使用。也可以预定义多个 PV，而通过 PVC 可以直接由调度去选择合适的 PV 进行绑定。
{{< /admonition >}}

### 2.1 Spec

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
* `accessModes` ：访问模式，与 PV 的访问模式要匹配
* `volumeMode` ：使用模式，与 PV 的访问模式要匹配
* `volumeName` ：指定绑定的 PV name，或者 Bind 后也会填充为对应的 PV name
* `resources` ：Pod 所需求的资源
* `selector`：进一步来过滤选择的 PV，只有满足匹配条件的 PV 才可以被绑定
* `storageClassName` ：指定所属的 StorageClass，只有相同 StorageClass 的 PV PVC 才可以绑定

  如果没有指定 StorageClass，如果系统设置了 Default StorageClass 则使用，否则只能绑定没有设置 StorageClass 的 PV。
  
  {{< admonition note "不指定 StorageClass">}}
  在没有指定 `storageClassName` 下，行为是有多种情况的，见 [**Class**](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#class-1)。
  {{< /admonition >}}

### 2.2 Status

```yaml
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 446Gi
  phase: Bound
```
* `accessModes` ：访问模式
* `capacity` ：可使用的容量
* `phase` ：PVC 的阶段，包括：
  * **Pending** ：没有绑定 PV；
  * **Bound** ：已经绑定到 PV；
  * **Lost** ：已经绑定到 PV，但是 PV 丢失了；

### 2.2 PVC 的使用

在 Pod 定义或者 Pod template 中，使用 pvc 类型 `volume` 来指定使用的 PVC。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:  # 添加 volume 挂载
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd  # volume 类型为 PVC
      persistentVolumeClaim:
        claimName: myclaim  # 指定使用的 PVC
```

## 3 StorageClass

PV 的创建方法有两种：
1. [静态创建]^(Static Provision)，由管理员手动创建所有支持的 PV；
1. [动态创建]^(Dynamic Provision)，定义 StorageClass，按 PVC 来创建合适的 PV；

所以明确，**`StorageClass`** 是**根据 PVC，来创建/回收 PV 的**。

而**创建与回收的 Driver** 就称为 **`Provisioner`**，不同类型的 PV 的创建与回收方式都是不同的，所以有着许多的 Provisioner 实现。

### 3.1 Spec

StorageClass 中大多数属性是其创建出 **PV 的模板**。

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
* `provisioner` ：指定 Provisioner 的实现，这也就决定了其创建 PV 的类型。

  其内置支持的 Provisioner 见 [**provisioner**](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner)。

  当然也可以使用自定义的 Provisioner。
* `reclaimPolicy` ：针对 PV 的回收策略，这也就决定了其创建 PV 的回收策略。默认为 Delete。
* `allowVolumeExpansion` ：是否允许通过修改 PVC 对象的属性，来对使用的 PV 进行扩容。

  大多数云存储都支持该选项，具体见 [**allow-volume-expansion**](https://kubernetes.io/docs/concepts/storage/storage-classes/#allow-volume-expansion)。
* `mountOptions` ：指定其创建的 PV 的 Mount Options。
* `volumeBindingMode` ：决定了何时进行将 PV mount 到节点上。
    * **Immediate** 表明一旦创建 PVC 后，就需要绑定 PV。
    * **WaitForFirstConsumer** 将 PV 与 PVC 的绑定延后，等到 Pod 创建并使用 PVC 时，才会去绑定对应的 PV。<br>
	  {{< admonition tip "Local PV 必须使用 WaitForFirstConsumer">}}
因为 Local PV 只有等到 Pod 创建并调度时，才能决定使用本地的 PV，所以必须使用 **WaitForFirstConsumer**。
      {{< /admonition >}}
* `allowedTopologies` ：在使用云服务的存储时，其 PV 是有 “访问位置”（拓扑域） 的限制的，而就需要满足节点上的 Pod 仅仅使用特定位置的 PV。

  所以在 WaitForFirstConsumer 模式下，通过 `allowedTopologies` 使得仅仅在特定的位置来创建 PV，从而节点的 Pod 可以正常的访问 PV。
* `parameters` ：提供给 Provisioner 创建 PV 的参数，最多支持 512 个参数，长度不能超过 256KiB。

  不同 Provisioner 支持不同参数，见 [**parameters**](https://kubernetes.io/docs/concepts/storage/storage-classes/#parameters)。

## 4 PV PVC StorageClass 之间的关系
下面一张图描述了 PV PVC StorageClass 之间的关系：
{{< find_img "img1.png" >}}

## 5 Local PV

### 5.1 手动创建使用 LocalPV

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

### 5.2 local pv static provisioner

[**local pv static provisioner**](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner) 将上面的步骤进行了简化，其启动了一个 DaemonSet，**根据配置好的目录，自动生成对应目录的 Local PV**。

所以带来的好处就是，我们不需要为一个个节点去创建对应的 PV 了，有 Pod 自动负责该事。

#### 5.2.1 原理

local pv static provisioner 为创建一个 DamonSet，在每个节点创建 Daemon Pod。

每个节点上的 Daemon Pod 会根据其 ConfigMap 配置好的 **`discovery dir`**，在目下查找可以作为 PV 的子目录。

在 discovery dir 下挑选目录有两个条件：

* **目录必须是一个挂载点**，也就是 mountinfo 中可以看到的；

  {{< admonition tip "为什么必须是挂载点？">}}
  我猜测这是为了通过挂载点得到目录容量大小，从而填充 PV 容量相关属性。
  {{< /admonition >}}

* **满足其 ConfigMap 中指定的目录匹配方式 namePattern**。

#### 5.2.2 示例

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

